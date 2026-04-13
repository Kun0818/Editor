<script setup>
import {
  Bold,
  Download,
  Eraser,
  Grid3x3,
  Heading1,
  Heading2,
  Heading3,
  Heading4,
  Heading5,
  Image,
  Italic,
  List,
  ListOrdered,
  Underline,
} from '@lucide/vue'

const props = defineProps({
  activeTools: {
    type: Array,
    default: () => [],
  },
  currentFontSize: {
    type: String,
    default: '12',
  },
  fontSizeOptions: {
    type: Array,
    default: () => [],
  },
})

const emit = defineEmits(['action', 'font-size-change'])

const tools = [
  // { key: 'heading1', label: 'Heading 1', title: 'Format as heading 1', icon: Heading1 },
  // { key: 'heading2', label: 'Heading 2', title: 'Format as heading 2', icon: Heading2 },
  // { key: 'heading3', label: 'Heading 3', title: 'Format as heading 3', icon: Heading3 },
  // { key: 'heading4', label: 'Heading 4', title: 'Format as heading 4', icon: Heading4 },
  // { key: 'heading5', label: 'Heading 5', title: 'Format as heading 5', icon: Heading5 },
  { key: 'bold', label: 'Bold', title: 'Bold text', icon: Bold },
  { key: 'italic', label: 'Italic', title: 'Italic text', icon: Italic },
  { key: 'underline', label: 'Underline', title: 'Underline text', icon: Underline },
  { key: 'list', label: 'List', title: 'Bullet list', icon: List },
  { key: 'orderedList', label: 'Ordered List', title: 'Numbered list', icon: ListOrdered },
  { key: 'image', label: 'Image', title: 'Insert or edit image', icon: Image },
  { key: 'table', label: 'Table', title: 'Insert a 3x3 table', icon: Grid3x3 },
]

const utilityTools = [
  { key: 'export', label: 'Export HTML', title: 'Export HTML', icon: Download },
  { key: 'clear', label: 'Clear', title: 'Clear editor', icon: Eraser },
]
</script>

<template>
  <div class="toolbar" role="toolbar" aria-label="Editor formatting">



    <div class="tools">
      <label class="size-control" aria-label="Font size">
        <select class="size-select" :value="props.currentFontSize" title="Font size" @mousedown.stop
          @change="emit('font-size-change', $event.target.value)">
          <option v-for="size in props.fontSizeOptions" :key="size" :value="String(size)">
            {{ size }}px
          </option>
        </select>
      </label>
      <button v-for="tool in tools" :key="tool.key" type="button" class="tool"
        :class="{ active: props.activeTools.includes(tool.key) }" :title="tool.title" :aria-label="tool.label"
        @mousedown.prevent @click="emit('action', tool.key)">
        <component :is="tool.icon" class="tool-icon" :size="16" :stroke-width="1.9" aria-hidden="true" />
      </button>
    </div>

    <div class="utility-tools">
      <button v-for="tool in utilityTools" :key="tool.key" type="button" class="tool utility" :title="tool.title"
        :aria-label="tool.label" @mousedown.prevent @click="emit('action', tool.key)">
        <component :is="tool.icon" class="tool-icon" :size="16" :stroke-width="1.9" aria-hidden="true" />
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
  /* border: 1px solid var(--editor-border); */
  /* border-radius: 0.9rem; */
  border-top-left-radius: 0.5rem;
  border-top-right-radius: 0.5rem;
  background: #ffffff;
}

.size-control {
  display: inline-flex;
  align-items: center;
}

.size-select {
  min-width: 6.8rem;
  height: 2.2rem;
  border: 1px solid var(--editor-border);
  border-radius: 0.6rem;
  background: #ffffff;
  color: var(--editor-ink);
  font-size: 0.85rem;
  font-weight: 600;
  padding: 0 0.55rem;
  outline: none;
}

.size-select:focus-visible {
  border-color: #0f766e;
  box-shadow: 0 0 0 2px rgba(15, 118, 110, 0.16);
}

.tools,
.utility-tools {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.tool {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 2.2rem;
  height: 2.2rem;
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.6rem;
  padding: 0;
  cursor: pointer;
  transition: transform 140ms ease, box-shadow 140ms ease, border-color 140ms ease;
}

.tool-icon {
  flex-shrink: 0;
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
