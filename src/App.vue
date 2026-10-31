<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const TZ = 'Asia/Kolkata'
const items = ref([])
const error = ref('')
const loaded = ref(false)
const now = ref(new Date())
let timer

const KINDS = { assignment: 'Graded assignment', project: 'Take-home project' }

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
  // deadlines.xml: written by hand (phases, partial deadlines).
  // exam-ends.xml: generated daily from the exam pages (final deadlines).
  const [manual, auto] = await Promise.allSettled([load('/deadlines.xml'), load('/exam-ends.xml')])
  if (manual.status === 'rejected') error.value = 'Could not read deadlines.xml. Check the file for typos.'
  const seen = new Set()
  items.value = [...(manual.value || []), ...(auto.value || [])]
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

const upcoming = computed(() => group(items.value.filter((d) => d.due >= now.value)))
const past = computed(() => group(items.value.filter((d) => d.due < now.value)).reverse())

// All deadlines that close at the next closing time (can be more than one).
const nextItems = computed(() => {
  const first = items.value.find((d) => d.due >= now.value)
  return first ? items.value.filter((d) => +d.due === +first.due) : []
})
const next = computed(() => nextItems.value[0])
const label = (d) => `${d.title}${d.part ? ' · ' + d.part : ''}`

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

const theme = ref(document.documentElement.dataset.theme || '')
function toggleTheme() {
  const dark = theme.value
    ? theme.value === 'dark'
    : window.matchMedia('(prefers-color-scheme: dark)').matches
  theme.value = dark ? 'light' : 'dark'
  document.documentElement.dataset.theme = theme.value
  try { localStorage.setItem('theme', theme.value) } catch (e) {}
}
</script>

<template>
  <div class="page">
    <div class="bar">
      <span>All times IST</span>
      <button type="button" @click="toggleTheme">Theme</button>
    </div>

    <header>
      <p class="kicker">Tools in Data Science · IIT Madras BS</p>
      <h1>Dueline</h1>
      <p class="lede">Never miss a TDS deadline again</p>
    </header>

    <main>
      <p v-if="error" class="error">{{ error }}</p>

      <section v-if="next" class="next" aria-label="Next deadline">
        <div class="label">Next up</div>
                <p v-for="(d, i) in nextItems" :key="i">
          <span class="next-title" :class="'kind-' + d.kind">{{ d.title }}</span>{{ d.part ? ' · ' + d.part : '' }}
        </p>
        <p class="when">{{ left(next) }}</p>
      </section>

      <template v-if="loaded && !error">
        <div class="section-label">Upcoming</div>
        <p v-if="!upcoming.length" class="empty">Nothing due. Add the next ones to deadlines.xml.</p>
        <section v-for="day in upcoming" :key="day.date" class="day">
          <h3>
            {{ day.label }}
            <span class="rel" :class="{ soon: day.diff <= 1 }">{{ rel(day.diff) }}</span>
          </h3>
          <ul class="items">
            <li v-for="(r, i) in day.rows" :key="i">
              <span class="what">
                <span class="kind" :class="'kind-' + r.kind">{{ KINDS[r.kind] || r.kind }}</span>
                <span class="title">{{ r.title }}</span>
                <span v-if="r.part" class="part"> · {{ r.part }}</span>
              </span>
              <time :datetime="r.date + 'T' + r.time">{{ clock(r) }}</time>
            </li>
          </ul>
        </section>

        <details v-if="past.length">
          <summary>Earlier ({{ past.reduce((n, d) => n + d.rows.length, 0) }})</summary>
          <section v-for="day in past" :key="day.date" class="day">
            <h3>
              {{ day.label }}
              <span class="rel">{{ rel(day.diff) }}</span>
            </h3>
            <ul class="items">
              <li v-for="(r, i) in day.rows" :key="i">
                <span class="what">
                  <span class="kind">{{ KINDS[r.kind] || r.kind }}</span>
                  <span class="title">{{ r.title }}</span>
                  <span v-if="r.part" class="part"> · {{ r.part }}</span>
                </span>
                <time :datetime="r.date + 'T' + r.time">{{ clock(r) }}</time>
              </li>
            </ul>
          </section>
        </details>
      </template>
    </main>

            <footer>
      <p class="made">Made by <a href="https://hi-sarang.vercel.app/" target="_blank" rel="noopener">Sarang</a></p>
      <p>This is not an official IIT Madras or TDS website. Always confirm deadlines on the course pages.</p>
      <p>Found an error? Send a gchat to 24f2005600@ds.study.iitm.ac.in</p>
    </footer>
  </div>
</template>