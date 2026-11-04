<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const TZ = 'Asia/Kolkata'
const items = ref([])
const error = ref('')
const loaded = ref(false)
const now = ref(new Date())
const filter = ref('all')
let timer

const KINDS = { assignment: 'Graded assignment', project: 'Take-home project' }
const FILTERS = [
  ['all', 'All'],
  ['assignment', 'Graded assignments'],
  ['project', 'Take-home projects'],
]

function parseXml(text) {
  const doc = new DOMParser().parseFromString(text, 'application/xml')
  if (doc.querySelector('parsererror')) throw new Error('XML parse error')
  return [...doc.querySelectorAll('deadline')].map((el) => {
    const pad = (n) => String(n).padStart(2, '0')
    const date = (el.getAttribute('date') || '').trim().split('-').map(pad).join('-')
    const time = (el.getAttribute('time') || '23:59').trim().split(':').map(pad).join(':')
    return {
      date,
      time,
      due: new Date(`${date}T${time}:00+05:30`),
      kind: el.getAttribute('kind') || 'assignment',
      title: el.getAttribute('title') || '',
      part: el.getAttribute('part') || '',
      marks: el.getAttribute('marks') || '',
      url: /^https?:\/\//.test(el.getAttribute('url') || '') ? el.getAttribute('url') : '',
    }
  })
}
async function load(url) {
  const res = await fetch(url, { cache: 'no-cache' })
  if (!res.ok) throw new Error(res.status)
  return parseXml(await res.text())
}

onMounted(async () => {
  timer = setInterval(() => (now.value = new Date()), 60_000)
  let list = []
  try {
    list = await load('/deadlines.xml')
  } catch (e) {
    error.value = 'Could not read deadlines.xml. Check the file for typos.'
  }
  const seen = new Set()
  items.value = list
    .filter((d) => {
      if (isNaN(d.due)) console.warn('Skipped deadline with invalid date/time:', d.title, d.part, d.date, d.time)
      return !isNaN(d.due)
    })
    .filter((d) => {
      const k = `${d.title}|${d.date}|${d.time}`
      return seen.has(k) ? false : (seen.add(k), true)
    })
    .sort((a, b) => a.due - b.due)
  loaded.value = true
})
onUnmounted(() => clearInterval(timer))

const todayStr = computed(() => now.value.toLocaleDateString('en-CA', { timeZone: TZ }))
const dayDiff = (dateStr) =>
  Math.round((Date.parse(dateStr + 'T00:00:00Z') - Date.parse(todayStr.value + 'T00:00:00Z')) / 864e5)

function group(list) {
  const map = new Map()
  for (const d of list) {
    if (!map.has(d.date)) map.set(d.date, [])
    map.get(d.date).push(d)
  }
  return [...map].map(([date, rows]) => ({
    date,
    rows,
    label: new Date(date + 'T00:00:00Z').toLocaleDateString('en-GB', {
      weekday: 'short', day: 'numeric', month: 'short', timeZone: 'UTC',
    }),
    diff: dayDiff(date),
  }))
}

const matches = (d) => filter.value === 'all' || d.kind === filter.value
const upcoming = computed(() => group(items.value.filter((d) => d.due >= now.value && matches(d))))
const past = computed(() => group(items.value.filter((d) => d.due < now.value && matches(d))).reverse())
const pastCount = computed(() => past.value.reduce((n, d) => n + d.rows.length, 0))
const upcomingCount = computed(() => items.value.filter((d) => d.due >= now.value).length)

// Every deadline on the same day as the next one.
const nextItems = computed(() => {
  const first = items.value.find((d) => d.due >= now.value)
  return first ? items.value.filter((d) => d.due >= now.value && d.date === first.date) : []
})
const next = computed(() => nextItems.value[0])

function rel(diff) {
  if (diff === 0) return 'Today'
  if (diff === 1) return 'Tomorrow'
  if (diff === -1) return 'Yesterday'
  return diff > 0 ? `In ${diff} days` : `${-diff} days ago`
}
function clock(d) {
  return d.due.toLocaleTimeString('en-IN', { hour: 'numeric', minute: '2-digit', timeZone: TZ }).toUpperCase()
}
function left(d) {
  const mins = Math.round((d.due - now.value) / 60000)
  if (mins < 60) return `${mins} min left`
  if (mins < 1440) return `${Math.floor(mins / 60)} h ${mins % 60} min left`
  return `${rel(dayDiff(d.date)).toLowerCase()}, ${clock(d)}`
}

const prefersDark = () => window.matchMedia('(prefers-color-scheme: dark)').matches
const saved = document.documentElement.dataset.theme
const isDark = ref(saved ? saved === 'dark' : prefersDark())
function toggleTheme() {
  isDark.value = !isDark.value
  const t = isDark.value ? 'dark' : 'light'
  document.documentElement.dataset.theme = t
  try { localStorage.setItem('theme', t) } catch (e) {}
}
</script>

<template>
  <div class="shell">
    <header class="top">
      <a class="brand" href="/"><span class="logo"><svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-hourglass-split" viewBox="0 0 16 16">
  <path d="M2.5 15a.5.5 0 1 1 0-1h1v-1a4.5 4.5 0 0 1 2.557-4.06c.29-.139.443-.377.443-.59v-.7c0-.213-.154-.451-.443-.59A4.5 4.5 0 0 1 3.5 3V2h-1a.5.5 0 0 1 0-1h11a.5.5 0 0 1 0 1h-1v1a4.5 4.5 0 0 1-2.557 4.06c-.29.139-.443.377-.443.59v.7c0 .213.154.451.443.59A4.5 4.5 0 0 1 12.5 13v1h1a.5.5 0 0 1 0 1zm2-13v1c0 .537.12 1.045.337 1.5h6.326c.216-.455.337-.963.337-1.5V2zm3 6.35c0 .701-.478 1.236-1.011 1.492A3.5 3.5 0 0 0 4.5 13s.866-1.299 3-1.48zm1 0v3.17c2.134.181 3 1.48 3 1.48a3.5 3.5 0 0 0-1.989-3.158C8.978 9.586 8.5 9.052 8.5 8.351z"/>
</svg></span>Dueline</a>
      <div class="top-right">
        <span class="tz">All times IST</span>
        <button
          type="button"
          class="icon-btn"
          :aria-label="isDark ? 'Switch to light mode' : 'Switch to dark mode'"
          @click="toggleTheme"
        >{{ isDark ? '☀' : '☾' }}</button>
      </div>
    </header>

    <div class="layout">
      <main>
        <p class="kicker">Tools in Data Science · IIT Madras BS</p>
        <h1>Never miss a TDS deadline again</h1>

        <div class="pills">
          <button
            v-for="[key, text] in FILTERS"
            :key="key"
            type="button"
            class="pill"
            :aria-pressed="filter === key"
            @click="filter = key"
          >{{ text }}</button>
        </div>

        <p v-if="error" class="error">{{ error }}</p>

        <template v-if="loaded && !error">
          <p v-if="!upcoming.length" class="empty">Nothing due here right now.</p>

          <section v-for="day in upcoming" :key="day.date" class="day">
            <h3 class="day-head">
              {{ day.label }}
              <span class="chip" :class="{ soon: day.diff <= 1 }">{{ rel(day.diff) }}</span>
            </h3>
            <div class="cards">
              <article v-for="(r, i) in day.rows" :key="i" class="card" :class="'k-' + r.kind">
                <div class="card-top">
                  <span class="kind">{{ KINDS[r.kind] || r.kind }}</span>
                  <time class="time" :datetime="r.date + 'T' + r.time">{{ clock(r) }}</time>
                </div>
                <h4 class="card-title">
                  <a v-if="r.url" :href="r.url" target="_blank" rel="noopener">{{ r.title }}</a>
                  <template v-else>{{ r.title }}</template>
                </h4>
                <p v-if="r.part" class="card-part">{{ r.part }}</p>
                <p v-if="r.marks" class="card-marks">{{ r.marks }} {{ r.marks == 1 ? 'mark' : 'marks' }}</p>
              </article>
            </div>
          </section>

          <details v-if="past.length" class="past">
            <summary>Earlier ({{ pastCount }})</summary>
            <section v-for="day in past" :key="day.date" class="day">
              <h3 class="day-head">
                {{ day.label }}
                <span class="chip">{{ rel(day.diff) }}</span>
              </h3>
              <div class="cards">
                <article v-for="(r, i) in day.rows" :key="i" class="card" :class="'k-' + r.kind">
                  <div class="card-top">
                    <span class="kind">{{ KINDS[r.kind] || r.kind }}</span>
                    <time class="time" :datetime="r.date + 'T' + r.time">{{ clock(r) }}</time>
                  </div>
                  <h4 class="card-title">
                    <a v-if="r.url" :href="r.url" target="_blank" rel="noopener">{{ r.title }}</a>
                    <template v-else>{{ r.title }}</template>
                  </h4>
                  <p v-if="r.part" class="card-part">{{ r.part }}</p>
                  <p v-if="r.marks" class="card-marks">{{ r.marks }} {{ r.marks == 1 ? 'mark' : 'marks' }}</p>
                </article>
              </div>
            </section>
          </details>
        </template>
      </main>

      <aside v-if="next" class="panel" aria-label="Next deadlines">
        <h2>Next up</h2>
        <p class="count">{{ upcomingCount }} {{ upcomingCount === 1 ? 'deadline' : 'deadlines' }} coming up</p>
        <div v-for="(d, i) in nextItems" :key="i" class="mini" :class="'k-' + d.kind">
          <div class="mini-title">{{ d.title }}</div>
          <div v-if="d.part" class="mini-part">{{ d.part }}</div>
          <div class="mini-left">{{ left(d) }}</div>
        </div>
      </aside>
    </div>

    <footer>
      <p class="made">Made by <a href="https://hi-sarang.vercel.app/" target="_blank" rel="noopener">Sarang</a></p>
      <p>This is not an official IIT Madras or TDS website. Always confirm deadlines on the course pages.</p>
      <p>Found an error? Send a gchat to 24f2005600@ds.study.iitm.ac.in</p>
    </footer>
  </div>
</template>