<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import EditorStatusBar from './editor/EditorStatusBar.vue'
import EditorToolbar from './editor/EditorToolbar.vue'

const STORAGE_KEY = 'editer:draft:wysiwyg:v1'
const defaultTitle = 'Untitled Document'
const defaultContent = `<h1>Product Notes</h1>
<p>Use this editor to draft ideas, release notes, or content blocks.</p>
<p><strong>Tip:</strong> Select text and click toolbar buttons to apply style.</p>`

const mentionVariables = [
  { key: 'first_name', label: 'First Name' },
  { key: 'last_name', label: 'Last Name' },
  { key: 'full_name', label: 'Full Name' },
  { key: 'email', label: 'Email' },
  { key: 'company', label: 'Company' },
  { key: 'today', label: 'Today Date' },
  { key: 'account_id', label: 'Account ID' },
]

const allowedTags = new Set([
  'p',
  'div',
  'span',
  'h1',
  'h2',
  'h3',
  'h4',
  'h5',
  'h6',
  'ul',
  'ol',
  'li',
  'strong',
  'em',
  'b',
  'i',
  'u',
  'blockquote',
  'pre',
  'code',
  'table',
  'thead',
  'tbody',
  'tr',
  'th',
  'td',
  'a',
  'time',
  'br',
  'hr',
])

const allowedAttrs = {
  a: new Set(['href', 'title', 'target', 'rel']),
  time: new Set(['datetime']),
  th: new Set(['colspan', 'rowspan']),
  td: new Set(['colspan', 'rowspan']),
}

const title = ref(defaultTitle)
const contentHtml = ref(defaultContent)
const editor = ref(null)
const lastSavedAt = ref(null)
const showExport = ref(false)
const exportedHtml = ref('')
const showMentionMenu = ref(false)
const mentionQuery = ref('')
const mentionRange = ref(null)
const mentionPosition = ref({ top: 0, left: 0 })
const mentionActiveIndex = ref(0)
const activeToolKeys = ref([])

const extractText = (rawHtml) => {
  const temp = document.createElement('div')
  temp.innerHTML = rawHtml
  return (temp.innerText || '').replaceAll('\u00A0', ' ')
}

const escapeHtml = (value) =>
  value
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#39;')

const isSafeHref = (href) =>
  /^(https?:|mailto:|tel:|\/|#)/i.test(href.trim())

const sanitizeHtml = (rawHtml) => {
  const parser = new DOMParser()
  const doc = parser.parseFromString(rawHtml, 'text/html')

  for (const node of Array.from(doc.body.querySelectorAll('*'))) {
    const tag = node.tagName.toLowerCase()
    if (!allowedTags.has(tag)) {
      node.replaceWith(doc.createTextNode(node.textContent || ''))
      continue
    }

    const keepAttrs = allowedAttrs[tag] || new Set()
    for (const attr of Array.from(node.attributes)) {
      const name = attr.name.toLowerCase()
      if (!keepAttrs.has(name)) {
        node.removeAttribute(name)
      }
    }

    if (tag === 'a') {
      const href = node.getAttribute('href')
      if (!href || !isSafeHref(href)) {
        node.removeAttribute('href')
      }
      if (node.getAttribute('target') === '_blank') {
        node.setAttribute('rel', 'noopener noreferrer')
      }
    }
  }

  return doc.body.innerHTML
}

const cleanEditorHtml = (rawHtml) => {
  const safe = sanitizeHtml(rawHtml)
  const plain = extractText(safe).trim()
  return plain ? safe : ''
}

const syncEditorFromModel = () => {
  if (!editor.value) {
    return
  }
  if (editor.value.innerHTML !== contentHtml.value) {
    editor.value.innerHTML = contentHtml.value
  }
}

const syncModelFromEditor = () => {
  if (!editor.value) {
    return
  }
  contentHtml.value = cleanEditorHtml(editor.value.innerHTML)
}

const plainText = computed(() => extractText(contentHtml.value))
const words = computed(() => plainText.value.trim().split(/\s+/).filter(Boolean).length)
const characters = computed(() => plainText.value.length)
const lines = computed(() => Math.max(1, plainText.value.split('\n').length))
const lastSavedLabel = computed(() =>
  lastSavedAt.value
    ? `Auto-saved ${lastSavedAt.value.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}`
    : 'Not saved yet',
)
const filteredMentions = computed(() => {
  const query = mentionQuery.value.trim().toLowerCase()
  const list = mentionVariables.filter((item) => {
    if (!query) {
      return true
    }
    return (
      item.key.toLowerCase().includes(query) ||
      item.label.toLowerCase().includes(query)
    )
  })
  return list.slice(0, 8)
})
const mentionMenuStyle = computed(() => ({
  top: `${mentionPosition.value.top}px`,
  left: `${mentionPosition.value.left}px`,
}))

let saveTimer = null
const saveDraft = () => {
  const payload = {
    title: title.value,
    content: cleanEditorHtml(contentHtml.value),
    savedAt: Date.now(),
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(payload))
  lastSavedAt.value = new Date(payload.savedAt)
}

watch([title, contentHtml], () => {
  if (saveTimer) {
    clearTimeout(saveTimer)
  }
  saveTimer = setTimeout(saveDraft, 250)
})
watch(filteredMentions, (list) => {
  if (!showMentionMenu.value) {
    return
  }
  if (!list.length) {
    showMentionMenu.value = false
    return
  }
  if (mentionActiveIndex.value >= list.length) {
    mentionActiveIndex.value = 0
  }
})

onMounted(() => {
  const raw = localStorage.getItem(STORAGE_KEY)
  if (raw) {
    try {
      const draft = JSON.parse(raw)
      if (typeof draft.title === 'string' && draft.title.trim()) {
        title.value = draft.title
      }
      if (typeof draft.content === 'string') {
        contentHtml.value = cleanEditorHtml(draft.content)
      }
      if (typeof draft.savedAt === 'number') {
        lastSavedAt.value = new Date(draft.savedAt)
      }
    } catch (error) {
      console.error('Failed to restore draft', error)
    }
  }

  nextTick(() => {
    syncEditorFromModel()
    updateActiveTools()
  })

  document.addEventListener('pointerdown', onDocumentPointerDown)
  document.addEventListener('selectionchange', onSelectionChange)
})

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', onDocumentPointerDown)
  document.removeEventListener('selectionchange', onSelectionChange)
})

const focusEditor = () => {
  editor.value?.focus()
}

const closeMentionMenu = () => {
  showMentionMenu.value = false
  mentionQuery.value = ''
  mentionRange.value = null
  mentionActiveIndex.value = 0
}

const formatVariableToken = (key) => `{{${key}}}`

const isNodeInEditor = (node) => {
  if (!editor.value || !node) {
    return false
  }
  if (node.nodeType === Node.TEXT_NODE) {
    return editor.value.contains(node.parentNode)
  }
  return editor.value.contains(node)
}

const queryStateSafe = (command) => {
  try {
    return document.queryCommandState(command)
  } catch {
    return false
  }
}

const findAncestorByTag = (node, tagNames) => {
  if (!node || !editor.value) {
    return null
  }

  const tags = new Set(tagNames)
  let current = node.nodeType === Node.TEXT_NODE ? node.parentNode : node

  while (current && current !== editor.value) {
    if (
      current.nodeType === Node.ELEMENT_NODE &&
      tags.has(current.tagName)
    ) {
      return current
    }
    current = current.parentNode
  }

  return null
}

const updateActiveTools = () => {
  const selection = window.getSelection()
  if (!selection || !selection.rangeCount) {
    activeToolKeys.value = []
    return
  }

  const node = selection.anchorNode
  if (!isNodeInEditor(node)) {
    activeToolKeys.value = []
    return
  }

  const active = new Set()

  if (queryStateSafe('bold')) {
    active.add('bold')
  }
  if (queryStateSafe('italic')) {
    active.add('italic')
  }
  if (queryStateSafe('insertUnorderedList')) {
    active.add('list')
  }

  const headingNode = findAncestorByTag(node, ['H1', 'H2', 'H3', 'H4', 'H5'])
  if (headingNode) {
    active.add(`heading${headingNode.tagName.slice(1)}`)
  }

  if (findAncestorByTag(node, ['BLOCKQUOTE'])) {
    active.add('quote')
  }
  if (findAncestorByTag(node, ['PRE', 'CODE'])) {
    active.add('code')
  }
  if (findAncestorByTag(node, ['TABLE', 'THEAD', 'TBODY', 'TR', 'TH', 'TD'])) {
    active.add('table')
  }

  activeToolKeys.value = Array.from(active)
}

const updateMentionFromSelection = () => {
  const selection = window.getSelection()
  if (!selection || !selection.rangeCount || !selection.isCollapsed) {
    closeMentionMenu()
    return
  }

  const anchorNode = selection.anchorNode
  if (!isNodeInEditor(anchorNode) || anchorNode?.nodeType !== Node.TEXT_NODE) {
    closeMentionMenu()
    return
  }

  const offset = selection.anchorOffset
  const textBefore = anchorNode.textContent?.slice(0, offset) ?? ''
  const match = textBefore.match(/(^|\s)@([a-zA-Z0-9_]*)$/)

  if (!match) {
    closeMentionMenu()
    return
  }

  const query = match[2] ?? ''
  const rangeStart = offset - query.length - 1
  const range = document.createRange()
  range.setStart(anchorNode, rangeStart)
  range.setEnd(anchorNode, offset)
  const queryChanged = query !== mentionQuery.value

  mentionRange.value = range
  mentionQuery.value = query
  if (queryChanged || !showMentionMenu.value) {
    mentionActiveIndex.value = 0
  }

  const rect = range.getBoundingClientRect()
  const left = Math.max(8, Math.min(rect.left, window.innerWidth - 300))
  mentionPosition.value = {
    top: rect.bottom + 8,
    left,
  }

  showMentionMenu.value = filteredMentions.value.length > 0
}

const selectMention = (item) => {
  if (!mentionRange.value) {
    return
  }

  focusEditor()
  const selection = window.getSelection()
  if (!selection) {
    return
  }

  selection.removeAllRanges()
  selection.addRange(mentionRange.value)
  mentionRange.value.deleteContents()

  const token = `{{${item.key}}}`
  const tokenNode = document.createTextNode(`${token} `)
  mentionRange.value.insertNode(tokenNode)

  const caret = document.createRange()
  caret.setStartAfter(tokenNode)
  caret.collapse(true)
  selection.removeAllRanges()
  selection.addRange(caret)

  closeMentionMenu()
  syncModelFromEditor()
  updateActiveTools()
}

const onEditorInput = () => {
  syncModelFromEditor()
  updateMentionFromSelection()
  updateActiveTools()
}

const onEditorCaretChange = () => {
  updateMentionFromSelection()
  updateActiveTools()
}

const onEditorKeydown = (event) => {
  if (!showMentionMenu.value) {
    return
  }

  const count = filteredMentions.value.length
  if (!count) {
    closeMentionMenu()
    return
  }

  if (event.key === 'ArrowDown') {
    event.preventDefault()
    mentionActiveIndex.value = (mentionActiveIndex.value + 1) % count
    return
  }

  if (event.key === 'ArrowUp') {
    event.preventDefault()
    mentionActiveIndex.value = (mentionActiveIndex.value - 1 + count) % count
    return
  }

  if (event.key === 'Enter' || event.key === 'Tab') {
    event.preventDefault()
    selectMention(filteredMentions.value[mentionActiveIndex.value])
    return
  }

  if (event.key === 'Escape') {
    event.preventDefault()
    closeMentionMenu()
  }
}

const onDocumentPointerDown = (event) => {
  if (!showMentionMenu.value) {
    return
  }

  const target = event.target
  if (editor.value?.contains(target)) {
    return
  }

  if (target instanceof Element && target.closest('.mention-menu')) {
    return
  }

  closeMentionMenu()
}

const onSelectionChange = () => {
  updateActiveTools()
}

const runCommand = (command, value = null) => {
  focusEditor()
  document.execCommand(command, false, value)
  syncModelFromEditor()
}

const insertHtmlAtSelection = (html) => {
  focusEditor()
  document.execCommand('insertHTML', false, html)
  syncModelFromEditor()
}

const wrapSelectionAsCode = () => {
  const selection = window.getSelection()
  const selectedText = selection?.rangeCount ? selection.getRangeAt(0).toString() : ''
  if (!selectedText) {
    insertHtmlAtSelection('<code>code</code>')
    return
  }

  const escaped = escapeHtml(selectedText)
  if (selectedText.includes('\n')) {
    insertHtmlAtSelection(`<pre><code>${escaped}</code></pre>`)
    return
  }
  insertHtmlAtSelection(`<code>${escaped}</code>`)
}

const insertDefaultTable = (rows = 3, cols = 3) => {
  const headerCells = Array.from(
    { length: cols },
    (_, index) => `<th>Header ${index + 1}</th>`,
  ).join('')

  const bodyRows = Array.from({ length: Math.max(1, rows - 1) }, (_, rowIndex) => {
    const cells = Array.from(
      { length: cols },
      (_, colIndex) => `<td>Cell ${rowIndex + 1}-${colIndex + 1}</td>`,
    ).join('')
    return `<tr>${cells}</tr>`
  }).join('')

  const tableHtml = `
<table>
  <thead>
    <tr>${headerCells}</tr>
  </thead>
  <tbody>
    ${bodyRows}
  </tbody>
</table>
<p><br></p>`

  insertHtmlAtSelection(tableHtml)
}

const exportAsHtml = () => {
  exportedHtml.value = cleanEditorHtml(contentHtml.value)
  showExport.value = true
}

const copyExportedHtml = async () => {
  try {
    await navigator.clipboard.writeText(exportedHtml.value)
  } catch (error) {
    console.error('Copy failed', error)
  }
}

const headingCommandMap = {
  heading1: 'H1',
  heading2: 'H2',
  heading3: 'H3',
  heading4: 'H4',
  heading5: 'H5',
}

const handleAction = (action) => {
  if (headingCommandMap[action]) {
    runCommand('formatBlock', headingCommandMap[action])
    return
  }

  switch (action) {
    case 'bold':
      runCommand('bold')
      break
    case 'italic':
      runCommand('italic')
      break
    case 'quote':
      runCommand('formatBlock', 'BLOCKQUOTE')
      break
    case 'list':
      runCommand('insertUnorderedList')
      break
    case 'table':
      insertDefaultTable(3, 3)
      break
    case 'code':
      wrapSelectionAsCode()
      break
    case 'date':
      insertHtmlAtSelection(
        `<time datetime="${new Date().toISOString()}">${new Date().toLocaleString()}</time>`,
      )
      break
    case 'clear':
      contentHtml.value = ''
      nextTick(() => {
        syncEditorFromModel()
        focusEditor()
      })
      break
    case 'export':
      exportAsHtml()
      break
    default:
      break
  }

  nextTick(() => {
    updateActiveTools()
  })
}
</script>

<template>
  <main class="workspace">
    <header class="workspace-header">
      <p class="eyebrow">Editor Components</p>
      <input
        v-model="title"
        class="title-input"
        type="text"
        aria-label="Document title"
        placeholder="Document title"
      />
    </header>

    <EditorToolbar :active-tools="activeToolKeys" @action="handleAction" />

    <article class="editor-pane" aria-label="Rich text editor">
      <header class="pane-header">Rich Text Draft</header>
      <div
        ref="editor"
        class="editor-input"
        contenteditable="true"
        role="textbox"
        aria-multiline="true"
        data-placeholder="Start writing your content..."
        @input="onEditorInput"
        @keyup="onEditorCaretChange"
        @mouseup="onEditorCaretChange"
        @keydown="onEditorKeydown"
      ></div>

      <ul
        v-if="showMentionMenu"
        class="mention-menu"
        :style="mentionMenuStyle"
        role="listbox"
        aria-label="Variable mentions"
      >
        <li
          v-for="(item, index) in filteredMentions"
          :key="item.key"
          class="mention-item"
        >
          <button
            type="button"
            class="mention-btn"
            :class="{ active: index === mentionActiveIndex }"
            role="option"
            :aria-selected="index === mentionActiveIndex"
            @mousedown.prevent
            @click="selectMention(item)"
          >
            <span class="mention-label">{{ item.label }}</span>
            <code class="mention-code">{{ formatVariableToken(item.key) }}</code>
          </button>
        </li>
      </ul>
    </article>

    <section v-if="showExport" class="export-pane" aria-label="Exported HTML">
      <header class="pane-header export-header">
        <span>Export HTML</span>
        <button type="button" class="copy-btn" @click="copyExportedHtml">Copy HTML</button>
      </header>
      <textarea class="export-output" readonly :value="exportedHtml"></textarea>
    </section>

    <EditorStatusBar
      :words="words"
      :characters="characters"
      :lines="lines"
      :last-saved-label="lastSavedLabel"
    />
  </main>
</template>

<style scoped>
.workspace {
  width: min(1140px, 100%);
  margin: 0 auto;
  display: grid;
  gap: 1rem;
}

.workspace-header {
  display: grid;
  gap: 0.48rem;
  padding: 0.8rem 0.25rem 0.2rem;
}

.eyebrow {
  margin: 0;
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--editor-muted);
}

.title-input {
  width: 100%;
  border: none;
  background: transparent;
  color: var(--editor-ink);
  font-size: clamp(1.2rem, 2vw, 1.75rem);
  font-weight: 700;
  letter-spacing: -0.015em;
  outline: none;
  padding: 0;
}

.title-input::placeholder {
  color: #99a6b6;
}

.editor-pane,
.export-pane {
  position: relative;
  border: 1px solid var(--editor-border);
  border-radius: 1rem;
  background: #ffffff;
  overflow: hidden;
}

.pane-header {
  padding: 0.8rem 1rem;
  border-bottom: 1px solid var(--editor-border);
  color: var(--editor-muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 0.96rem;
}

.editor-input {
  min-height: 320px;
  padding: 0.95rem 1rem;
  outline: none;
  font-size: 1rem;
  line-height: 1.6;
  color: var(--editor-ink);
  overflow-y: auto;
  white-space: pre-wrap;
  word-break: break-word;
}

.editor-input:empty::before {
  content: attr(data-placeholder);
  color: #9aa8b8;
}

.editor-input :deep(h1),
.editor-input :deep(h2),
.editor-input :deep(h3),
.editor-input :deep(h4),
.editor-input :deep(h5) {
  margin: 0.5rem 0 0.7rem;
  color: #0b3b35;
}

.editor-input :deep(blockquote) {
  margin: 0.6rem 0;
  padding: 0.4rem 0.8rem;
  border-left: 4px solid var(--editor-brand);
  background: #effcf9;
}

.editor-input :deep(pre) {
  margin: 0.7rem 0;
  background: #0f172a;
  color: #d9e6ff;
  border-radius: 0.6rem;
  padding: 0.7rem;
}

.editor-input :deep(code) {
  font-family: var(--editor-mono);
  background: #f2f6fa;
  padding: 0.1rem 0.35rem;
  border-radius: 0.35rem;
}

.editor-input :deep(table) {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
  margin: 0.8rem 0;
}

.editor-input :deep(th),
.editor-input :deep(td) {
  border: 1px solid #cbd5e1;
  padding: 0.45rem 0.55rem;
  text-align: left;
  vertical-align: top;
}

.editor-input :deep(th) {
  background: #f3f7fb;
  font-weight: 600;
}

.mention-menu {
  position: fixed;
  z-index: 40;
  margin: 0;
  padding: 0.4rem;
  list-style: none;
  width: min(320px, calc(100vw - 16px));
  border: 1px solid var(--editor-border);
  border-radius: 0.7rem;
  background: #ffffff;
  box-shadow: 0 12px 30px rgba(15, 23, 42, 0.16);
}

.mention-item + .mention-item {
  margin-top: 0.2rem;
}

.mention-btn {
  width: 100%;
  border: none;
  background: transparent;
  color: var(--editor-ink);
  border-radius: 0.55rem;
  padding: 0.4rem 0.5rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
  cursor: pointer;
  text-align: left;
}

.mention-btn:hover,
.mention-btn.active {
  background: #ecfeff;
}

.mention-label {
  font-size: 0.88rem;
}

.mention-code {
  font-family: var(--editor-mono);
  font-size: 0.78rem;
  color: #0f766e;
  background: #f0fdfa;
  padding: 0.1rem 0.3rem;
  border-radius: 0.3rem;
}

.export-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.copy-btn {
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.55rem;
  font-size: 0.8rem;
  font-weight: 600;
  padding: 0.26rem 0.6rem;
  cursor: pointer;
}

.copy-btn:hover {
  border-color: var(--editor-brand);
}

.export-output {
  width: 100%;
  min-height: 210px;
  resize: vertical;
  border: none;
  outline: none;
  font-family: var(--editor-mono);
  font-size: 0.86rem;
  line-height: 1.45;
  color: var(--editor-ink);
  background: #fbfcfe;
  padding: 0.9rem 1rem;
}
</style>
