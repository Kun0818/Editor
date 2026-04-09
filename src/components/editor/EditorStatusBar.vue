<script setup>
const props = defineProps({
  words: {
    type: Number,
    required: true,
  },
  characters: {
    type: Number,
    required: true,
  },
  lines: {
    type: Number,
    required: true,
  },
  lastSavedLabel: {
    type: String,
    default: 'Not saved yet',
  },
})

const items = [
  { key: 'words', label: 'Words' },
  { key: 'characters', label: 'Chars' },
  { key: 'lines', label: 'Lines' },
]
</script>

<template>
  <footer class="status" aria-label="Editor status">
    <div class="metrics">
      <p v-for="item in items" :key="item.key" class="metric">
        <span class="metric-label">{{ item.label }}</span>
        <strong class="metric-value">{{ props[item.key] }}</strong>
      </p>
    </div>
    <p class="saved-time">{{ lastSavedLabel }}</p>
  </footer>
</template>

<style scoped>
.status {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 0.72rem 0.95rem;
  border: 1px solid var(--editor-border);
  border-radius: 0.8rem;
  background: linear-gradient(120deg, #ffffff, #f4fbf9);
}

.metrics {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.metric {
  margin: 0;
  display: flex;
  align-items: center;
  gap: 0.34rem;
  color: var(--editor-muted);
}

.metric-label {
  font-size: 0.76rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.metric-value {
  font-size: 0.9rem;
  color: var(--editor-ink);
}

.saved-time {
  margin: 0;
  color: var(--editor-muted);
  font-size: 0.82rem;
  text-align: right;
}

@media (max-width: 720px) {
  .status {
    flex-direction: column;
    align-items: flex-start;
  }

  .saved-time {
    text-align: left;
  }
}
</style>
