<script setup>
import { ref, computed, onMounted, watch } from 'vue'

const loading = ref(false)
const error = ref(null)

const teamsRaw = ref([])
const rankingsRaw = ref([])
const allGames = ref([])

const gamesLoading = ref(false)
const gamesError = ref(null)

const searchQuery = ref('')
const selectedConference = ref('ALL')
const sortKey = ref('conference')
const sortDirection = ref('asc')
const currentPage = ref(1)
const pageSize = ref(25)

// ---- MODAL STATE (Conference -> Teams) ----
const showConferenceModal = ref(false)
const selectedConferenceName = ref(null)

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
      throw new Error(`Teams API error: ${res.status} ${res.statusText}`)
    }

    const json = await res.json()
    teamsRaw.value = Array.isArray(json) ? json : []
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Unable to load teams.'
  } finally {
    loading.value = false
  }
}

// ---- API LOAD: RANKER (rank) ----
async function loadRankings () {
  try {
    const res = await fetch('https://api.collegefootballranks.com/v1/ranker', {
      headers: {
        Accept: 'application/json',
      },
    })

    if (!res.ok) {
      throw new Error(`Ranker API error: ${res.status} ${res.statusText}`)
    }

    const json = await res.json()
    // /v1/ranker returns { raw_ranking: [...] }
    rankingsRaw.value = Array.isArray(json) ? json : (json.raw_ranking || [])
  } catch (err) {
    console.error(err)
    // keep page visible even if rankings fail
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
      throw new Error(`Games API error: ${res.status} ${res.statusText}`)
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
  // Load everything needed for conference-level summary
  loadTeams()
  loadRankings()
  loadGamesIfNeeded()
})

// ---- LOOKUPS ----

// team_id -> { rank }
const rankingByTeamId = computed(() => {
  const m = new Map()
  rankingsRaw.value.forEach((r) => {
    const id = r.team_id ?? r.id
    if (id != null) {
      m.set(id, {
        rank: r.rank ?? null,
      })
    }
  })
  return m
})

// teams with rank merged in
const teamsWithRank = computed(() =>
  teamsRaw.value.map((t) => {
    const r = rankingByTeamId.value.get(t.team_id) || {}
    return {
      ...t,
      rank: r.rank ?? null,
    }
  }),
)

// team_id -> conference
const teamConferenceById = computed(() => {
  const m = new Map()
  teamsRaw.value.forEach((t) => {
    if (t.team_id != null && t.conference) {
      m.set(t.team_id, t.conference)
    }
  })
  return m
})

// ---- TEAM-LEVEL GAME STATS (for avg points per game) ----
const teamGameStats = computed(() => {
  const stats = new Map() // team_id -> { pointsFor, games }

  function ensureTeam (id) {
    if (!stats.has(id)) {
      stats.set(id, { pointsFor: 0, games: 0 })
    }
    return stats.get(id)
  }

  allGames.value.forEach((g) => {
    const homeId = g.home_team_id
    const awayId = g.away_team_id
    const hs = g.home_score ?? g.home_points
    const as = g.away_score ?? g.away_points

    if (homeId == null || awayId == null) return
    if (hs == null || as == null) return

    // Home team
    {
      const s = ensureTeam(homeId)
      s.pointsFor += hs
      s.games += 1
    }

    // Away team
    {
      const s = ensureTeam(awayId)
      s.pointsFor += as
      s.games += 1
    }
  })

  return stats
})

function getTeamAvgPoints (teamId) {
  if (teamId == null) return null
  const s = teamGameStats.value.get(teamId)
  if (!s || !s.games) return null
  return s.pointsFor / s.games
}

// ---- GAME-BASED CONFERENCE STATS (avg margin vs non-conf) ----
const conferenceGameStats = computed(() => {
  const stats = new Map() // conf -> { totalMargin, games }

  function ensureConf (conf) {
    if (!stats.has(conf)) {
      stats.set(conf, { totalMargin: 0, games: 0 })
    }
    return stats.get(conf)
  }

  const confById = teamConferenceById.value

  allGames.value.forEach((g) => {
    const homeId = g.home_team_id
    const awayId = g.away_team_id
    const hs = g.home_score ?? g.home_points
    const as = g.away_score ?? g.away_points

    if (homeId == null || awayId == null) return
    if (hs == null || as == null) return

    const homeConf = confById.get(homeId)
    const awayConf = confById.get(awayId)
    if (!homeConf || !awayConf) return

    // Only consider non-conference matchups
    if (homeConf === awayConf) return

    // From home conference perspective
    {
      const s = ensureConf(homeConf)
      s.totalMargin += (hs - as)
      s.games += 1
    }

    // From away conference perspective
    {
      const s = ensureConf(awayConf)
      s.totalMargin += (as - hs)
      s.games += 1
    }
  })

  return stats
})

// ---- CONFERENCE AGGREGATION ----
const conferenceRows = computed(() => {
  const rows = []
  const gameStats = conferenceGameStats.value
  const teamStats = teamGameStats.value

  // Group teams by conference
  const byConf = new Map()
  teamsWithRank.value.forEach((t) => {
    const conf = t.conference || 'Independent'
    if (!byConf.has(conf)) byConf.set(conf, [])
    byConf.get(conf).push(t)
  })

  for (const [conf, teamArr] of byConf.entries()) {
    const numTeams = teamArr.length

    const ranks = teamArr
      .map((t) => (typeof t.rank === 'number' ? t.rank : null))
      .filter((r) => r != null)

    const avgRank =
      ranks.length > 0
        ? ranks.reduce((a, b) => a + b, 0) / ranks.length
        : null

    const gs = gameStats.get(conf)
    const avgPointDiff =
      gs && gs.games > 0 ? gs.totalMargin / gs.games : null

    // Average points per game for the conference:
    // sum(pointsFor for all teams in conf) / sum(games for those teams)
    let totalPoints = 0
    let totalGames = 0
    teamArr.forEach((t) => {
      const s = teamStats.get(t.team_id)
      if (s && s.games > 0) {
        totalPoints += s.pointsFor
        totalGames += s.games
      }
    })
    const avgPointsFor =
      totalGames > 0 ? totalPoints / totalGames : null

    rows.push({
      conference: conf,
      numTeams,
      avgRank,
      avgPointDiff,
      avgPointsFor,
    })
  }

  return rows
})

// ---- DERIVED METRICS ----
const totalTeams = computed(() => teamsRaw.value.length)
const totalConferences = computed(() => conferenceRows.value.length)

// For dropdown
const conferenceOptions = computed(() => [
  'ALL',
  ...conferenceRows.value
    .map((r) => r.conference)
    .sort((a, b) => a.localeCompare(b)),
])

// ---- FILTERING & SORTING ----
const filteredConferences = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  const confFilter = selectedConference.value

  return conferenceRows.value.filter((row) => {
    if (confFilter !== 'ALL' && row.conference !== confFilter) return false
    if (!q) return true
    return row.conference.toLowerCase().includes(q)
  })
})

const numericSortKeys = new Set([
  'numTeams',
  'avgRank',
  'avgPointDiff',
  'avgPointsFor',
])

const sortedConferences = computed(() => {
  const key = sortKey.value
  const dir = sortDirection.value
  const arr = [...filteredConferences.value]

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
  Math.max(1, Math.ceil((sortedConferences.value.length || 1) / pageSize.value)),
)

const paginatedConferences = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return sortedConferences.value.slice(start, end)
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

function formatNumber (val, digits = 3) {
  if (val == null || Number.isNaN(val)) return '—'
  return Number(val).toFixed(digits)
}

// ---- CONFERENCE MODAL HELPERS ----
function openConferenceModal (row) {
  selectedConferenceName.value = row.conference
  showConferenceModal.value = true
}

function closeConferenceModal () {
  showConferenceModal.value = false
}

// Teams in the selected conference with rank + avg points per game
const selectedConferenceTeams = computed(() => {
  const conf = selectedConferenceName.value
  if (!conf) return []

  return teamsWithRank.value
    .filter((t) => (t.conference || 'Independent') === conf)
    .map((t) => ({
      ...t,
      avgPointsFor: getTeamAvgPoints(t.team_id),
    }))
    .sort((a, b) => {
      const ar = a.rank ?? 9999
      const br = b.rank ?? 9999
      return ar - br
    })
})
</script>

<template>
  <div class="py-4">
    <!-- HEADER / HERO -->
    <CRow class="align-items-center mb-4">
      <CCol :md="8">
        <div class="d-flex flex-column gap-1">
          <h2 class="fw-bold mb-0">
            College Football – Conference Summary
          </h2>
          <p class="text-body-secondary mb-0">
            Aggregated by conference using
            <span class="fw-semibold">api.collegefootballranks.com</span>.
          </p>
          <small class="text-body-secondary">
            Average rank from <code>/v1/ranker</code>;
            average point diff vs non-conference opponents and average points
            per game from <code>/v1/games</code>.
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
          @click="() => { loadTeams(); loadRankings(); loadGamesIfNeeded(); }"
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
              Total Conferences
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
              Showing (after filters)
            </div>
            <div class="d-flex align-items-end justify-content-between">
              <div class="fs-3 fw-bold">
                {{ sortedConferences.length }}
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
              Conference Table
            </div>
            <small class="text-body-secondary">
              Click a conference name to see member teams with rank and scoring.
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
                placeholder="Search by conference name..."
              />
            </CInputGroup>
          </CCol>
          <CCol :md="2">
            <CFormSelect v-model="selectedConference">
              <option
                v-for="conf in conferenceOptions"
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
            @click="() => { loadTeams(); loadRankings(); loadGamesIfNeeded(); }"
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
                  @click="changeSort('conference')"
                >
                  Conference
                  <span class="ms-1 small">
                    {{ sortIcon('conference') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('numTeams')"
                >
                  # Teams
                  <span class="ms-1 small">
                    {{ sortIcon('numTeams') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('avgRank')"
                >
                  Avg Rank
                  <span class="ms-1 small">
                    {{ sortIcon('avgRank') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('avgPointDiff')"
                >
                  Avg Point Diff (vs non-conf)
                  <span class="ms-1 small">
                    {{ sortIcon('avgPointDiff') }}
                  </span>
                </CTableHeaderCell>
                <CTableHeaderCell
                  class="text-center clickable"
                  @click="changeSort('avgPointsFor')"
                >
                  Avg Pts/G (Conf)
                  <span class="ms-1 small">
                    {{ sortIcon('avgPointsFor') }}
                  </span>
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>

            <CTableBody>
              <CTableRow
                v-for="row in paginatedConferences"
                :key="row.conference"
              >
                <CTableDataCell>
                  <span
                    class="fw-semibold clickable text-primary text-decoration-underline"
                    @click="openConferenceModal(row)"
                  >
                    {{ row.conference }}
                  </span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ row.numTeams }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatNumber(row.avgRank, 1) }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatNumber(row.avgPointDiff, 1) }}
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatNumber(row.avgPointsFor, 1) }}
                </CTableDataCell>
              </CTableRow>

              <CTableRow v-if="!paginatedConferences.length">
                <CTableDataCell colspan="5" class="text-center py-4">
                  <div class="fw-semibold mb-1">
                    No conferences match your filters.
                  </div>
                  <div class="small text-body-secondary">
                    Try clearing the search or changing the conference filter.
                  </div>
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>
        </div>
      </CCardBody>

      <!-- PAGINATION -->
      <CCardFooter
        v-if="!loading && !error && sortedConferences.length"
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
              Math.min(currentPage * pageSize, sortedConferences.length)
            }}
          </span>
          of
          <span class="fw-semibold">
            {{ sortedConferences.length }}
          </span>
          conferences
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

    <!-- OPTIONAL NOTE IF GAMES FAILED -->
    <div
      v-if="gamesError"
      class="mt-2 small text-danger"
    >
      Note: game data failed to load, so average point differentials and
      points-per-game metrics may be missing.
    </div>

    <!-- CONFERENCE TEAMS MODAL -->
    <CModal
      v-model:visible="showConferenceModal"
      size="lg"
      scrollable
    >
      <CModalHeader>
        <CModalTitle>
          {{ selectedConferenceName || 'Conference' }} – Teams
        </CModalTitle>
        <CButtonClose @click="closeConferenceModal" />
      </CModalHeader>
      <CModalBody>
        <div v-if="gamesLoading" class="text-center py-3">
          <CSpinner class="me-2" />
          Loading game stats…
        </div>
        <div v-else-if="gamesError" class="text-danger small mb-2">
          {{ gamesError }}
        </div>
        <div v-else>
          <CTable
            v-if="selectedConferenceTeams.length"
            hover
            striped
            responsive
            align="middle"
            class="mb-0"
          >
            <CTableHead class="text-nowrap">
              <CTableRow>
                <CTableHeaderCell>Team</CTableHeaderCell>
                <CTableHeaderCell class="text-center">
                  Rank
                </CTableHeaderCell>
                <CTableHeaderCell class="text-center">
                  Avg Pts/G
                </CTableHeaderCell>
              </CTableRow>
            </CTableHead>
            <CTableBody>
              <CTableRow
                v-for="team in selectedConferenceTeams"
                :key="team.team_id ?? team.team_name"
              >
                <CTableDataCell>
                  <div class="fw-semibold">
                    {{ team.team_name }}
                  </div>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  <span
                    v-if="team.rank != null"
                    class="pill-rank"
                  >
                    {{ team.rank }}
                  </span>
                  <span v-else>—</span>
                </CTableDataCell>
                <CTableDataCell class="text-center">
                  {{ formatNumber(team.avgPointsFor, 1) }}
                </CTableDataCell>
              </CTableRow>
            </CTableBody>
          </CTable>

          <div
            v-else
            class="text-body-secondary small text-center py-3"
          >
            No teams found for this conference.
          </div>
        </div>
      </CModalBody>
      <CModalFooter>
        <CButton color="secondary" variant="outline" @click="closeConferenceModal">
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

/* Orange rank pill */
.pill-rank {
  background-color: #fd7e14;
  color: #fff;
  border-radius: 999px;
  font-size: 0.75rem;
  padding: 0.15rem 0.5rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
</style>
