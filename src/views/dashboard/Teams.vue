<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const loading = ref(false)
const error = ref(null)

const teamsRaw = ref([])

const searchQuery = ref('')
const selectedConference = ref('ALL')
const sortKey = ref('team_name')
const sortDirection = ref('asc')
const currentPage = ref(1)
const pageSize = ref(25)

// ---- GAMES / MODAL STATE ----
const showGamesModal = ref(false)
const selectedTeam = ref(null)
const gamesLoading = ref(false)
const gamesError = ref(null)
const allGames = ref([])

// ---- API LOAD: TEAMS ----
async function loadTeams () {
  loading.value = true
  error.value = null

  try {
    const res = await fetch('https://api.collegefootballranks.com/v1/teams', {
      headers: {
        Accept: 'application/json',
      },
    })

    if (!res.ok) {
      throw new Error(`API error: ${res.status} ${res.statusText}`)
    }

    const json = await res.json()

    // teams endpoint returns a plain array
    teamsRaw.value = Array.isArray(json) ? json : []
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Unable to load teams.'
  } finally {
    loading.value = false
  }
}

// ---- API LOAD: GAMES ----
async function loadGamesIfNeeded () {
  if (allGames.value.length) return // already loaded

  gamesLoading.value = true
  gamesError.value = null

  try {
    const res = await fetch('https://api.collegefootballranks.com/v1/games', {
      headers: {
        Accept: 'application/json',
      },
    })

    if (!res.ok) {
      throw new Error(`API error: ${res.status} ${res.statusText}`)
    }

    const json = await res.json()
    allGames.value = Array.isArray(json) ? json : []
  } catch (err) {
    console.error(err)
    gamesError.value = err?.message || 'Unable to load games.'
  } finally {
    gamesLoading.value = false
  }
}

onMounted(() => {
  loadTeams()
})

// ---- DERIVED METRICS ----
const totalTeams = computed(() => teamsRaw.value.length)

const conferences = computed(() => {
  const set = new Set()
  teamsRaw.value.forEach((t) => {
    if (t.conference) set.add(t.conference)
  })
  return ['ALL', ...Array.from(set).sort()]
})

const totalConferences = computed(() => conferences.value.length - 1)

// map team_id -> team_name for game display
const teamNameById = computed(() => {
  const m = new Map()
  teamsRaw.value.forEach((t) => {
    if (t.team_id != null) m.set(t.team_id, t.team_name)
  })
  return m
})

// ---- FILTERING & SORTING ----
const filteredTeams = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  const conf = selectedConference.value

  return teamsRaw.value.filter((team) => {
    if (conf !== 'ALL' && team.conference !== conf) return false

    if (!q) return true

    const haystack = [
      team.team_name,
      team.mascot,
      team.conference,
    ]
      .filter(Boolean)
      .join(' ')
      .toLowerCase()

    return haystack.includes(q)
  })
})

const numericSortKeys = new Set(['team_id'])

const sortedTeams = computed(() => {
  const key = sortKey.value
  const dir = sortDirection.value

  const arr = [...filteredTeams.value]

  arr.sort((a, b) => {
    const aVal = a[key]
    const bVal = b[key]

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
  Math.max(1, Math.ceil((sortedTeams.value.length || 1) / pageSize.value)),
)

const paginatedTeams = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return sortedTeams.value.slice(start, end)
})

watch([searchQuery, selectedConference, pageSize], () => {
  currentPage.value = 1
})

// ---- UI HELPERS ----
function changeSort (key) {
  if (sortKey.value === key) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortDirection.value = 'asc'
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

// ---- MODAL / GAMES HELPERS ----
function openTeamGames (team) {
  selectedTeam.value = team
  showGamesModal.value = true
  loadGamesIfNeeded()
}

function closeGamesModal () {
  showGamesModal.value = false
}

// NOTE: this assumes the games endpoint includes `home_team_id` and `away_team_id`
// that match the `team_id` from the teams endpoint. Adjust if your field names differ.
const selectedTeamGames = computed(() => {
  if (!selectedTeam.value) return []
  const id = selectedTeam.value.team_id
  if (id == null) return []
  return allGames.value.filter(
    (g) => g.home_team_id === id || g.away_team_id === id,
  )
})

function formatGameDate (game) {
  const raw = game.game_date || game.date || game.gameDate
  if (!raw) return '—'
  const d = new Date(raw)
  if (Number.isNaN(d.getTime())) return raw
  return d.toLocaleDateString(undefined, {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  })
}

function getHomeTeamName (game) {
  const id = game.home_team_id
  if (id == null) return '—'
  return teamNameById.value.get(id) || `Team ${id}`
}

function getAwayTeamName (game) {
  const id = game.away_team_id
  if (id == null) return '—'
  return teamNameById.value.get(id) || `Team ${id}`
}

function formatScore (game) {
  const hs = game.home_score ?? game.home_points
  const as = game.away_score ?? game.away_points
  if (hs == null || as == null) return '—'
  return `${hs} - ${as}`
}
</script>

<template>
  <div class="py-4">
    <!-- HEADER / HERO -->
    <CRow class="align-items-center mb-4">
      <CCol :md="8">
        <div class="d-flex flex-column gap-1">
          <h2 class="fw-bold mb-0">
            College Football Teams
          </h2>
          <p class="text-body-secondary mb-0">
            Teams from
            <span class="fw-semibold">api.collegefootballranks.com</span>.
            Click a team to view its games.
          </p>
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
          @click="loadTeams"
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
              Total Teams
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ totalTeams || '—' }}
              </div>
              <CIcon icon="cil-people" size="xl" class="opacity-50" />
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
              Showing (after filters)
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ sortedTeams.length }}
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
          <CCol :md="4">
            <div class="fw-semibold">
              Teams Table
            </div>
            <small class="text-body-secondary">
              Click a row to view that team’s games.
            </small>
          </CCol>
          <CCol :md="4">
            <CInputGroup>
              <CInputGroupText>
                <CIcon icon="cil-magnifying-glass" />
              </CInputGroupText>
              <CFormInput
                v-model="searchQuery"
                type="text"
                placeholder="Search by team, mascot, or conference..."
              />
            </CInputGroup>
          </CCol>
          <CCol :md="2">
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
          <CCol :md="2">
            <CFormSelect v-model.number="pageSize">
              <option :value="10">10 rows</option>
              <option :value="25">25 rows</option>
              <option :value="50">50 rows</option>
              <option :value="100">100 rows</option>
            </CFormSelect>
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
          Loading teams…
        </div>

        <div
          v-else-if="error"
          class="py-4 px-3 text-center text-danger"
        >
          <div class="fw-semibold mb-1">
            Failed to load teams
          </div>
          <div class="small mb-3">
            {{ error }}
          </div>
          <CButton
            color="primary"
            size="sm"
            @click="loadTeams"
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
                  @click="changeSort('team_name')"
                >
                  Team
                  <span class="ms-1 small">
                    {{ sortIcon('team_name') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="clickable"
                  @click="changeSort('mascot')"
                >
                  Mascot
                  <span class="ms-1 small">
                    {{ sortIcon('mascot') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="clickable"
                  @click="changeSort('conference')"
                >
                  Conference
                  <span class="ms-1 small">
                    {{ sortIcon('conference') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('team_id')"
                >
                  Team ID
                  <span class="ms-1 small">
                    {{ sortIcon('team_id') }}
                  </span>
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>

            <CTableBody>
              <CTableRow
                v-for="team in paginatedTeams"
                :key="team.team_id ?? team.team_name"
                class="cursor-pointer"
                @click="openTeamGames(team)"
              >
                <CTableDataCell>
                  <div class="fw-semibold">
                    {{ team.team_name }}
                  </div>
                </CTableDataCell>
                <CTableDataCell>
                  {{ team.mascot }}
                </CTableDataCell>
                <CTableDataCell>
                  <span class="badge text-bg-light border">
                    {{ team.conference || '—' }}
                  </span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ team.team_id }}
                </CTableDataCell>
              </CTableRow>

              <CTableRow v-if="!paginatedTeams.length">
                <CTableDataCell colspan="4" class="text-center py-4">
                  <div class="fw-semibold mb-1">
                    No teams match your filters.
                  </div>
                  <div class="small text-body-secondary">
                    Try clearing the search or changing the conference.
                  </div>
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>
        </div>
      </CCardBody>

      <!-- PAGINATION -->
      <CCardFooter
        v-if="!loading && !error && sortedTeams.length"
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
              Math.min(currentPage * pageSize, sortedTeams.length)
            }}
          </span>
          of
          <span class="fw-semibold">
            {{ sortedTeams.length }}
          </span>
          teams
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

    <!-- TEAM GAMES MODAL -->
    <CModal
      v-model:visible="showGamesModal"
      size="lg"
      scrollable
    >
      <CModalHeader>
        <CModalTitle>
          {{ selectedTeam ? selectedTeam.team_name : 'Team' }} – Games
        </CModalTitle>
        <CButtonClose @click="closeGamesModal" />
      </CModalHeader>
      <CModalBody>
        <div v-if="gamesLoading" class="text-center py-3">
          <CSpinner class="me-2" />
          Loading games…
        </div>
        <div v-else-if="gamesError" class="text-danger small mb-2">
          {{ gamesError }}
        </div>
        <div v-else>
          <CTable
            v-if="selectedTeamGames.length"
            hover
            striped
            responsive
            align="middle"
            class="mb-0"
          >
            <CTableHead class="text-nowrap">
              <CTableRow>
                <CTableHeaderCell>Date</CTableHeaderCell>
                <CTableHeaderCell>Home</CTableHeaderCell>
                <CTableHeaderCell>Away</CTableHeaderCell>
                <CTableHeaderCell class="text-center">
                  Score
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>
            <CTableBody>
              <CTableRow
                v-for="(game, idx) in selectedTeamGames"
                :key="game.game_id ?? idx"
              >
                <CTableDataCell>
                  {{ formatGameDate(game) }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ getHomeTeamName(game) }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ getAwayTeamName(game) }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatScore(game) }}
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>

          <div
            v-else
            class="text-body-secondary small text-center py-3"
          >
            No games found for this team.
          </div>
        </div>
      </CModalBody>
      <CModalFooter>
        <CButton color="secondary" variant="outline" @click="closeGamesModal">
          Close
        </CButton>
      </CModalFooter>
    </CModal>
  </div>
</template>

<style scoped>
.clickable {
  cursor: pointer;
}

.clickable:hover {
  text-decoration: underline;
}

.cursor-pointer {
  cursor: pointer;
}
</style>
