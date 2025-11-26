<script setup>
import { ref, computed, onMounted } from 'vue'
import { CChartBar } from '@coreui/vue-chartjs'

const loading = ref(false)
const error = ref(null)

const gamesRaw = ref([])
const rankingsRaw = ref([])

async function loadData () {
  loading.value = true
  error.value = null

  try {
    const [gamesRes, rankerRes] = await Promise.all([
      fetch('https://api.collegefootballranks.com/v1/games', {
        headers: { Accept: 'application/json' },
      }),
      fetch('https://api.collegefootballranks.com/v1/ranker', {
        headers: { Accept: 'application/json' },
      }),
    ])

    if (!gamesRes.ok) {
      throw new Error(`Games API error: ${gamesRes.status} ${gamesRes.statusText}`)
    }
    if (!rankerRes.ok) {
      throw new Error(`Ranker API error: ${rankerRes.status} ${rankerRes.statusText}`)
    }

    const gamesJson = await gamesRes.json()
    const rankerJson = await rankerRes.json()

    gamesRaw.value = Array.isArray(gamesJson) ? gamesJson : []
    rankingsRaw.value = Array.isArray(rankerJson)
      ? rankerJson
      : (rankerJson.raw_ranking || [])
  } catch (err) {
    console.error(err)
    error.value = err?.message || 'Unable to load games/rankings.'
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadData()
})

// ---- TEAM GAME STATS (POINTS & GAMES) ----
const teamGameStats = computed(() => {
  const stats = new Map()

  function addGamePoints (teamId, points) {
    if (teamId == null || points == null) return
    if (!stats.has(teamId)) {
      stats.set(teamId, { points: 0, games: 0 })
    }
    const obj = stats.get(teamId)
    obj.points += Number(points) || 0
    obj.games += 1
  }

  for (const g of gamesRaw.value) {
    const homeId = g.home_team_id
    const awayId = g.away_team_id
    const homeScore = g.home_score ?? g.home_points
    const awayScore = g.away_score ?? g.away_points

    if (homeScore != null) addGamePoints(homeId, homeScore)
    if (awayScore != null) addGamePoints(awayId, awayScore)
  }

  return stats
})

// ---- TOP 25 TEAMS BY RATING (DESC) ----
const top25ByRating = computed(() => {
  if (!rankingsRaw.value.length) return []

  const arr = rankingsRaw.value
    .map((t) => ({
      ...t,
      _rating: Number(t.rating ?? 0),
    }))
    .sort((a, b) => b._rating - a._rating) // highest rating → lowest

  return arr.slice(0, 25)
})

// ---- MERGE RATING + GAME STATS ----
const top25WithPpg = computed(() => {
  const statsMap = teamGameStats.value
  return top25ByRating.value.map((t, idx) => {
    const s = statsMap.get(t.team_id) || { points: 0, games: 0 }
    const ppg = s.games > 0 ? s.points / s.games : 0
    return {
      team_id: t.team_id,
      team_name: t.team_name,
      conference: t.conference,
      rating: t._rating,
      rating_rank: idx + 1, // rank by rating
      points: s.points,
      games: s.games,
      ppg,
    }
  })
})

// ---- NEW: SORT FOR CHART LEFT → RIGHT BY PPG (ASC) ----
const sortedTop25ForChart = computed(() => {
  const arr = [...top25WithPpg.value]
  arr.sort((a, b) => a.ppg - b.ppg) // lowest PPG on left, highest on right
  return arr
})

const overallAvgPPG = computed(() => {
  if (!top25WithPpg.value.length) return null
  const sum = top25WithPpg.value.reduce((acc, t) => acc + t.ppg, 0)
  return sum / top25WithPpg.value.length
})

// ---- CHART DATA ----
const chartData = computed(() => {
  const labels = sortedTop25ForChart.value.map((t) => {
    return `${t.rating_rank}. ${t.team_name}` // keep label short-ish
  })

  const data = sortedTop25ForChart.value.map((t) => Number(t.ppg.toFixed(2)))

  return {
    labels,
    datasets: [
      {
        label: 'Avg Points per Game',
        data,
        backgroundColor: 'rgba(40, 167, 69, 0.7)',
        borderColor: 'rgba(33, 136, 56, 1)',
        borderWidth: 1,
        borderRadius: 4,
        hoverBackgroundColor: 'rgba(32, 201, 151, 0.9)',
      },
    ],
  }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      display: true,
      position: 'top',
    },
    tooltip: {
      callbacks: {
        label (ctx) {
          const team = sortedTop25ForChart.value[ctx.dataIndex]
          if (!team) {
            return `${ctx.dataset.label}: ${ctx.parsed.y.toFixed(2)} PPG`
          }
          const ppgPart = `${team.ppg.toFixed(2)} PPG`
          const volPart = `${team.points} pts in ${team.games} games`
          const ratingPart = `rating ${team.rating.toFixed(3)} (rating rank ${team.rating_rank})`
          return `${team.team_name}: ${ppgPart} (${volPart}, ${ratingPart})`
        },
      },
    },
  },
  scales: {
    x: {
      ticks: {
        autoSkip: false,
        maxRotation: 60,
        minRotation: 45,
        font: {
          size: 11,
        },
      },
      grid: {
        display: false,
      },
    },
    y: {
      beginAtZero: true,
      title: {
        display: true,
        text: 'Points per Game',
      },
    },
  },
}
</script>

<template>
  <div class="py-4">
    <CRow class="mb-3">
      <CCol>
        <h2 class="fw-bold mb-1">
          Top 25 by Rating · Points per Game
        </h2>
        <p class="text-body-secondary mb-0">
          Top 25 teams from <code>/v1/ranker</code> (by <code>rating</code>), graphed by
          <strong>average points per game</strong> from <code>/v1/games</code>.
          Bars are sorted left → right from lowest to highest PPG.
        </p>
        <small
          v-if="overallAvgPPG !== null"
          class="text-body-secondary"
        >
          Avg PPG among these teams:
          <strong>{{ overallAvgPPG.toFixed(2) }}</strong>
        </small>
      </CCol>
    </CRow>

    <CRow>
      <CCol>
        <CCard class="border-0 shadow-sm">
          <CCardBody>
            <div
              v-if="loading"
              class="py-5 text-center text-body-secondary"
            >
              <CSpinner class="me-2" />
              Loading chart…
            </div>

            <div
              v-else-if="error"
              class="py-4 text-center text-danger"
            >
              <div class="fw-semibold mb-2">
                Couldn’t load chart data
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

            <div
              v-else-if="!sortedTop25ForChart.length"
              class="py-4 text-center text-body-secondary small"
            >
              No ranked teams available yet.
            </div>

            <div
              v-else
              class="chart-container"
            >
              <CChartBar
                :data="chartData"
                :options="chartOptions"
              />
            </div>
          </CCardBody>
        </CCard>
      </CCol>
    </CRow>
  </div>
</template>

<style scoped>
.chart-container {
  position: relative;
  height: 800px;
}
</style>
