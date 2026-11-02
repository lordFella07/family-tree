<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'
import TreeNode from './components/TreeNode.vue'

/* ===== 1. RETRIEVE the JSON data ===== */
const title = ref('Family Tree')
const people = ref([])
const loading = ref(true)
const error = ref('')

onMounted(async () => {
  try {
    const res = await fetch(import.meta.env.BASE_URL + 'data/people.json')
    if (!res.ok) throw new Error('HTTP ' + res.status)
    const data = await res.json()
    title.value = data.title || 'Family Tree'
    people.value = data.people
    console.log(`Loaded ${data.people.length} people from people.json`, data)
  } catch (e) {
    error.value = 'Could not load people.json: ' + e.message
  } finally {
    loading.value = false
  }
})

/* ===== 2. PROCESS: relationships between people ===== */
const byId = computed(() => new Map(people.value.map(p => [p.id, p])))

function parentsOf(p) {
  return [byId.value.get(p.fatherId), byId.value.get(p.motherId)].filter(Boolean)
}
function childrenOf(p) {
  return people.value.filter(c => c.fatherId === p.id || c.motherId === p.id)
}
// Spouses are read in both directions, so one-sided data still works
function spousesOf(p) {
  const ids = new Set((p.spouses || []).map(s => s.id))
  people.value.forEach(o => {
    if ((o.spouses || []).some(s => s.id === p.id)) ids.add(o.id)
  })
  return [...ids].map(id => byId.value.get(id)).filter(Boolean)
}
function siblingsOf(p) {
  return people.value.filter(o => o.id !== p.id &&
    ((p.fatherId && o.fatherId === p.fatherId) || (p.motherId && o.motherId === p.motherId)))
}

// Builds the tree: a node = one person + spouses, plus the children of that couple
function buildNode(person) {
  const members = [person, ...spousesOf(person)]
  const ids = members.map(m => m.id)
  const kids = people.value
    .filter(c => ids.includes(c.fatherId) || ids.includes(c.motherId))
    .sort((a, b) => (a.dob || '').localeCompare(b.dob || ''))
  return { members, children: kids.map(buildNode) }
}

// Roots = people with no parents who did not marry into the family
const roots = computed(() => {
  const used = new Set()
  const out = []
  people.value
    .filter(p => !p.fatherId && !p.motherId && !p.isInLaw)
    .forEach(p => {
      if (used.has(p.id)) return
      const node = buildNode(p)
      node.members.forEach(m => used.add(m.id))
      out.push(node)
    })
  return out
})

// Generation number (0 = oldest). In-laws take their spouse's generation.
const genOf = computed(() => {
  const cache = new Map()
  function gen(p) {
    if (cache.has(p.id)) return cache.get(p.id)
    const parents = parentsOf(p)
    let g
    if (parents.length) {
      g = Math.max(...parents.map(gen)) + 1
    } else {
      const sp = spousesOf(p).find(s => parentsOf(s).length)
      g = sp ? gen(sp) : 0
    }
    cache.set(p.id, g)
    return g
  }
  people.value.forEach(gen)
  return cache
})
const generations = computed(() => [...new Set(genOf.value.values())].sort((a, b) => a - b))
const places = computed(() => [...new Set(people.value.map(p => p.address).filter(Boolean))].sort())

/* ===== 3. Date helpers ===== */
function ageOf(p) {
  if (!p.dob) return null
  const born = new Date(p.dob)
  const end = p.dateOfDeath ? new Date(p.dateOfDeath) : new Date()
  let age = end.getFullYear() - born.getFullYear()
  const m = end.getMonth() - born.getMonth()
  if (m < 0 || (m === 0 && end.getDate() < born.getDate())) age--
  return age
}
function formatDate(d) {
  return d ? new Date(d).toLocaleDateString('en-MY', { day: 'numeric', month: 'long', year: 'numeric', timeZone: 'UTC' }) : '—'
}
function initials(name) {
  return name.split(' ').map(w => w[0]).slice(0, 2).join('').toUpperCase()
}

const stats = computed(() => {
  const total = people.value.length
  const living = people.value.filter(p => !p.dateOfDeath).length
  return { total, living, gone: total - living, gens: generations.value.length }
})

const nextBirthday = computed(() => {
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  let best = null
  people.value.filter(p => p.dob && !p.dateOfDeath).forEach(p => {
    const b = new Date(p.dob)
    const next = new Date(today.getFullYear(), b.getUTCMonth(), b.getUTCDate())
    if (next < today) next.setFullYear(today.getFullYear() + 1)
    const days = Math.round((next - today) / 86400000)
    if (!best || days < best.days) {
      best = { name: p.name, days, turning: next.getFullYear() - b.getUTCFullYear() }
    }
  })
  return best
})

/* ===== 4. FILTERING ===== */
const q = ref('')
const fGender = ref('all')
const fGen = ref('all')
const fStatus = ref('all')
const fPlace = ref('all')

const matches = computed(() => {
  const text = q.value.toLowerCase().trim()
  return new Set(
    people.value.filter(p => {
      const okText = !text || [p.name, p.profession, p.address].join(' ').toLowerCase().includes(text)
      const okGender = fGender.value === 'all' || p.gender === fGender.value
      const okGen = fGen.value === 'all' || genOf.value.get(p.id) === Number(fGen.value)
      const okStatus = fStatus.value === 'all' || (fStatus.value === 'living' ? !p.dateOfDeath : !!p.dateOfDeath)
      const okPlace = fPlace.value === 'all' || p.address === fPlace.value
      return okText && okGender && okGen && okStatus && okPlace
    }).map(p => p.id)
  )
})
function resetFilters() {
  q.value = ''
  fGender.value = fGen.value = fStatus.value = fPlace.value = 'all'
}

/* ===== 5. NAVIGATION: select, jump to relatives, go back ===== */
const selectedId = ref(null)
const history = ref([])
const selected = computed(() => byId.value.get(selectedId.value) || null)

function select(id, remember = true) {
  if (remember && selectedId.value && selectedId.value !== id) history.value.push(selectedId.value)
  selectedId.value = id
  console.log('Selected:', byId.value.get(id)?.name)
  nextTick(() => {
    document.getElementById('person-' + id)?.scrollIntoView({ behavior: 'smooth', block: 'center', inline: 'center' })
  })
}
function goBack() {
  const prev = history.value.pop()
  if (prev) select(prev, false)
}

const relatives = computed(() => {
  const p = selected.value
  if (!p) return []
  return [
    { label: 'Parents', list: parentsOf(p) },
    { label: 'Spouse', list: spousesOf(p) },
    { label: 'Siblings', list: siblingsOf(p) },
    { label: 'Children', list: childrenOf(p) }
  ].filter(g => g.list.length)
})

/* ===== 6. EXTRA: RELATIONSHIP FINDER (breadth-first search) ===== */
const relA = ref('')
const relB = ref('')
const relResult = ref(null)
const sortedPeople = computed(() => [...people.value].sort((a, b) => a.name.localeCompare(b.name)))

// How the last person on the path relates to the first, by the steps taken
const LABELS = {
  'parent': ['father', 'mother', 'parent'],
  'child': ['son', 'daughter', 'child'],
  'spouse': ['husband', 'wife', 'spouse'],
  'parent,child': ['brother', 'sister', 'sibling'],
  'parent,parent': ['grandfather', 'grandmother', 'grandparent'],
  'child,child': ['grandson', 'granddaughter', 'grandchild'],
  'parent,parent,parent': ['great-grandfather', 'great-grandmother', 'great-grandparent'],
  'child,child,child': ['great-grandson', 'great-granddaughter', 'great-grandchild'],
  'parent,parent,child': ['uncle', 'aunt', 'aunt/uncle'],
  'parent,child,child': ['nephew', 'niece', 'nephew/niece'],
  'parent,parent,child,child': ['cousin', 'cousin', 'cousin'],
  'spouse,parent': ['father-in-law', 'mother-in-law', 'parent-in-law'],
  'child,spouse': ['son-in-law', 'daughter-in-law', 'child-in-law'],
  'parent,child,spouse': ['brother-in-law', 'sister-in-law', 'sibling-in-law'],
  'spouse,parent,child': ['brother-in-law', 'sister-in-law', 'sibling-in-law']
}

function neighbours(p) {
  return [
    ...parentsOf(p).map(x => ({ person: x, step: 'parent' })),
    ...childrenOf(p).map(x => ({ person: x, step: 'child' })),
    ...spousesOf(p).map(x => ({ person: x, step: 'spouse' }))
  ]
}
function findPath(aId, bId) {
  const queue = [{ id: aId, path: [] }]
  const seen = new Set([aId])
  while (queue.length) {
    const { id, path } = queue.shift()
    if (id === bId) return path
    for (const n of neighbours(byId.value.get(id))) {
      if (!seen.has(n.person.id)) {
        seen.add(n.person.id)
        queue.push({ id: n.person.id, path: [...path, { step: n.step, person: n.person }] })
      }
    }
  }
  return null
}
function findRelation() {
  if (!relA.value || !relB.value) return
  const a = byId.value.get(relA.value)
  const b = byId.value.get(relB.value)
  if (a.id === b.id) {
    relResult.value = { text: 'That is the same person.', chain: [] }
    return
  }
  const path = findPath(a.id, b.id)
  if (!path) {
    relResult.value = { text: 'No relationship found in the data.', chain: [] }
    return
  }
  const labels = LABELS[path.map(s => s.step).join(',')]
  const label = labels ? labels[b.gender === 'male' ? 0 : b.gender === 'female' ? 1 : 2] : null
  relResult.value = {
    text: label
      ? `${b.name} is ${a.name}'s ${label}.`
      : `${b.name} is a relative of ${a.name}, ${path.length} steps apart.`,
    chain: [a.name, ...path.map(s => s.person.name)]
  }
  console.log('Relationship:', relResult.value.text, relResult.value.chain)
}
</script>

<template>
  <div>
    <header class="hero">
      <p class="eyebrow">Salasilah • ITT588 Lab Work 4</p>
      <h1>{{ title }}</h1>
      <div class="stats">
        <div><b>{{ stats.total }}</b><span>members</span></div>
        <div><b>{{ stats.gens }}</b><span>generations</span></div>
        <div><b>{{ stats.living }}</b><span>living</span></div>
        <div><b>{{ stats.gone }}</b><span>passed away</span></div>
      </div>
      <p v-if="nextBirthday" class="birthday">
        🎂 Next birthday: <b>{{ nextBirthday.name }}</b>
        {{ nextBirthday.days === 0 ? 'is today!' : 'in ' + nextBirthday.days + ' days' }}
        (turning {{ nextBirthday.turning }})
      </p>
    </header>

    <!-- Filters -->
    <section class="controls">
      <input type="search" v-model="q" placeholder="Search name, profession or place...">
      <select v-model="fGender">
        <option value="all">All genders</option>
        <option value="male">Male</option>
        <option value="female">Female</option>
      </select>
      <select v-model="fGen">
        <option value="all">All generations</option>
        <option v-for="g in generations" :key="g" :value="g">Generation {{ g + 1 }}</option>
      </select>
      <select v-model="fStatus">
        <option value="all">Living and deceased</option>
        <option value="living">Living</option>
        <option value="deceased">Deceased</option>
      </select>
      <select v-model="fPlace">
        <option value="all">All places</option>
        <option v-for="pl in places" :key="pl" :value="pl">{{ pl }}</option>
      </select>
      <button class="btn" @click="resetFilters">Reset</button>
      <span class="count">{{ matches.size }} of {{ people.length }} match</span>
    </section>

    <p v-if="loading" class="msg">Loading family data...</p>
    <p v-else-if="error" class="msg error">{{ error }}</p>

    <main v-else class="layout">
      <!-- Tree -->
      <section class="treebox">
        <ul class="tree">
          <TreeNode
            v-for="r in roots"
            :key="r.members[0].id"
            :node="r"
            :selected-id="selectedId"
            :matches="matches"
            @select="select"
          />
        </ul>
      </section>

      <!-- Side panels -->
      <aside class="side">
        <div class="panel">
          <p v-if="!selected" class="hint">Click anyone in the tree to see their details.</p>
          <template v-else>
            <button class="link" :disabled="!history.length" @click="goBack">← Back</button>
            <div class="who">
              <span class="avatar big">
                <img v-if="selected.photo" :src="selected.photo" :alt="selected.name">
                <template v-else>{{ initials(selected.name) }}</template>
              </span>
              <div>
                <h3>{{ selected.name }}</h3>
                <small>{{ selected.gender }}{{ selected.isInLaw ? ' • married into the family' : '' }}</small>
              </div>
            </div>
            <dl>
              <dt>Born</dt><dd>{{ formatDate(selected.dob) }}</dd>
              <template v-if="selected.dateOfDeath">
                <dt>Died</dt><dd>{{ formatDate(selected.dateOfDeath) }}</dd>
              </template>
              <dt>Age</dt>
              <dd>{{ ageOf(selected) === null ? '—' : ageOf(selected) }}{{ selected.dateOfDeath ? ' (at death)' : '' }}</dd>
              <dt>Job</dt><dd>{{ selected.profession || '—' }}</dd>
              <dt>Place</dt><dd>{{ selected.address || '—' }}</dd>
              <dt>Blood</dt><dd>{{ selected.bloodGroup || '—' }}</dd>
            </dl>
            <div v-for="g in relatives" :key="g.label" class="group">
              <h4>{{ g.label }}</h4>
              <button v-for="r in g.list" :key="r.id" class="chip" @click="select(r.id)">{{ r.name }}</button>
            </div>
          </template>
        </div>

        <div class="panel finder">
          <h3>How are they related?</h3>
          <select v-model="relA">
            <option value="" disabled>Person A</option>
            <option v-for="p in sortedPeople" :key="p.id" :value="p.id">{{ p.name }}</option>
          </select>
          <select v-model="relB">
            <option value="" disabled>Person B</option>
            <option v-for="p in sortedPeople" :key="p.id" :value="p.id">{{ p.name }}</option>
          </select>
          <button class="btn" @click="findRelation">Find relationship</button>
          <div v-if="relResult" class="result">
            <b>{{ relResult.text }}</b>
            <p v-if="relResult.chain.length" class="chain">{{ relResult.chain.join(' → ') }}</p>
          </div>
        </div>
      </aside>
    </main>
  </div>
</template>

<style>
:root {
  --forest: #2f4a3a; --forest2: #3f6150; --cream: #f7f1e3; --paper: #fffdf7;
  --clay: #c8553d; --gold: #d9a441; --ink: #25302a; --line: #b9a97e;
}
* { box-sizing: border-box; }
body { display: block; margin: 0; font-family: 'Segoe UI', system-ui, sans-serif; background: var(--cream); color: var(--ink); }
#app { display: block; max-width: none; margin: 0; padding: 0; }

/* Header */
.hero { background: linear-gradient(135deg, var(--forest), var(--forest2)); color: #fff; text-align: center; padding: 40px 16px 28px; }
.eyebrow { margin: 0; letter-spacing: 2px; text-transform: uppercase; color: var(--gold); font-size: 0.8rem; }
.hero h1 { font-family: Georgia, 'Times New Roman', serif; font-size: clamp(1.8rem, 5vw, 2.8rem); margin: 0.2em 0 0.6em; }
.stats { display: flex; justify-content: center; gap: 28px; flex-wrap: wrap; }
.stats div { display: flex; flex-direction: column; }
.stats b { font-size: 1.6rem; color: var(--gold); }
.stats span { font-size: 0.8rem; opacity: 0.85; }
.birthday { margin: 16px 0 0; font-size: 0.95rem; }

/* Filters */
.controls { display: flex; flex-wrap: wrap; gap: 10px; align-items: center; padding: 16px; max-width: 1300px; margin: 0 auto; }
.controls input, .controls select, .finder select {
  padding: 10px 12px; border: 2px solid #d9cfae; border-radius: 10px; font: inherit; background: #fff;
}
.controls input { flex: 1 1 240px; }
.btn { border: 0; border-radius: 999px; padding: 10px 22px; background: var(--clay); color: #fff; font: inherit; font-weight: 600; cursor: pointer; }
.btn:hover { filter: brightness(1.1); }
.count { margin-left: auto; color: #5c6657; font-size: 0.9rem; }
.msg { text-align: center; padding: 30px; }
.error { color: var(--clay); font-weight: 600; }

/* Layout */
.layout { display: grid; grid-template-columns: 1fr 340px; gap: 20px; padding: 0 16px 40px; max-width: 1300px; margin: 0 auto; align-items: start; }
.treebox { background: var(--paper); border-radius: 18px; padding: 28px 16px; overflow: auto; box-shadow: 0 6px 20px rgba(47, 74, 58, 0.1); }
.side { display: flex; flex-direction: column; gap: 16px; position: sticky; top: 12px; }
@media (max-width: 900px) {
  .layout { grid-template-columns: 1fr; }
  .side { position: static; }
  .count { margin-left: 0; }
}

/* Panels */
.panel { background: var(--paper); border-radius: 18px; padding: 18px; box-shadow: 0 6px 20px rgba(47, 74, 58, 0.1); }
.panel h3 { margin: 0; font-family: Georgia, serif; color: var(--forest); }
.hint { margin: 0; color: #6a6f62; }
.who { display: flex; gap: 12px; align-items: center; margin: 8px 0; }
.who small { color: #6a6f62; text-transform: capitalize; }
.avatar.big { width: 56px; height: 56px; flex: none; font-size: 1.1rem; }
dl { display: grid; grid-template-columns: auto 1fr; gap: 4px 14px; margin: 12px 0; font-size: 0.9rem; }
dt { color: #6a6f62; }
dd { margin: 0; }
.group h4 { margin: 12px 0 6px; font-size: 0.8rem; text-transform: uppercase; letter-spacing: 1px; color: var(--forest); }
.chip { border: 0; background: var(--forest); color: #fff; padding: 5px 12px; border-radius: 999px; cursor: pointer; margin: 0 6px 6px 0; font: inherit; font-size: 0.82rem; }
.chip:hover { background: var(--clay); }
.link { background: none; border: 0; color: var(--forest); cursor: pointer; font: inherit; padding: 0; margin-bottom: 6px; }
.link:disabled { opacity: 0.35; cursor: default; }
.finder select { width: 100%; margin: 8px 0 0; }
.finder .btn { margin-top: 10px; }
.result { margin-top: 12px; background: #fff3d6; border-left: 5px solid var(--gold); padding: 10px 12px; border-radius: 8px; }
.chain { margin: 6px 0 0; font-size: 0.8rem; color: #5c6657; }
</style>