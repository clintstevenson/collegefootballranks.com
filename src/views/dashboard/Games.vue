<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const loading = ref(false)
const error = ref(null)

const gamesRaw = ref([])
const teamsRaw = ref([])

const latestGameDate = ref(null)

// Filters
const selectedDate = ref('') // yyyy-mm-dd or ''
const selectedConference = ref('ALL')
const teamSearch = ref('')
const selectedTeamId = ref(0) // 0 = All teams

// Sorting / paging
const sortKey = ref('game_date')
const sortDirection = ref('desc')
const currentPage = ref(1)
const pageSize = ref(25)

// ---- API LOAD ----
async function loadData () {
  loading.value = true
  error.value = null

  try {
    const [gamesRes, teamsRes] = await Promise.all([
      fetch('https://api.collegefootballranks.com/v1/games', {
        headers: { Accept: 'application/json' },
      }),
      fetch('https://api.collegefootballranks.com/v1/teams', {
        headers: { Accept: 'application/json' },
      }),
    ])

    if (!gamesRes.ok) {
      throw new Error(`Games API error: ${gamesRes.status} ${gamesRes.statusText}`)
    }
    if (!teamsRes.ok) {
      throw new Error(`Teams API error: ${teamsRes.status} ${teamsRes.statusText}`)
    }

    const gamesJson = await gamesRes.json()
    const teamsJson = await teamsRes.json()

    gamesRaw.value = Array.isArray(gamesJson) ? gamesJson : []
    teamsRaw.value = Array.isArray(teamsJson) ? teamsJson : []

    // compute latest game date
    let maxDate = null
    for (const g of gamesRaw.value) {
      const raw = g.game_date || g.date || g.gameDate
      if (!raw) continue
      const d = new Date(raw)
      if (!Number.isNaN(d.getTime())) {
        if (!maxDate || d > maxDate) maxDate = d
      }
    }
    latestGameDate.value = maxDate ? maxDate.toISOString() : null
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Unable to load games/teams.'
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadData()
})

// ---- DERIVED LOOKUPS ----
const teamById = computed(() => {
  const m = new Map()
  teamsRaw.value.forEach((t) => {
    if (t.team_id != null) m.set(t.team_id, t)
  })
  return m
})

const totalGames = computed(() => gamesRaw.value.length)

// Conferences from teams
const conferences = computed(() => {
  const set = new Set()
  teamsRaw.value.forEach((t) => {
    if (t.conference) set.add(t.conference)
  })
  return ['ALL', ...Array.from(set).sort()]
})

const totalConferences = computed(() => conferences.value.length - 1)

// Teams for typeahead dropdown
const teamsForDropdown = computed(() => {
  const q = teamSearch.value.trim().toLowerCase()
  const arr = [...teamsRaw.value].sort((a, b) =>
    String(a.team_name ?? '').localeCompare(String(b.team_name ?? '')),
  )
  if (!q) return arr
  return arr.filter((t) =>
    String(t.team_name ?? '').toLowerCase().includes(q),
  )
})

const selectedTeamLabel = computed(() => {
  if (!selectedTeamId.value) return 'All Teams'
  const t = teamById.value.get(selectedTeamId.value)
  return t?.team_name || `Team ${selectedTeamId.value}`
})

// ---- FILTERING & SORTING ----
function normalizeDateString (raw) {
  if (!raw) return ''
  const d = new Date(raw)
  if (Number.isNaN(d.getTime())) return ''
  return d.toISOString().slice(0, 10) // yyyy-mm-dd
}

const filteredGames = computed(() => {
  const dateFilter = selectedDate.value
  const conf = selectedConference.value
  const teamId = selectedTeamId.value

  return gamesRaw.value.filter((g) => {
    const gameDateRaw = g.game_date || g.date || g.gameDate

    if (dateFilter) {
      const gameDateStr = normalizeDateString(gameDateRaw)
      if (gameDateStr !== dateFilter) return false
    }

    if (conf !== 'ALL') {
      const homeTeam = teamById.value.get(g.home_team_id)
      const awayTeam = teamById.value.get(g.away_team_id)
      const homeConf = homeTeam?.conference
      const awayConf = awayTeam?.conference
      if (homeConf !== conf && awayConf !== conf) return false
    }

    if (teamId && teamId !== 0) {
      if (g.home_team_id !== teamId && g.away_team_id !== teamId) return false
    }

    return true
  })
})

const numericSortKeys = new Set([]) // no purely numeric sort keys now

const sortedGames = computed(() => {
  const key = sortKey.value
  const dir = sortDirection.value

  const arr = [...filteredGames.value]

  arr.sort((a, b) => {
    let aVal = a[key]
    let bVal = b[key]

    if (key === 'game_date') {
      const aRaw = a.game_date || a.date || a.gameDate
      const bRaw = b.game_date || b.date || b.gameDate
      const aTime = new Date(aRaw || 0).getTime()
      const bTime = new Date(bRaw || 0).getTime()
      return dir === 'asc' ? aTime - bTime : bTime - aTime
    }

    if (numericSortKeys.has(key)) {
      const aNum = Number(aVal ?? 0)
      const bNum = Number(bVal ?? 0)
      return dir === 'asc' ? aNum - bNum : bNum - aNum
    } else {
      const aStr = String(aVal ?? '').toLowerCase()
      const bStr = String(bVal ?? '').toLowerCase()
      if (aStr === bStr) return 0
      if (dir === 'asc') return aStr < bStr ? -1 : 1
      return aStr > bStr ? -1 : 1
    }
  })

  return arr
})

const totalPages = computed(() =>
  Math.max(1, Math.ceil((sortedGames.value.length || 1) / pageSize.value)),
)

const paginatedGames = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return sortedGames.value.slice(start, end)
})

watch([selectedDate, selectedConference, selectedTeamId, pageSize], () => {
  currentPage.value = 1
})

// ---- UI HELPERS ----
function changeSort (key) {
  if (sortKey.value === key) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortDirection.value = key === 'game_date' ? 'desc' : 'asc'
  }
}

function sortIcon (key) {
  if (sortKey.value !== key) return '↕'
  return sortDirection.value === 'asc' ? '↑' : '↓'
}

function goToPage (page) {
  if (page < 1 || page > totalPages.value) return
  currentPage.value = page
}

function formatDate (d) {
  if (!d) return '—'
  const date = new Date(d)
  if (Number.isNaN(date.getTime())) return d
  return date.toLocaleDateString(undefined, {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  })
}

function formatGameDate (game) {
  const raw = game.game_date || game.date || game.gameDate
  return formatDate(raw)
}

function getHomeTeamName (game) {
  const t = teamById.value.get(game.home_team_id)
  return t?.team_name || `Team ${game.home_team_id ?? '—'}`
}

function getAwayTeamName (game) {
  const t = teamById.value.get(game.away_team_id)
  return t?.team_name || `Team ${game.away_team_id ?? '—'}`
}

function getHomeConference (game) {
  const t = teamById.value.get(game.home_team_id)
  return t?.conference || '—'
}

function getAwayConference (game) {
  const t = teamById.value.get(game.away_team_id)
  return t?.conference || '—'
}

function formatScore (game) {
  const hs = game.home_score ?? game.home_points
  const as = game.away_score ?? game.away_points
  if (hs == null || as == null) return '—'
  return `${hs} - ${as}`
}

// winner helpers
function getHomeScore (game) {
  return game.home_score ?? game.home_points
}
function getAwayScore (game) {
  return game.away_score ?? game.away_points
}
function isHomeWinner (game) {
  const hs = getHomeScore(game)
  const as = getAwayScore(game)
  if (hs == null || as == null) return false
  return hs > as
}
function isAwayWinner (game) {
  const hs = getHomeScore(game)
  const as = getAwayScore(game)
  if (hs == null || as == null) return false
  return as > hs
}

// typeahead dropdown selection
function selectTeam (id) {
  selectedTeamId.value = id
}
</script>

<template>
  <div class="py-4">
    <!-- HEADER / HERO -->
    <CRow class="align-items-center mb-4">
      <CCol :md="8">
        <div class="d-flex flex-column gap-1">
          <h2 class="fw-bold mb-0">
            College Football Games
          </h2>
          <p class="text-body-secondary mb-0">
            All games from
            <span class="fw-semibold">api.collegefootballranks.com</span>.
            Filter by date, conference, or team.
          </p>
          <small class="text-body-secondary">
            Latest game date:
            <span class="fw-semibold">{{ formatDate(latestGameDate) }}</span>
          </small>
        </div>
      </CCol>
      <CCol
        :md="4"
        class="text-md-end mt-3 mt-md-0 d-flex justify-content-md-end justify-content-start gap-2"
      >
        <CButton
          color="secondary"
          variant="outline"
          size="sm"
          @click="loadData"
        >
          <CIcon icon="cil-reload" class="me-1" />
          Refresh
        </CButton>
      </CCol>
    </CRow>

    <!-- STATS CARDS -->
    <CRow class="mb-4 g-3">
      <CCol :sm="6" :lg="4">
        <CCard class="h-100 border-0 shadow-sm">
          <CCardBody class="d-flex flex-column justify-content-between">
            <div class="text-body-secondary small mb-1">
              Total Games
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ totalGames || '—' }}
              </div>
              <CIcon icon="cil-list" size="xl" class="opacity-50" />
            </div>
          </CCardBody>
        </CCard>
      </CCol>

      <CCol :sm="6" :lg="4">
        <CCard class="h-100 border-0 shadow-sm">
          <CCardBody class="d-flex flex-column justify-content-between">
            <div class="text-body-secondary small mb-1">
              Conferences
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ totalConferences || '—' }}
              </div>
              <CIcon icon="cil-grid" size="xl" class="opacity-50" />
            </div>
          </CCardBody>
        </CCard>
      </CCol>

      <CCol :sm="6" :lg="4">
        <CCard class="h-100 border-0 shadow-sm">
          <CCardBody class="d-flex flex-column justify-content-between">
            <div class="text-body-secondary small mb-1">
              Filters Active
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-6">
                <span class="badge text-bg-light me-1">
                  {{ selectedDate ? 'Date' : 'Any Date' }}
                </span>
                <span class="badge text-bg-light me-1">
                  {{ selectedConference === 'ALL' ? 'All Conferences' : selectedConference }}
                </span>
                <span class="badge text-bg-light">
                  {{ selectedTeamLabel }}
                </span>
              </div>
            </div>
          </CCardBody>
        </CCard>
      </CCol>
    </CRow>

    <!-- MAIN TABLE CARD -->
    <CCard class="border-0 shadow-sm">
      <CCardHeader class="bg-body">
        <CRow class="align-items-center g-3">
          <CCol :md="3">
            <div class="fw-semibold">
              Games Table
            </div>
            <small class="text-body-secondary">
              Use filters to narrow by date, conference, or team.
            </small>
          </CCol>

          <!-- Date filter -->
          <CCol :md="3">
            <CFormLabel class="small text-body-secondary mb-1">
              Filter by Date
            </CFormLabel>
            <CFormInput
              v-model="selectedDate"
              type="date"
            />
          </CCol>

          <!-- Conference filter -->
          <CCol :md="3">
            <CFormLabel class="small text-body-secondary mb-1">
              Filter by Conference
            </CFormLabel>
            <CFormSelect v-model="selectedConference">
              <option
                v-for="conf in conferences"
                :key="conf"
                :value="conf"
              >
                {{ conf === 'ALL' ? 'All Conferences' : conf }}
              </option>
            </CFormSelect>
          </CCol>

          <!-- Team typeahead dropdown -->
          <CCol :md="3">
            <CFormLabel class="small text-body-secondary mb-1">
              Filter by Team
            </CFormLabel>
            <CDropdown variant="btn-group" class="w-100">
              <CDropdownToggle color="light" class="w-100 d-flex justify-content-between align-items-center">
                <span class="text-truncate">{{ selectedTeamLabel }}</span>
                <CIcon icon="cil-caret-bottom" class="ms-2" />
              </CDropdownToggle>
              <CDropdownMenu class="p-2" style="max-height: 320px; overflow-y: auto; min-width: 100%;">
                <CFormInput
                  v-model="teamSearch"
                  type="text"
                  size="sm"
                  placeholder="Type to filter teams..."
                  class="mb-2"
                />
                <CDropdownItem
                  :active="selectedTeamId === 0"
                  @click="selectTeam(0)"
                >
                  All Teams
                </CDropdownItem>
                <CDropdownItem
                  v-for="t in teamsForDropdown"
                  :key="t.team_id"
                  :active="selectedTeamId === t.team_id"
                  @click="selectTeam(t.team_id)"
                >
                  {{ t.team_name }}
                </CDropdownItem>
              </CDropdownMenu>
            </CDropdown>
          </CCol>
        </CRow>
      </CCardHeader>

      <CCardBody class="p-0">
        <!-- LOADING / ERROR STATES -->
        <div
          v-if="loading"
          class="py-5 text-center text-body-secondary"
        >
          <CSpinner class="me-2" />
          Loading games…
        </div>

        <div
          v-else-if="error"
          class="py-4 px-3 text-center text-danger"
        >
          <div class="fw-semibold mb-1">
            Failed to load games
          </div>
          <div class="small mb-3">
            {{ error }}
          </div>
          <CButton
            color="primary"
            size="sm"
            @click="loadData"
          >
            Retry
          </CButton>
        </div>

        <div v-else>
          <CTable
            hover
            responsive
            striped
            align="middle"
            class="mb-0"
          >
            <CTableHead class="text-nowrap bg-body-secondary bg-opacity-50">
              <CTableRow>
                <CTableHeaderCell
                  class="clickable"
                  @click="changeSort('game_date')"
                >
                  Date
                  <span class="ms-1 small">
                    {{ sortIcon('game_date') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="clickable"
                  @click="changeSort('home_team_id')"
                >
                  Home
                  <span class="ms-1 small">
                    {{ sortIcon('home_team_id') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="clickable"
                  @click="changeSort('away_team_id')"
                >
                  Away
                  <span class="ms-1 small">
                    {{ sortIcon('away_team_id') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell class="text-center">
                  Score
                </CTableHeaderCell>
                <CTableHeaderCell>
                  Home Conf
                </CTableHeaderCell>
                <CTableHeaderCell>
                  Away Conf
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>

            <CTableBody>
              <CTableRow
                v-for="(game, idx) in paginatedGames"
                :key="game.game_id ?? idx"
              >
                <CTableDataCell>
                  {{ formatGameDate(game) }}
                </CTableDataCell>
                <CTableDataCell>
                  <span
                    class="badge"
                    :class="isHomeWinner(game) ? 'text-bg-success' : 'text-bg-light border'"
                  >
                    {{ getHomeTeamName(game) }}
                  </span>
                </CTableDataCell>
                <CTableDataCell>
                  <span
                    class="badge"
                    :class="isAwayWinner(game) ? 'text-bg-success' : 'text-bg-light border'"
                  >
                    {{ getAwayTeamName(game) }}
                  </span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatScore(game) }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ getHomeConference(game) }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ getAwayConference(game) }}
                </CTableDataCell>
              </CTableRow>

              <CTableRow v-if="!paginatedGames.length">
                <CTableDataCell colspan="6" class="text-center py-4">
                  <div class="fw-semibold mb-1">
                    No games match your filters.
                  </div>
                  <div class="small text-body-secondary">
                    Try clearing some filters.
                  </div>
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>
        </div>
      </CCardBody>

      <!-- PAGINATION -->
      <CCardFooter
        v-if="!loading && !error && sortedGames.length"
        class="d-flex flex-wrap align-items-center justify-content-between gap-2"
      >
        <div class="small text-body-secondary">
          Showing
          <span class="fw-semibold">
            {{ (currentPage - 1) * pageSize + 1 }}
          </span>
          –
          <span class="fw-semibold">
            {{
              Math.min(currentPage * pageSize, sortedGames.length)
            }}
          </span>
          of
          <span class="fw-semibold">
            {{ sortedGames.length }}
          </span>
          games
        </div>
        <div class="d-flex align-items-center gap-1">
          <CButton
            color="secondary"
            variant="outline"
            size="sm"
            :disabled="currentPage === 1"
            @click="goToPage(currentPage - 1)"
          >
            ‹ Prev
          </CButton>
          <span class="small mx-2">
            Page
            <span class="fw-semibold">{{ currentPage }}</span>
            /
            <span class="fw-semibold">{{ totalPages }}</span>
          </span>
          <CButton
            color="secondary"
            variant="outline"
            size="sm"
            :disabled="currentPage === totalPages"
            @click="goToPage(currentPage + 1)"
          >
            Next ›
          </CButton>
        </div>
      </CCardFooter>
    </CCard>
  </div>
</template>

<style scoped>
.clickable {
  cursor: pointer;
}
.clickable:hover {
  text-decoration: underline;
}
</style>
