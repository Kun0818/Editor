<script setup>
const props = defineProps({
  activeTools: {
    type: Array,
    default: () => [],
  },
})

const emit = defineEmits(['action'])

const tools = [
  { key: 'heading1', label: 'H1', title: 'Format as heading 1' },
  { key: 'heading2', label: 'H2', title: 'Format as heading 2' },
  { key: 'heading3', label: 'H3', title: 'Format as heading 3' },
  { key: 'heading4', label: 'H4', title: 'Format as heading 4' },
  { key: 'heading5', label: 'H5', title: 'Format as heading 5' },
  { key: 'bold', label: 'Bold', title: 'Bold text' },
  { key: 'italic', label: 'Italic', title: 'Italic text' },
  { key: 'quote', label: 'Quote', title: 'Block quote' },
  { key: 'list', label: 'List', title: 'Bullet list' },
  { key: 'table', label: 'Table', title: 'Insert a 3x3 table' },
  { key: 'tableMerge', label: 'Merge', title: 'Merge selected table cells' },
  { key: 'tableUnmerge', label: 'Unmerge', title: 'Split current merged cell' },
  { key: 'tableAddRow', label: 'Add Row', title: 'Add a row below current row' },
  { key: 'tableAddCol', label: 'Add Col', title: 'Add a column to the right' },
  { key: 'tableDeleteRow', label: 'Del Row', title: 'Delete current table row' },
  { key: 'tableDeleteCol', label: 'Del Col', title: 'Delete current table column' },
  { key: 'code', label: 'Code', title: 'Inline code / code block' },
  { key: 'date', label: 'Date', title: 'Insert current date and time' },
]

const utilityTools = [
  { key: 'export', label: 'Export HTML' },
  { key: 'clear', label: 'Clear' },
]
</script>

<template>
  <div class="toolbar" role="toolbar" aria-label="Editor formatting">
    <div class="tools">
      <button
        v-for="tool in tools"
        :key="tool.key"
        type="button"
        class="tool"
        :class="{ active: props.activeTools.includes(tool.key) }"
        :title="tool.title"
        @mousedown.prevent
        @click="emit('action', tool.key)"
      >
        {{ tool.label }}
      </button>
    </div>

    <div class="utility-tools">
      <button
        v-for="tool in utilityTools"
        :key="tool.key"
        type="button"
        class="tool utility"
        @mousedown.prevent
        @click="emit('action', tool.key)"
      >
        {{ tool.label }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 0.8rem;
  padding: 0.8rem;
  border: 1px solid var(--editor-border);
  border-radius: 0.9rem;
  background: linear-gradient(160deg, #ffffff, #f7fbff 55%, #eefbf7);
}

.tools,
.utility-tools {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.tool {
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.65rem;
  font-size: 0.86rem;
  font-weight: 600;
  padding: 0.38rem 0.72rem;
  cursor: pointer;
  transition: transform 140ms ease, box-shadow 140ms ease, border-color 140ms ease;
}

.tool:hover {
  transform: translateY(-1px);
  border-color: var(--editor-brand);
  box-shadow: 0 8px 24px rgba(15, 118, 110, 0.14);
}

.tool.active {
  border-color: #0f766e;
  background: #ecfeff;
  color: #0f766e;
  box-shadow: inset 0 0 0 1px rgba(15, 118, 110, 0.25);
}

.tool:focus-visible {
  outline: 2px solid var(--editor-brand);
  outline-offset: 2px;
}

.utility {
  background: #f8fafc;
}

@media (max-width: 720px) {
  .toolbar {
    padding: 0.7rem;
  }
}
</style>
