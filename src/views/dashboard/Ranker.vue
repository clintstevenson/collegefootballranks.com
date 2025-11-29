<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const loading = ref(false)
const error = ref(null)

const latestGameDate = ref(null)
const homeFieldAdv = ref(null) // existing HFA metric
const homeFieldAdvProb = ref(null) // global equal-team HFA probability (still used in modal header)
const rankingsRaw = ref([])

const teamsRaw = ref([])
const gamesRaw = ref([])

const searchQuery = ref('')
const selectedConference = ref('ALL')
const sortKey = ref('rank')
const sortDirection = ref('asc')
const currentPage = ref(1)
const pageSize = ref(25)

// ---- MODAL STATE ----
const showGamesModal = ref(false)
const activeTeamId = ref(null) // team_id from the clicked row

// ---- API LOADS ----
async function loadRankings () {
  loading.value = true
  error.value = null

  try {
    // Using /v1/ranker as the data source
    const res = await fetch('https://api.collegefootballranks.com/v1/ranker', {
      headers: {
        Accept: 'application/json',
      },
    })

    if (!res.ok) {
      throw new Error(`API error: ${res.status} ${res.statusText}`)
    }

    const json = await res.json()

    // /v1/ranker returns { raw_ranking: [...] } plus globals
    rankingsRaw.value = Array.isArray(json) ? json : (json.raw_ranking || [])

    latestGameDate.value = json.latest_game_date || null
    homeFieldAdv.value = json.home_field_adv ?? null
    homeFieldAdvProb.value = json.home_field_equal_team_win_prob ?? null
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Unable to load rankings.'
  } finally {
    loading.value = false
  }
}

async function loadTeams () {
  try {
    const res = await fetch('https://api.collegefootballranks.com/v1/teams', {
      headers: { Accept: 'application/json' },
    })
    if (!res.ok) {
      throw new Error(`Teams API error: ${res.status} ${res.statusText}`)
    }
    const json = await res.json()
    teamsRaw.value = Array.isArray(json) ? json : (json.teams || [])
  } catch (err) {
    console.error(err)
  }
}

async function loadGames () {
  try {
    const res = await fetch('https://api.collegefootballranks.com/v1/games', {
      headers: { Accept: 'application/json' },
    })
    if (!res.ok) {
      throw new Error(`Games API error: ${res.status} ${res.statusText}`)
    }
    const json = await res.json()
    gamesRaw.value = Array.isArray(json) ? json : (json.games || [])
  } catch (err) {
    console.error(err)
  }
}

onMounted(async () => {
  await Promise.all([loadRankings(), loadTeams(), loadGames()])
})

// ---- LOOKUP MAPS ----
const teamsById = computed(() => {
  const map = Object.create(null)
  for (const t of teamsRaw.value) {
    const id = t.team_id ?? t.id
    if (id != null) {
      map[String(id)] = t
    }
  }
  return map
})

// Build a lookup from team_id and team_name to rank + sos_score
const rankingsLookup = computed(() => {
  const byId = Object.create(null)
  const byName = Object.create(null)

  for (const t of rankingsRaw.value) {
    const id = t.team_id ?? t.id
    const name = t.team_name
    const rankVal = t.rank ?? null
    const sosScore = t.sos_score ?? null

    const payload = { rank: rankVal, sos_score: sosScore }

    if (id != null) {
      byId[String(id)] = payload
    }
    if (name) {
      byName[name.toLowerCase()] = payload
    }
  }

  return { byId, byName }
})

function getRankingForTeam (teamId, teamName) {
  const { byId, byName } = rankingsLookup.value

  if (teamId != null && byId[String(teamId)]) {
    return byId[String(teamId)]
  }

  if (teamName) {
    const key = teamName.toLowerCase()
    if (byName[key]) return byName[key]
  }

  return { rank: null, sos_score: null }
}

// ---- DERIVED METRICS ----
const totalTeams = computed(() => rankingsRaw.value.length)

const conferences = computed(() => {
  const set = new Set()
  rankingsRaw.value.forEach((t) => {
    if (t.conference) set.add(t.conference)
  })
  return ['ALL', ...Array.from(set).sort()]
})

const totalConferences = computed(() => conferences.value.length - 1)

// ---- FILTERING & SORTING ----
const filteredTeams = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  const conf = selectedConference.value

  return rankingsRaw.value.filter((team) => {
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

const numericSortKeys = new Set([
  'rank',
  'wins',
  'losses',
  'games',
  'win_pct',
  'sos_tough_opponents',
  'sos_score',
  'home_field_equal_team_win_prob',
])

const sortedTeams = computed(() => {
  const key = sortKey.value
  const dir = sortDirection.value

  const arr = [...filteredTeams.value]

  arr.sort((a, b) => {
    let aVal
    let bVal

    if (key === 'rank') {
      aVal = a.rank ?? 9999
      bVal = b.rank ?? 9999
    } else {
      aVal = a[key]
      bVal = b[key]
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
  Math.max(1, Math.ceil(sortedTeams.value.length / pageSize.value || 1)),
)

const paginatedTeams = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return sortedTeams.value.slice(start, end)
})

watch([searchQuery, selectedConference, pageSize], () => {
  currentPage.value = 1
})

// ---- MODAL: ACTIVE TEAM INFO ----
const activeTeamName = computed(() => {
  if (activeTeamId.value == null) return ''
  const idStr = String(activeTeamId.value)
  const fromRanker = rankingsRaw.value.find(
    (t) => String(t.team_id ?? t.id) === idStr,
  )
  if (fromRanker?.team_name) return fromRanker.team_name

  const fromTeams = teamsById.value[idStr]
  return fromTeams?.team_name || fromTeams?.name || ''
})

const activeTeamGames = computed(() => {
  if (activeTeamId.value == null) return []

  const idStr = String(activeTeamId.value)

  return gamesRaw.value
    .filter(
      (g) =>
        String(g.home_team_id) === idStr ||
        String(g.away_team_id) === idStr,
    )
    .map((g) => {
      const isHome = String(g.home_team_id) === idStr
      const oppTeamId = isHome ? g.away_team_id : g.home_team_id

      const homeTeam = teamsById.value[String(g.home_team_id)] || {}
      const awayTeam = teamsById.value[String(g.away_team_id)] || {}

      const oppTeam = isHome ? awayTeam : homeTeam

      const oppRanking = getRankingForTeam(
        oppTeamId,
        oppTeam.team_name || oppTeam.name,
      )

      const dateRaw = g.game_date || g.date
      const dateObj = dateRaw ? new Date(dateRaw) : new Date(NaN)

      return {
        ...g,
        dateObj,
        isHome,
        homeName: homeTeam.team_name || homeTeam.name || `Team ${g.home_team_id}`,
        awayName: awayTeam.team_name || awayTeam.name || `Team ${g.away_team_id}`,
        oppName: oppTeam.team_name || oppTeam.name || `Team ${oppTeamId}`,
        oppRank: oppRanking.rank,
        oppSosScore: oppRanking.sos_score,
      }
    })
    .sort((a, b) => b.dateObj - a.dateObj) // DESC on Date
})

// ---- UI HELPERS ----
function changeSort (key) {
  if (sortKey.value === key) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortKey.value = key
    sortDirection.value = key === 'rank' ? 'asc' : 'desc'
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

function formatWinPct (team) {
  if (team.win_pct == null) return '–'
  return (team.win_pct * 100).toFixed(1) + '%'
}

function formatRecord (team) {
  const w = team.wins ?? 0
  const l = team.losses ?? 0
  return `${w}-${l}`
}

function formatNumber (val, digits = 3) {
  if (val == null) return '–'
  return Number(val).toFixed(digits)
}

function formatInteger (val) {
  if (val == null) return '–'
  return Math.round(Number(val))
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

function openTeamModal (team) {
  const id = team.team_id ?? team.id
  if (id == null) return
  activeTeamId.value = id
  showGamesModal.value = true
}
</script>

<template>
  <div class="py-4">
    <!-- HEADER / HERO -->
    <CRow class="align-items-center mb-4">
      <CCol :md="8">
        <div class="d-flex flex-column gap-1">
          <h2 class="fw-bold mb-0">
            College Football Model Rankings
          </h2>
          <p class="text-body-secondary mb-0">
            Live rankings powered by
            <span class="fw-semibold">api.collegefootballranks.com</span>.
            Sort, search, and slice the field by conference.
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
          @click="() => { loadRankings(); loadGames(); loadTeams(); }"
        >
          <CIcon icon="cil-reload" class="me-1" />
          Refresh
        </CButton>
      </CCol>
    </CRow>

    <!-- STATS CARDS -->
    <CRow class="mb-4 g-3">
      <CCol :sm="6" :lg="3">
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

      <CCol :sm="6" :lg="3">
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

      <CCol :sm="6" :lg="3">
        <CCard class="h-100 border-0 shadow-sm">
          <CCardBody class="d-flex flex-column justify-content-between">
            <div class="text-body-secondary small mb-1">
              Home Field Advantage
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ homeFieldAdv != null ? homeFieldAdv.toFixed(3) : '—' }}
              </div>
              <CIcon icon="cil-home" size="xl" class="opacity-50" />
            </div>
          </CCardBody>
        </CCard>
      </CCol>

      <CCol :sm="6" :lg="3">
        <CCard class="h-100 border-0 shadow-sm">
          <CCardBody class="d-flex flex-column justify-content-between">
            <div class="text-body-secondary small mb-1">
              Showing
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
              Rankings Table
            </div>
            <small class="text-body-secondary">
              Use search and conference filters to explore.
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
          Loading rankings…
        </div>

        <div
          v-else-if="error"
          class="py-4 px-3 text-center text-danger"
        >
          <div class="fw-semibold mb-1">
            Failed to load rankings
          </div>
          <div class="small mb-3">
            {{ error }}
          </div>
          <CButton
            color="primary"
            size="sm"
            @click="() => { loadRankings(); loadGames(); loadTeams(); }"
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
                  class="text-center clickable"
                  @click="changeSort('rank')"
                >
                  #
                  <span class="ms-1 small">
                    {{ sortIcon('rank') }}
                  </span>
                </CTableHeaderCell>
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
                  @click="changeSort('conference')"
                >
                  Conference
                  <span class="ms-1 small">
                    {{ sortIcon('conference') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('wins')"
                >
                  Record
                  <span class="ms-1 small">
                    {{ sortIcon('wins') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('win_pct')"
                >
                  Win %
                  <span class="ms-1 small">
                    {{ sortIcon('win_pct') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('home_field_equal_team_win_prob')"
                >
                  HFA Prob
                  <span class="ms-1 small">
                    {{ sortIcon('home_field_equal_team_win_prob') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('sos_tough_opponents')"
                >
                  SoS Tough Opp.
                  <span class="ms-1 small">
                    {{ sortIcon('sos_tough_opponents') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('sos_score')"
                >
                  SoS Score
                  <span class="ms-1 small">
                    {{ sortIcon('sos_score') }}
                  </span>
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>

            <CTableBody>
              <CTableRow
                v-for="(team, idx) in paginatedTeams"
                :key="team.team_id ?? team.team_name ?? idx"
              >
                <CTableDataCell class="text-center fw-semibold">
                  {{ team.rank ?? (idx + 1 + (currentPage - 1) * pageSize) }}
                </CTableDataCell>
                <CTableDataCell class="clickable" @click="openTeamModal(team)">
                  <div class="fw-semibold">
                    {{ team.team_name }}
                  </div>
                  <div class="small text-body-secondary">
                    {{ team.mascot }}
                  </div>
                </CTableDataCell>
                <CTableDataCell>
                  <span class="badge text-bg-light border">
                    {{ team.conference || 'Independent' }}
                  </span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <div class="fw-semibold">
                    {{ formatRecord(team) }}
                  </div>
                  <div class="small text-body-secondary">
                    {{ team.games ?? (team.wins ?? 0) + (team.losses ?? 0) }} games
                  </div>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatWinPct(team) }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <!-- HFA probability per team, 2 decimals -->
                  {{
                    team.home_field_equal_team_win_prob != null
                      ? Number(team.home_field_equal_team_win_prob).toFixed(2)
                      : '–'
                  }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <!-- integer SoS Tough Opp. -->
                  {{ formatInteger(team.sos_tough_opponents) }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <!-- SoS score to 2 decimals -->
                  {{ formatNumber(team.sos_score, 2) }}
                </CTableDataCell>
              </CTableRow>

              <CTableRow v-if="!paginatedTeams.length">
                <CTableDataCell colspan="8" class="text-center py-4">
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
            {{ Math.min(currentPage * pageSize, sortedTeams.length) }}
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

    <!-- GAMES MODAL -->
    <CModal
      :visible="showGamesModal"
      size="lg"
      alignment="center"
      scrollable
      @close="showGamesModal = false"
    >
      <CModalHeader>
        <CModalTitle>
          Games for
          <span class="fw-semibold">{{ activeTeamName || 'Selected Team' }}</span>
          <!-- show global HFA probability in modal for context -->
          <span
            v-if="homeFieldAdvProb != null"
            class="ms-3 small text-body-secondary"
          >
            HFA Prob (equal teams):
            <span class="fw-semibold">
              {{ homeFieldAdvProb.toFixed(3) }}
            </span>
          </span>
        </CModalTitle>
      </CModalHeader>
      <CModalBody>
        <div v-if="!activeTeamGames.length" class="text-center text-body-secondary py-3">
          No games found for this team.
        </div>
        <div v-else>
          <CTable
            striped
            hover
            small
            responsive
            align="middle"
          >
            <CTableHead class="text-nowrap bg-body-secondary bg-opacity-50">
              <CTableRow>
                <CTableHeaderCell>Date</CTableHeaderCell>
                <CTableHeaderCell>Home Team</CTableHeaderCell>
                <CTableHeaderCell>Away Team</CTableHeaderCell>
                <CTableHeaderCell class="text-center">Score</CTableHeaderCell>
                <CTableHeaderCell>Opponent (Rank)</CTableHeaderCell>
                <CTableHeaderCell class="text-center">Opp SoS Score</CTableHeaderCell>
              </CTableRow>
            </CTableHead>
            <CTableBody>
              <CTableRow
                v-for="g in activeTeamGames"
                :key="g.game_id ?? g.id"
              >
                <CTableDataCell>
                  {{ formatDate(g.game_date || g.date) }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ g.homeName }}
                </CTableDataCell>
                <CTableDataCell>
                  {{ g.awayName }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <span class="fw-semibold">
                    {{ g.home_score }} – {{ g.away_score }}
                  </span>
                </CTableDataCell>
                <CTableDataCell>
                  <span class="fw-semibold">
                    {{ g.oppName }}
                  </span>
                  <span v-if="g.oppRank != null" class="ms-2">
                    <span
                      :class="[
                        'rank-pill',
                        g.oppRank >= 1 && g.oppRank <= 25
                          ? 'rank-pill-top25'
                          : 'rank-pill-other'
                      ]"
                    >
                      {{ g.oppRank }}
                    </span>
                  </span>
                  <span v-else class="ms-2 text-body-secondary">–</span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{
                    g.oppSosScore != null
                      ? formatNumber(g.oppSosScore, 2)
                      : '–'
                  }}
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>
        </div>
      </CModalBody>
      <CModalFooter>
        <CButton color="secondary" @click="showGamesModal = false">
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

/* Rank pills in modal */
.rank-pill {
  border-radius: 999px;
  padding: 0.15rem 0.5rem;
  font-size: 0.75rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.rank-pill-top25 {
  background-color: #fd7e14; /* orange */
  color: #fff;
}

.rank-pill-other {
  background-color: #343a40; /* dark gray */
  color: #fff;
}
</style>
