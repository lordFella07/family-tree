<script setup>
// Recursive component: draws one couple (or one person) and then calls itself for their children.
const props = defineProps({
  node: Object,       // { members: [person, ...], children: [node, ...] }
  selectedId: String, // the person currently selected
  matches: Set        // ids that pass the current filters
})
const emit = defineEmits(['select'])

function initials(name) {
  return name.split(' ').map(w => w[0]).slice(0, 2).join('').toUpperCase()
}
function years(p) {
  const born = p.dob ? p.dob.slice(0, 4) : '?'
  return p.dateOfDeath ? `${born} – ${p.dateOfDeath.slice(0, 4)}` : `b. ${born}`
}
</script>

<template>
  <li>
    <div class="unit">
      <template v-for="(p, i) in node.members" :key="p.id">
        <span v-if="i > 0" class="heart">♥</span>
        <button
          :id="'person-' + p.id"
          class="person"
          :class="[p.gender, { selected: p.id === selectedId, dim: !matches.has(p.id), gone: p.dateOfDeath }]"
          @click="emit('select', p.id)"
        >
          <span class="avatar">
            <img v-if="p.photo" :src="p.photo" :alt="p.name">
            <template v-else>{{ initials(p.name) }}</template>
          </span>
          <strong>{{ p.name }}</strong>
          <small>{{ years(p) }}</small>
        </button>
      </template>
    </div>

    <!-- Children: the component calls itself here -->
    <ul v-if="node.children.length">
      <TreeNode
        v-for="c in node.children"
        :key="c.members[0].id"
        :node="c"
        :selected-id="selectedId"
        :matches="matches"
        @select="emit('select', $event)"
      />
    </ul>
  </li>
</template>

<style>
/* ===== Tree connectors (pure CSS) ===== */
.tree, .tree ul { list-style: none; margin: 0; padding: 0; }
.tree { display: flex; justify-content: center; width: max-content; min-width: 100%; }
.tree ul { display: flex; justify-content: center; padding-top: 28px; position: relative; }
.tree li { position: relative; padding: 28px 10px 0; text-align: center; }
.tree > li { padding-top: 0; }
.tree > li::before, .tree > li::after { display: none; }

.tree li::before, .tree li::after {
  content: ''; position: absolute; top: 0; right: 50%;
  width: 50%; height: 28px; border-top: 2px solid var(--line);
}
.tree li::after { right: auto; left: 50%; border-left: 2px solid var(--line); }
.tree li:only-child::before, .tree li:only-child::after { display: none; }
.tree li:only-child { padding-top: 0; }
.tree li:first-child::before, .tree li:last-child::after { border: 0 none; }
.tree li:last-child::before { border-right: 2px solid var(--line); border-radius: 0 10px 0 0; }
.tree li:first-child::after { border-radius: 10px 0 0 0; }
.tree ul::before {
  content: ''; position: absolute; top: 0; left: 50%;
  height: 28px; border-left: 2px solid var(--line);
}

/* ===== Person cards ===== */
.unit { display: inline-flex; align-items: center; gap: 6px; }
.heart { color: var(--clay); font-size: 1.1rem; }
.person {
  width: 116px; padding: 10px 6px; border: 2px solid #e3d9bb; border-radius: 14px;
  background: var(--paper); cursor: pointer; font: inherit; color: inherit;
  display: flex; flex-direction: column; align-items: center; gap: 3px;
  transition: transform 0.15s, box-shadow 0.15s, opacity 0.2s;
}
.person:hover { transform: translateY(-3px); box-shadow: 0 8px 18px rgba(47, 74, 58, 0.18); }
.person.male { border-color: #9db9d3; }
.person.female { border-color: #e0a3a8; }
.person.gone { background: #efe9d8; }
.person.selected { outline: 3px solid var(--gold); outline-offset: 2px; }
.person.dim { opacity: 0.25; }
.person strong { font-size: 0.82rem; line-height: 1.2; }
.person small { color: #6a6f62; font-size: 0.72rem; }
.avatar {
  width: 44px; height: 44px; border-radius: 50%; background: var(--forest); color: #fff;
  display: grid; place-items: center; font-weight: 700; overflow: hidden;
}
.avatar img { width: 100%; height: 100%; object-fit: cover; }
</style>