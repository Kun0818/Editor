<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: String,
    default: '',
  },
  colors: {
    type: Array,
    default: () => [],
  },
  title: {
    type: String,
    default: 'Color',
  },
  clearLabel: {
    type: String,
    default: 'Clear color',
  },
  moreLabel: {
    type: String,
    default: 'More colors...',
  },
  showClear: {
    type: Boolean,
    default: true,
  },
  showMore: {
    type: Boolean,
    default: true,
  },
  closeOnSelect: {
    type: Boolean,
    default: true,
  },
  disabled: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['update:modelValue', 'select'])

const rootRef = ref(null)
const nativeColorInputRef = ref(null)
const isOpen = ref(false)

const normalizeColor = (value) => String(value || '').trim().toLowerCase()

const selectedColor = computed(() => normalizeColor(props.modelValue))

const normalizedColors = computed(() =>
  props.colors
    .map((item) => {
      if (typeof item === 'string') {
        const value = normalizeColor(item)
        return value ? { label: value, value } : null
      }

      if (item && typeof item === 'object') {
        const value = normalizeColor(item.value)
        if (!value) {
          return null
        }
        return {
          label: item.label || value,
          value,
        }
      }

      return null
    })
    .filter(Boolean),
)

const closePanel = () => {
  isOpen.value = false
}

const openPanel = () => {
  if (props.disabled) {
    return
  }
  isOpen.value = true
}

const togglePanel = () => {
  if (props.disabled) {
    return
  }
  if (isOpen.value) {
    closePanel()
  } else {
    openPanel()
  }
}

const emitSelection = (value) => {
  const normalized = normalizeColor(value)
  emit('update:modelValue', normalized)
  emit('select', normalized)
}

const onSelectColor = (value) => {
  if (props.disabled) {
    return
  }
  emitSelection(value)
  if (props.closeOnSelect) {
    closePanel()
  }
}

const onClearColor = () => {
  if (props.disabled) {
    return
  }
  emitSelection('')
  if (props.closeOnSelect) {
    closePanel()
  }
}

const isActiveColor = (value) =>
  normalizeColor(value) === selectedColor.value && !!selectedColor.value

const openNativeColorPicker = () => {
  if (props.disabled) {
    return
  }
  nativeColorInputRef.value?.click()
}

const onNativeColorChange = (event) => {
  const value = event.target?.value || ''
  if (value) {
    onSelectColor(value)
  }
  if (event.target) {
    event.target.value = ''
  }
}

const onDocumentPointerDown = (event) => {
  if (!isOpen.value) {
    return
  }
  if (!(event.target instanceof Node)) {
    return
  }
  if (rootRef.value?.contains(event.target)) {
    return
  }
  closePanel()
}

const onDocumentKeydown = (event) => {
  if (!isOpen.value) {
    return
  }
  if (event.key === 'Escape') {
    event.preventDefault()
    closePanel()
  }
}

watch(isOpen, (open) => {
  if (open) {
    document.addEventListener('pointerdown', onDocumentPointerDown)
    document.addEventListener('keydown', onDocumentKeydown)
  } else {
    document.removeEventListener('pointerdown', onDocumentPointerDown)
    document.removeEventListener('keydown', onDocumentKeydown)
  }
})

watch(
  () => props.disabled,
  (disabled) => {
    if (disabled) {
      closePanel()
    }
  },
)

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', onDocumentPointerDown)
  document.removeEventListener('keydown', onDocumentKeydown)
})
</script>

<template>
  <div ref="rootRef" class="color-picker-popover">
    <button
      type="button"
      class="color-picker-trigger"
      :class="{ open: isOpen, disabled: props.disabled }"
      :disabled="props.disabled"
      @mousedown.prevent
      @click.stop="togglePanel"
    >
      <slot name="trigger" :open="isOpen" :selected-color="selectedColor">
        <span class="fallback-trigger">
          <span class="fallback-swatch" :style="{ backgroundColor: selectedColor || '#ffffff' }"></span>
          <span class="fallback-label">{{ title }}</span>
        </span>
      </slot>
    </button>

    <div v-if="isOpen" class="color-picker-panel" role="dialog" :aria-label="title" @mousedown.stop>
      <header class="color-panel-title">{{ title }}</header>

      <div class="color-swatch-grid" role="listbox">
        <button
          v-for="color in normalizedColors"
          :key="color.value"
          type="button"
          class="color-swatch"
          :class="{ active: isActiveColor(color.value) }"
          :title="color.label"
          :aria-label="color.label"
          :aria-selected="isActiveColor(color.value)"
          @mousedown.prevent
          @click="onSelectColor(color.value)"
        >
          <span class="color-swatch-fill" :style="{ backgroundColor: color.value }"></span>
        </button>
      </div>

      <div class="color-panel-actions">
        <button
          v-if="showClear"
          type="button"
          class="color-action-btn"
          @mousedown.prevent
          @click="onClearColor"
        >
          {{ clearLabel }}
        </button>
        <button
          v-if="showMore"
          type="button"
          class="color-action-btn ghost"
          @mousedown.prevent
          @click="openNativeColorPicker"
        >
          {{ moreLabel }}
        </button>
        <input ref="nativeColorInputRef" class="native-color-input" type="color" @change="onNativeColorChange" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.color-picker-popover {
  position: relative;
  display: inline-flex;
}

.color-picker-trigger {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 2.7rem;
  height: 2.2rem;
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.6rem;
  padding: 0 0.45rem;
  cursor: pointer;
  transition: transform 140ms ease, box-shadow 140ms ease, border-color 140ms ease;
}

.color-picker-trigger:hover,
.color-picker-trigger.open {
  border-color: var(--editor-brand);
  box-shadow: 0 8px 24px rgba(15, 118, 110, 0.14);
}

.color-picker-trigger.disabled {
  opacity: 0.56;
  cursor: not-allowed;
  box-shadow: none;
  transform: none;
}

.fallback-trigger {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
}

.fallback-swatch {
  width: 0.88rem;
  height: 0.88rem;
  border: 1px solid rgba(15, 23, 42, 0.22);
  border-radius: 0.2rem;
}

.fallback-label {
  font-size: 0.8rem;
  font-weight: 600;
}

.color-picker-panel {
  position: absolute;
  top: calc(100% + 0.45rem);
  left: 0;
  z-index: 70;
  width: 238px;
  border: 1px solid var(--editor-border);
  border-radius: 0.72rem;
  background: #ffffff;
  box-shadow: 0 16px 36px rgba(15, 23, 42, 0.2);
  padding: 0.62rem;
}

.color-panel-title {
  margin-bottom: 0.5rem;
  font-size: 0.76rem;
  font-weight: 700;
  color: var(--editor-muted);
  letter-spacing: 0.04em;
}

.color-swatch-grid {
  display: grid;
  grid-template-columns: repeat(8, minmax(0, 1fr));
  gap: 0.35rem;
}

.color-swatch {
  border: none;
  background: transparent;
  padding: 0;
  cursor: pointer;
}

.color-swatch-fill {
  display: block;
  width: 1.26rem;
  height: 1.26rem;
  border: 1px solid rgba(15, 23, 42, 0.18);
  border-radius: 0.32rem;
  transition: transform 120ms ease, box-shadow 120ms ease;
}

.color-swatch:hover .color-swatch-fill {
  transform: translateY(-1px);
  box-shadow: 0 3px 10px rgba(15, 23, 42, 0.2);
}

.color-swatch.active .color-swatch-fill {
  box-shadow: 0 0 0 2px #0f766e;
}

.color-panel-actions {
  margin-top: 0.6rem;
  display: flex;
  gap: 0.45rem;
}

.color-action-btn {
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.5rem;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.3rem 0.55rem;
  cursor: pointer;
}

.color-action-btn:hover {
  border-color: #0f766e;
}

.color-action-btn.ghost {
  background: #f8fafc;
}

.native-color-input {
  position: absolute;
  opacity: 0;
  pointer-events: none;
}
</style>
