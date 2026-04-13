<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
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
  'img',
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
  img: new Set(['src', 'alt', 'title', 'width', 'height', 'data-align']),
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
const imageFileInput = ref(null)
const showImagePanel = ref(false)
const selectedImage = ref(null)
const imageInsertRange = ref(null)
const showTableContextMenu = ref(false)
const tableContextPosition = ref({ top: 0, left: 0 })
const tableContextCell = ref(null)
const selectedTableCells = ref([])
const tableRangeAnchor = ref(null)
const tableRangeFocus = ref(null)
const isTableDragging = ref(false)
const editingTableCell = ref(null)

const TABLE_SELECTED_CLASS = 'table-cell-selected'
const TABLE_EDITING_CLASS = 'table-cell-editing'
const IMAGE_SELECTED_CLASS = 'editor-image-selected'
const imageAlignOptions = ['left', 'center', 'right']

const imageForm = reactive({
  src: '',
  alt: '',
  title: '',
  width: '',
  height: '',
  align: 'center',
})

const tableContextActions = [
  { key: 'tableMerge', label: 'Merge' },
  { key: 'tableUnmerge', label: 'Unmerge' },
  { key: 'tableAddRow', label: 'Add Row' },
  { key: 'tableAddCol', label: 'Add Col' },
  { key: 'tableDeleteRow', label: 'Delete Row' },
  { key: 'tableDeleteCol', label: 'Delete Col' },
]

const extractText = (rawHtml) => {
  const temp = document.createElement('div')
  temp.innerHTML = rawHtml
  return (temp.innerText || '').replaceAll('\u00A0', ' ')
}

const isSafeHref = (href) =>
  /^(https?:|mailto:|tel:|\/|#)/i.test(href.trim())

const isSafeImageSrc = (src) =>
  /^(https?:|data:image\/|blob:|\/|\.\/|\.\.\/)/i.test(src.trim())

const normalizeImageDimension = (value) => {
  if (value === null || value === undefined) {
    return ''
  }
  const cleaned = String(value).trim()
  if (!cleaned) {
    return ''
  }
  if (!/^\d{1,4}$/.test(cleaned)) {
    return ''
  }
  const parsed = Number.parseInt(cleaned, 10)
  if (!Number.isFinite(parsed) || parsed <= 0) {
    return ''
  }
  return String(parsed)
}

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

    if (tag === 'img') {
      const src = node.getAttribute('src')?.trim() || ''
      if (!src || !isSafeImageSrc(src)) {
        node.remove()
        continue
      }

      const width = normalizeImageDimension(node.getAttribute('width'))
      const height = normalizeImageDimension(node.getAttribute('height'))
      if (width) {
        node.setAttribute('width', width)
      } else {
        node.removeAttribute('width')
      }
      if (height) {
        node.setAttribute('height', height)
      } else {
        node.removeAttribute('height')
      }

      const align = (node.getAttribute('data-align') || 'center').toLowerCase()
      if (imageAlignOptions.includes(align)) {
        node.setAttribute('data-align', align)
      } else {
        node.setAttribute('data-align', 'center')
      }
      if (!node.hasAttribute('alt')) {
        node.setAttribute('alt', '')
      }
    }
  }

  return doc.body.innerHTML
}

const cleanEditorHtml = (rawHtml) => {
  const safe = sanitizeHtml(rawHtml)
  const plain = extractText(safe).trim()
  const temp = document.createElement('div')
  temp.innerHTML = safe
  const hasImage = !!temp.querySelector('img')
  return plain || hasImage ? safe : ''
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
const tableContextMenuStyle = computed(() => ({
  top: `${tableContextPosition.value.top}px`,
  left: `${tableContextPosition.value.left}px`,
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
  stopTableDragSelection()
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

const closeTableContextMenu = () => {
  showTableContextMenu.value = false
  tableContextCell.value = null
}

const resetImageForm = () => {
  imageForm.src = ''
  imageForm.alt = ''
  imageForm.title = ''
  imageForm.width = ''
  imageForm.height = ''
  imageForm.align = 'center'
}

const closeImagePanel = () => {
  showImagePanel.value = false
}

const clearSelectedImage = () => {
  if (selectedImage.value) {
    selectedImage.value.classList.remove(IMAGE_SELECTED_CLASS)
  }
  selectedImage.value = null
}

const getClosestEditorImage = (node) => {
  if (!node) {
    return null
  }
  const element = node.nodeType === Node.TEXT_NODE ? node.parentElement : node
  if (!(element instanceof Element)) {
    return null
  }
  const image = element.closest('img')
  if (!image || !editor.value?.contains(image)) {
    return null
  }
  return image
}

const setImageFormFromElement = (image) => {
  imageForm.src = image.getAttribute('src') || ''
  imageForm.alt = image.getAttribute('alt') || ''
  imageForm.title = image.getAttribute('title') || ''
  imageForm.width = image.getAttribute('width') || ''
  imageForm.height = image.getAttribute('height') || ''
  const align = (image.getAttribute('data-align') || 'center').toLowerCase()
  imageForm.align = imageAlignOptions.includes(align) ? align : 'center'
}

const selectImage = (image, { openPanel = false } = {}) => {
  if (!image || !editor.value?.contains(image)) {
    clearSelectedImage()
    return
  }
  clearSelectedImage()
  selectedImage.value = image
  selectedImage.value.classList.add(IMAGE_SELECTED_CLASS)
  setImageFormFromElement(image)
  if (openPanel) {
    showImagePanel.value = true
  }
}

const getImagePayloadFromForm = () => {
  const src = imageForm.src.trim()
  if (!src || !isSafeImageSrc(src)) {
    return null
  }

  return {
    src,
    alt: imageForm.alt.trim(),
    title: imageForm.title.trim(),
    width: normalizeImageDimension(imageForm.width),
    height: normalizeImageDimension(imageForm.height),
    align: imageAlignOptions.includes(imageForm.align) ? imageForm.align : 'center',
  }
}

const applyImagePayloadToElement = (image, payload) => {
  image.setAttribute('src', payload.src)
  image.setAttribute('alt', payload.alt)
  if (payload.title) {
    image.setAttribute('title', payload.title)
  } else {
    image.removeAttribute('title')
  }
  if (payload.width) {
    image.setAttribute('width', payload.width)
  } else {
    image.removeAttribute('width')
  }
  if (payload.height) {
    image.setAttribute('height', payload.height)
  } else {
    image.removeAttribute('height')
  }
  image.setAttribute('data-align', payload.align)
}

const insertImageAtSelection = (payload) => {
  if (!editor.value) {
    return null
  }

  focusEditor()
  const selection = window.getSelection()
  if (!selection) {
    return null
  }

  let range = null
  if (selection.rangeCount && isNodeInEditor(selection.anchorNode)) {
    range = selection.getRangeAt(0)
  } else if (imageInsertRange.value) {
    range = imageInsertRange.value.cloneRange()
    selection.removeAllRanges()
    selection.addRange(range)
  } else {
    range = document.createRange()
    range.selectNodeContents(editor.value)
    range.collapse(false)
    selection.removeAllRanges()
    selection.addRange(range)
  }

  const image = document.createElement('img')
  applyImagePayloadToElement(image, payload)

  range.deleteContents()
  range.insertNode(image)
  const spacer = document.createTextNode(' ')
  image.after(spacer)

  const caretRange = document.createRange()
  caretRange.setStartAfter(spacer)
  caretRange.collapse(true)
  selection.removeAllRanges()
  selection.addRange(caretRange)
  imageInsertRange.value = caretRange.cloneRange()
  return image
}

const applyImageChanges = () => {
  const payload = getImagePayloadFromForm()
  if (!payload) {
    return
  }

  if (selectedImage.value && editor.value?.contains(selectedImage.value)) {
    applyImagePayloadToElement(selectedImage.value, payload)
    selectImage(selectedImage.value)
  } else {
    const inserted = insertImageAtSelection(payload)
    if (inserted) {
      selectImage(inserted)
    }
  }

  syncModelFromEditor()
  updateActiveTools()
}

const removeSelectedImage = () => {
  const image = selectedImage.value
  if (!image || !editor.value?.contains(image)) {
    clearSelectedImage()
    return
  }

  const selection = window.getSelection()
  const fallbackNode = image.previousSibling || image.parentNode
  image.remove()
  clearSelectedImage()
  syncModelFromEditor()

  if (selection && fallbackNode) {
    const range = document.createRange()
    if (fallbackNode.nodeType === Node.TEXT_NODE) {
      range.setStart(fallbackNode, fallbackNode.textContent?.length || 0)
    } else {
      range.selectNodeContents(fallbackNode)
      range.collapse(false)
    }
    selection.removeAllRanges()
    selection.addRange(range)
    imageInsertRange.value = range.cloneRange()
  }

  updateActiveTools()
}

const triggerImageUpload = () => {
  imageFileInput.value?.click()
}

const onImageFileChange = (event) => {
  const files = event.target?.files
  const file = files?.[0]
  if (!file || !file.type.startsWith('image/')) {
    if (event.target) {
      event.target.value = ''
    }
    return
  }

  const reader = new FileReader()
  reader.onload = () => {
    if (typeof reader.result !== 'string') {
      return
    }
    imageForm.src = reader.result
    applyImageChanges()
    showImagePanel.value = true
  }
  reader.readAsDataURL(file)

  if (event.target) {
    event.target.value = ''
  }
}

const openImagePanelForEditing = () => {
  closeMentionMenu()
  closeTableContextMenu()
  stopTableDragSelection()
  if (editingTableCell.value) {
    exitTableCellEditMode()
  }
  clearSelectedTableCells()

  if (selectedImage.value && editor.value?.contains(selectedImage.value)) {
    imageInsertRange.value = null
    setImageFormFromElement(selectedImage.value)
  } else {
    imageInsertRange.value = getCurrentEditorRange()
    resetImageForm()
  }

  showImagePanel.value = true
}

const clearSelectedTableCells = () => {
  for (const cell of selectedTableCells.value) {
    cell.classList.remove(TABLE_SELECTED_CLASS)
  }
  selectedTableCells.value = []
  tableRangeAnchor.value = null
  tableRangeFocus.value = null
}

const setSelectedTableCells = (cells) => {
  for (const cell of selectedTableCells.value) {
    cell.classList.remove(TABLE_SELECTED_CLASS)
  }

  const unique = []
  for (const cell of cells) {
    if (!cell || !editor.value?.contains(cell)) {
      continue
    }
    if (!unique.includes(cell)) {
      unique.push(cell)
    }
  }

  for (const cell of unique) {
    if (cell !== editingTableCell.value) {
      cell.classList.add(TABLE_SELECTED_CLASS)
    }
  }
  selectedTableCells.value = unique
}

const getPrimarySelectedTableCell = () => {
  if (
    editingTableCell.value &&
    editor.value?.contains(editingTableCell.value)
  ) {
    return editingTableCell.value
  }

  const first = selectedTableCells.value.find((cell) =>
    editor.value?.contains(cell),
  )
  return first || null
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

const getCurrentEditorRange = () => {
  const selection = window.getSelection()
  if (!selection || !selection.rangeCount) {
    return null
  }
  if (!isNodeInEditor(selection.anchorNode)) {
    return null
  }
  return selection.getRangeAt(0).cloneRange()
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

const getTableContextFromCell = (cell) => {
  if (!cell || !editor.value?.contains(cell)) {
    return null
  }

  const row = cell.closest('tr')
  const table = cell.closest('table')
  if (!row || !table) {
    return null
  }

  return { cell, row, table }
}

const getCurrentTableContext = () => {
  const selectedCell = getPrimarySelectedTableCell()
  if (selectedCell) {
    return getTableContextFromCell(selectedCell)
  }

  const selection = window.getSelection()
  if (!selection || !selection.rangeCount) {
    return null
  }

  const node = selection.anchorNode
  if (!isNodeInEditor(node)) {
    return null
  }

  const cell = getClosestTableCell(node)
  return getTableContextFromCell(cell)
}

const getClosestTableCell = (node) => {
  if (!node) {
    return null
  }

  const element =
    node.nodeType === Node.TEXT_NODE ? node.parentElement : node
  if (!(element instanceof Element)) {
    return null
  }

  const cell = element.closest('td, th')
  if (!cell || !editor.value?.contains(cell)) {
    return null
  }
  return cell
}

const getSelectedTableRange = () => {
  if (tableRangeAnchor.value && tableRangeFocus.value) {
    const range = selectTableCellRange(
      tableRangeAnchor.value,
      tableRangeFocus.value,
    )
    if (range) {
      return range
    }
  }

  const selection = window.getSelection()
  if (selection && selection.rangeCount) {
    const anchorCell = getClosestTableCell(selection.anchorNode)
    const focusCell = getClosestTableCell(selection.focusNode)
    if (anchorCell && focusCell) {
      const range = selectTableCellRange(anchorCell, focusCell)
      if (range) {
        return range
      }
    }
  }

  return null
}

const buildTableLayout = (table) => {
  const rows = Array.from(table.querySelectorAll('tr'))
  const grid = rows.map(() => [])
  const rowEntries = rows.map(() => [])

  for (let rowIndex = 0; rowIndex < rows.length; rowIndex += 1) {
    const row = rows[rowIndex]
    const cells = Array.from(row.children)
    let colIndex = 0

    for (const cell of cells) {
      while (grid[rowIndex][colIndex]) {
        colIndex += 1
      }

      const rowSpan = Math.max(1, Number.parseInt(cell.getAttribute('rowspan') || '1', 10))
      const colSpan = Math.max(1, Number.parseInt(cell.getAttribute('colspan') || '1', 10))

      rowEntries[rowIndex].push({
        cell,
        startCol: colIndex,
        rowSpan,
        colSpan,
      })

      for (let r = rowIndex; r < Math.min(rows.length, rowIndex + rowSpan); r += 1) {
        for (let c = colIndex; c < colIndex + colSpan; c += 1) {
          grid[r][c] = cell
        }
      }

      colIndex += colSpan
    }
  }

  return { rows, grid, rowEntries }
}

const getCellPositionInLayout = (layout, targetCell) => {
  for (let rowIndex = 0; rowIndex < layout.grid.length; rowIndex += 1) {
    const row = layout.grid[rowIndex]
    for (let colIndex = 0; colIndex < row.length; colIndex += 1) {
      if (row[colIndex] === targetCell) {
        return { row: rowIndex, col: colIndex }
      }
    }
  }
  return null
}

const selectTableCellRange = (anchorCell, focusCell = anchorCell) => {
  if (!anchorCell || !focusCell) {
    clearSelectedTableCells()
    return null
  }

  const anchorTable = anchorCell.closest('table')
  const focusTable = focusCell.closest('table')
  if (!anchorTable || anchorTable !== focusTable) {
    clearSelectedTableCells()
    return null
  }

  const layout = buildTableLayout(anchorTable)
  const anchorPos = getCellPositionInLayout(layout, anchorCell)
  const focusPos = getCellPositionInLayout(layout, focusCell)
  if (!anchorPos || !focusPos) {
    clearSelectedTableCells()
    return null
  }

  const rowStart = Math.min(anchorPos.row, focusPos.row)
  const rowEnd = Math.max(anchorPos.row, focusPos.row)
  const colStart = Math.min(anchorPos.col, focusPos.col)
  const colEnd = Math.max(anchorPos.col, focusPos.col)

  const cells = []
  for (let rowIndex = rowStart; rowIndex <= rowEnd; rowIndex += 1) {
    for (let colIndex = colStart; colIndex <= colEnd; colIndex += 1) {
      const cell = layout.grid[rowIndex]?.[colIndex]
      if (cell) {
        cells.push(cell)
      }
    }
  }

  tableRangeAnchor.value = anchorCell
  tableRangeFocus.value = focusCell
  setSelectedTableCells(cells)

  return {
    table: anchorTable,
    layout,
    rowStart,
    rowEnd,
    colStart,
    colEnd,
    rowCount: rowEnd - rowStart + 1,
    colCount: colEnd - colStart + 1,
    cells: selectedTableCells.value,
  }
}

const placeCaretAtCellEnd = (cell) => {
  const selection = window.getSelection()
  if (!selection) {
    return
  }

  const range = document.createRange()
  range.selectNodeContents(cell)
  range.collapse(false)
  selection.removeAllRanges()
  selection.addRange(range)
}

const exitTableCellEditMode = () => {
  if (!editingTableCell.value) {
    return
  }

  const cell = editingTableCell.value
  cell.classList.remove(TABLE_EDITING_CLASS)
  cell.removeAttribute('contenteditable')
  editingTableCell.value = null
  syncModelFromEditor()
  updateActiveTools()
}

const enterTableCellEditMode = (cell) => {
  if (!cell || !editor.value?.contains(cell)) {
    return
  }

  if (editingTableCell.value && editingTableCell.value !== cell) {
    exitTableCellEditMode()
  }

  editingTableCell.value = cell
  cell.classList.remove(TABLE_SELECTED_CLASS)
  cell.classList.add(TABLE_EDITING_CLASS)
  cell.setAttribute('contenteditable', 'true')
  cell.focus()
  placeCaretAtCellEnd(cell)
}

const getCellFromPointerEvent = (event) => {
  const fromTarget = getClosestTableCell(event.target)
  if (fromTarget) {
    return fromTarget
  }

  const pointElement = document.elementFromPoint(event.clientX, event.clientY)
  return getClosestTableCell(pointElement)
}

const stopTableDragSelection = () => {
  if (!isTableDragging.value) {
    return
  }
  isTableDragging.value = false
  document.removeEventListener('mousemove', onTableDragMove)
  document.removeEventListener('mouseup', onTableDragEnd)
}

const setCaretToElementStart = (element) => {
  const selection = window.getSelection()
  if (!selection) {
    return
  }

  const range = document.createRange()
  range.selectNodeContents(element)
  range.collapse(true)
  selection.removeAllRanges()
  selection.addRange(range)
}

const removeTableWithFallbackParagraph = (table) => {
  const paragraph = document.createElement('p')
  paragraph.innerHTML = '<br>'
  table.replaceWith(paragraph)
  clearSelectedTableCells()
  if (editingTableCell.value) {
    editingTableCell.value = null
  }
  setCaretToElementStart(paragraph)
}

const mergeSelectedTableCells = () => {
  const selection = getSelectedTableRange()
  if (!selection) {
    return
  }

  if (selection.rowCount === 1 && selection.colCount === 1) {
    return
  }

  const topLeftRow = selection.layout.rows[selection.rowStart]
  const topLeftCell = Array.from(topLeftRow.children)[selection.colStart]
  if (!topLeftCell) {
    return
  }

  const existingHtml = topLeftCell.innerHTML.trim()
  const mergedHtml = selection.cells
    .filter((cell) => cell !== topLeftCell)
    .map((cell) => cell.innerHTML.trim())
    .filter(Boolean)

  if (mergedHtml.length) {
    const parts = []
    if (existingHtml && existingHtml !== '<br>') {
      parts.push(existingHtml)
    }
    parts.push(...mergedHtml)
    topLeftCell.innerHTML = parts.join('<br>')
  }

  if (selection.rowCount > 1) {
    topLeftCell.setAttribute('rowspan', String(selection.rowCount))
  } else {
    topLeftCell.removeAttribute('rowspan')
  }

  if (selection.colCount > 1) {
    topLeftCell.setAttribute('colspan', String(selection.colCount))
  } else {
    topLeftCell.removeAttribute('colspan')
  }

  for (let index = selection.cells.length - 1; index >= 0; index -= 1) {
    const cell = selection.cells[index]
    if (cell !== topLeftCell) {
      cell.remove()
    }
  }

  tableRangeAnchor.value = topLeftCell
  tableRangeFocus.value = topLeftCell
  selectTableCellRange(topLeftCell, topLeftCell)
  setCaretToElementStart(topLeftCell)
  syncModelFromEditor()
}

const unmergeCurrentTableCell = (contextOverride = null) => {
  const context = contextOverride || getCurrentTableContext()
  if (!context) {
    return
  }

  const { cell, table } = context
  const layout = buildTableLayout(table)

  let baseRowIndex = -1
  let cellEntry = null
  for (let i = 0; i < layout.rowEntries.length; i += 1) {
    const entry = layout.rowEntries[i].find((item) => item.cell === cell)
    if (entry) {
      baseRowIndex = i
      cellEntry = entry
      break
    }
  }

  if (!cellEntry || baseRowIndex < 0) {
    return
  }

  const rowSpan = cellEntry.rowSpan
  const colSpan = cellEntry.colSpan
  if (rowSpan === 1 && colSpan === 1) {
    return
  }

  const baseCol = cellEntry.startCol
  const isHeader = cell.tagName === 'TH'

  cell.removeAttribute('rowspan')
  cell.removeAttribute('colspan')

  const createCell = () => {
    const nextCell = document.createElement(isHeader ? 'th' : 'td')
    nextCell.innerHTML = '<br>'
    return nextCell
  }

  for (let rowIndex = baseRowIndex; rowIndex < Math.min(layout.rows.length, baseRowIndex + rowSpan); rowIndex += 1) {
    const row = layout.rows[rowIndex]
    const entries = layout.rowEntries[rowIndex]
    const afterCol = baseCol + colSpan
    const referenceEntry = entries.find((entry) => entry.startCol >= afterCol)
    const referenceCell = referenceEntry?.cell || null
    const toInsert = rowIndex === baseRowIndex ? colSpan - 1 : colSpan

    for (let count = 0; count < toInsert; count += 1) {
      const newCell = createCell()
      if (referenceCell) {
        row.insertBefore(newCell, referenceCell)
      } else {
        row.appendChild(newCell)
      }
    }
  }

  tableRangeAnchor.value = cell
  tableRangeFocus.value = cell
  selectTableCellRange(cell, cell)
  setCaretToElementStart(cell)
  syncModelFromEditor()
}

const deleteCurrentTableRow = (contextOverride = null) => {
  const context = contextOverride || getCurrentTableContext()
  if (!context) {
    return
  }

  const { cell, row, table } = context
  const allRows = Array.from(table.querySelectorAll('tr'))
  if (!allRows.length) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  if (allRows.length === 1) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  const rowIndex = allRows.indexOf(row)
  const cellIndex = Math.max(0, Array.from(row.children).indexOf(cell))
  const nextRow = allRows[rowIndex + 1] || allRows[rowIndex - 1]

  row.remove()

  if (!table.isConnected || !table.querySelector('tr')) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  if (nextRow && nextRow.isConnected) {
    const nextCells = Array.from(nextRow.children)
    const targetCell = nextCells[Math.min(cellIndex, Math.max(0, nextCells.length - 1))]
    if (targetCell) {
      tableRangeAnchor.value = targetCell
      tableRangeFocus.value = targetCell
      selectTableCellRange(targetCell, targetCell)
      setCaretToElementStart(targetCell)
    }
  }

  syncModelFromEditor()
}

const addCurrentTableRow = (contextOverride = null) => {
  const context = contextOverride || getCurrentTableContext()
  if (!context) {
    return
  }

  const { row, table } = context
  const cells = Array.from(row.children)
  if (!cells.length) {
    return
  }

  const isHeaderRow = row.parentElement?.tagName === 'THEAD'
  const newRow = document.createElement('tr')
  for (const cell of cells) {
    const tag = !isHeaderRow && cell.tagName === 'TH' ? 'th' : 'td'
    const newCell = document.createElement(tag)
    newCell.textContent = tag === 'th' ? 'Header' : 'Cell'
    newRow.appendChild(newCell)
  }

  if (isHeaderRow) {
    let tbody = table.querySelector('tbody')
    if (!tbody) {
      tbody = document.createElement('tbody')
      table.appendChild(tbody)
    }
    const firstBodyRow = tbody.querySelector('tr')
    if (firstBodyRow) {
      tbody.insertBefore(newRow, firstBodyRow)
    } else {
      tbody.appendChild(newRow)
    }
  } else {
    row.insertAdjacentElement('afterend', newRow)
  }

  const firstCell = newRow.children[0]
  if (firstCell) {
    tableRangeAnchor.value = firstCell
    tableRangeFocus.value = firstCell
    selectTableCellRange(firstCell, firstCell)
    setCaretToElementStart(firstCell)
  }
  syncModelFromEditor()
}

const addCurrentTableColumn = (contextOverride = null) => {
  const context = contextOverride || getCurrentTableContext()
  if (!context) {
    return
  }

  const { cell, row, table } = context
  const rowCells = Array.from(row.children)
  const targetIndex = Math.max(0, rowCells.indexOf(cell)) + 1

  const allRows = Array.from(table.querySelectorAll('tr'))
  for (const currentRow of allRows) {
    const currentCells = Array.from(currentRow.children)
    const baseCell = currentCells[Math.min(targetIndex - 1, Math.max(0, currentCells.length - 1))]
    const newTag = baseCell?.tagName === 'TH' ? 'th' : 'td'
    const newCell = document.createElement(newTag)
    newCell.textContent = newTag === 'th' ? 'Header' : 'Cell'

    if (targetIndex >= currentCells.length) {
      currentRow.appendChild(newCell)
    } else {
      currentRow.insertBefore(newCell, currentCells[targetIndex])
    }
  }

  const latestRows = Array.from(table.querySelectorAll('tr'))
  const rowIndex = Math.max(0, allRows.indexOf(row))
  const targetRow = latestRows[Math.min(rowIndex, latestRows.length - 1)]
  const targetCell = targetRow?.children[targetIndex]
  if (targetCell) {
    tableRangeAnchor.value = targetCell
    tableRangeFocus.value = targetCell
    selectTableCellRange(targetCell, targetCell)
    setCaretToElementStart(targetCell)
  }

  syncModelFromEditor()
}

const deleteCurrentTableColumn = (contextOverride = null) => {
  const context = contextOverride || getCurrentTableContext()
  if (!context) {
    return
  }

  const { cell, row, table } = context
  const allRows = Array.from(table.querySelectorAll('tr'))
  if (!allRows.length) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  const cellIndex = Math.max(0, Array.from(row.children).indexOf(cell))
  const maxColumns = allRows.reduce(
    (max, currentRow) => Math.max(max, currentRow.children.length),
    0,
  )

  if (maxColumns <= 1) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  for (const currentRow of allRows) {
    const cells = Array.from(currentRow.children)
    if (cellIndex < cells.length) {
      cells[cellIndex].remove()
    }
    if (!currentRow.children.length) {
      currentRow.remove()
    }
  }

  const rowsLeft = Array.from(table.querySelectorAll('tr'))
  if (!rowsLeft.length) {
    removeTableWithFallbackParagraph(table)
    syncModelFromEditor()
    return
  }

  const rowIndex = Math.max(0, allRows.indexOf(row))
  const targetRow = rowsLeft[Math.min(rowIndex, rowsLeft.length - 1)]
  const targetCells = Array.from(targetRow.children)
  const targetCell = targetCells[Math.min(cellIndex, Math.max(0, targetCells.length - 1))]
  if (targetCell) {
    tableRangeAnchor.value = targetCell
    tableRangeFocus.value = targetCell
    selectTableCellRange(targetCell, targetCell)
    setCaretToElementStart(targetCell)
  }

  syncModelFromEditor()
}

const updateActiveTools = () => {
  const active = new Set()
  const selectedCell = getPrimarySelectedTableCell()
  const selectedImageNode =
    selectedImage.value && editor.value?.contains(selectedImage.value)
      ? selectedImage.value
      : null

  if (!selectedImageNode && selectedImage.value) {
    clearSelectedImage()
  }

  if (selectedImageNode) {
    active.add('image')
  }
  if (showImagePanel.value) {
    active.add('image')
  }

  const selection = window.getSelection()
  const node =
    selection && selection.rangeCount && isNodeInEditor(selection.anchorNode)
      ? selection.anchorNode
      : null

  if (node) {
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
  }

  const inTable =
    !!selectedCell ||
    !!(node && findAncestorByTag(node, ['TABLE', 'THEAD', 'TBODY', 'TR', 'TH', 'TD']))

  if (inTable) {
    active.add('table')
    active.add('tableMerge')
    active.add('tableAddRow')
    active.add('tableAddCol')
    active.add('tableDeleteRow')
    active.add('tableDeleteCol')

    const context = selectedCell
      ? getTableContextFromCell(selectedCell)
      : getCurrentTableContext()
    if (context) {
      const rowSpan = Math.max(1, Number.parseInt(context.cell.getAttribute('rowspan') || '1', 10))
      const colSpan = Math.max(1, Number.parseInt(context.cell.getAttribute('colspan') || '1', 10))
      if (rowSpan > 1 || colSpan > 1) {
        active.add('tableUnmerge')
      }
    }
  }

  if (!node && !selectedCell && !selectedImageNode && !showImagePanel.value) {
    activeToolKeys.value = []
    return
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
  if (selectedImage.value && !editor.value?.contains(selectedImage.value)) {
    clearSelectedImage()
  }
  if (!selectedImage.value) {
    imageInsertRange.value = getCurrentEditorRange()
  }
  syncModelFromEditor()
  updateMentionFromSelection()
  updateActiveTools()
}

const onEditorCaretChange = () => {
  if (!selectedImage.value) {
    imageInsertRange.value = getCurrentEditorRange()
  }
  updateMentionFromSelection()
  updateActiveTools()
}

const onTableDragMove = (event) => {
  if (!isTableDragging.value || !tableRangeAnchor.value) {
    return
  }

  const cell = getCellFromPointerEvent(event)
  if (!cell) {
    return
  }

  const anchorTable = tableRangeAnchor.value.closest('table')
  if (!anchorTable || cell.closest('table') !== anchorTable) {
    return
  }

  event.preventDefault()
  tableRangeFocus.value = cell
  selectTableCellRange(tableRangeAnchor.value, cell)
  updateActiveTools()
}

const onTableDragEnd = () => {
  stopTableDragSelection()
  updateActiveTools()
}

const onEditorMouseDown = (event) => {
  if (event.button !== 0) {
    return
  }

  const image = getClosestEditorImage(event.target)
  if (image) {
    stopTableDragSelection()
    if (editingTableCell.value) {
      exitTableCellEditMode()
    }
    clearSelectedTableCells()
    closeMentionMenu()
    closeTableContextMenu()
    event.preventDefault()
    selectImage(image)
    updateActiveTools()
    return
  }

  if (selectedImage.value) {
    clearSelectedImage()
  }

  const cell = getClosestTableCell(event.target)
  if (!cell) {
    stopTableDragSelection()
    if (editingTableCell.value) {
      exitTableCellEditMode()
    }
    clearSelectedTableCells()
    updateActiveTools()
    return
  }

  if (editingTableCell.value === cell) {
    return
  }

  if (editingTableCell.value && editingTableCell.value !== cell) {
    exitTableCellEditMode()
  }

  closeMentionMenu()
  closeTableContextMenu()

  event.preventDefault()
  tableRangeAnchor.value = cell
  tableRangeFocus.value = cell
  selectTableCellRange(cell, cell)
  updateActiveTools()

  isTableDragging.value = true
  document.addEventListener('mousemove', onTableDragMove)
  document.addEventListener('mouseup', onTableDragEnd)
}

const onEditorDblClick = (event) => {
  const image = getClosestEditorImage(event.target)
  if (image) {
    event.preventDefault()
    selectImage(image, { openPanel: true })
    updateActiveTools()
    return
  }

  const cell = getClosestTableCell(event.target)
  if (!cell) {
    return
  }

  event.preventDefault()
  stopTableDragSelection()
  tableRangeAnchor.value = cell
  tableRangeFocus.value = cell
  selectTableCellRange(cell, cell)
  enterTableCellEditMode(cell)
  updateActiveTools()
}

const onEditorKeydown = (event) => {
  if (event.key === 'Escape') {
    if (showMentionMenu.value) {
      event.preventDefault()
      closeMentionMenu()
      return
    }
    if (showTableContextMenu.value) {
      event.preventDefault()
      closeTableContextMenu()
      return
    }
    if (showImagePanel.value) {
      event.preventDefault()
      closeImagePanel()
      return
    }
    if (selectedImage.value) {
      event.preventDefault()
      clearSelectedImage()
      updateActiveTools()
      return
    }
    if (editingTableCell.value) {
      event.preventDefault()
      exitTableCellEditMode()
      const cell = getPrimarySelectedTableCell()
      if (cell) {
        selectTableCellRange(cell, cell)
      }
      return
    }
  }

  if (
    selectedImage.value &&
    (event.key === 'Backspace' || event.key === 'Delete')
  ) {
    event.preventDefault()
    removeSelectedImage()
    return
  }

  if (!showMentionMenu.value) {
    if (selectedTableCells.value.length && !editingTableCell.value) {
      const ignoredKeys = new Set([
        'Shift',
        'Control',
        'Alt',
        'Meta',
        'CapsLock',
        'Tab',
        'ArrowUp',
        'ArrowDown',
        'ArrowLeft',
        'ArrowRight',
      ])
      if (!ignoredKeys.has(event.key)) {
        event.preventDefault()
      }
    }
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

}

const onDocumentPointerDown = (event) => {
  const target = event.target

  if (selectedImage.value) {
    const clickedImage = getClosestEditorImage(target)
    const clickedInImageUi =
      target instanceof Element &&
      (target.closest('.image-panel') || target.closest('.toolbar'))
    if (clickedImage !== selectedImage.value && !clickedInImageUi) {
      clearSelectedImage()
      updateActiveTools()
    }
  }

  if (showMentionMenu.value) {
    if (
      editor.value?.contains(target) ||
      (target instanceof Element && target.closest('.mention-menu'))
    ) {
      // Keep mention menu open while interacting with editor or mention list.
    } else {
      closeMentionMenu()
    }
  }

  if (showTableContextMenu.value) {
    if (target instanceof Element && target.closest('.table-context-menu')) {
      return
    }
    closeTableContextMenu()
  }

  if (editingTableCell.value) {
    if (
      target instanceof Node &&
      editingTableCell.value.contains(target)
    ) {
      return
    }
    exitTableCellEditMode()
  }
}

const onSelectionChange = () => {
  updateActiveTools()
}

const onEditorContextMenu = (event) => {
  const context = getTableContextFromCell(getClosestTableCell(event.target))
  if (!context) {
    closeTableContextMenu()
    return
  }

  event.preventDefault()
  closeMentionMenu()
  if (editingTableCell.value && editingTableCell.value !== context.cell) {
    exitTableCellEditMode()
  }
  if (!selectedTableCells.value.includes(context.cell)) {
    tableRangeAnchor.value = context.cell
    tableRangeFocus.value = context.cell
    selectTableCellRange(context.cell, context.cell)
  }
  tableContextCell.value = context.cell

  const maxWidth = 200
  const maxHeight = 240
  tableContextPosition.value = {
    left: Math.max(8, Math.min(event.clientX, window.innerWidth - maxWidth - 8)),
    top: Math.max(8, Math.min(event.clientY, window.innerHeight - maxHeight - 8)),
  }
  showTableContextMenu.value = true
}

const isTableContextActionDisabled = (action) => {
  const context = getTableContextFromCell(tableContextCell.value)
  if (!context) {
    return true
  }

  if (action === 'tableUnmerge') {
    const rowSpan = Math.max(1, Number.parseInt(context.cell.getAttribute('rowspan') || '1', 10))
    const colSpan = Math.max(1, Number.parseInt(context.cell.getAttribute('colspan') || '1', 10))
    return rowSpan === 1 && colSpan === 1
  }

  if (action === 'tableMerge') {
    const selected = getSelectedTableRange()
    return !selected || (selected.rowCount === 1 && selected.colCount === 1)
  }

  return false
}

const onTableContextAction = (action) => {
  const context = getTableContextFromCell(tableContextCell.value)
  if (!context) {
    closeTableContextMenu()
    return
  }

  closeTableContextMenu()
  handleAction(action, context)
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

const handleAction = (action, contextOverride = null) => {
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
    case 'image':
      if (showImagePanel.value) {
        closeImagePanel()
      } else {
        openImagePanelForEditing()
      }
      break
    case 'table':
      insertDefaultTable(3, 3)
      break
    case 'tableMerge':
      mergeSelectedTableCells()
      break
    case 'tableUnmerge':
      unmergeCurrentTableCell(contextOverride)
      break
    case 'tableAddRow':
      addCurrentTableRow(contextOverride)
      break
    case 'tableAddCol':
      addCurrentTableColumn(contextOverride)
      break
    case 'tableDeleteRow':
      deleteCurrentTableRow(contextOverride)
      break
    case 'tableDeleteCol':
      deleteCurrentTableColumn(contextOverride)
      break
    case 'clear':
      closeMentionMenu()
      closeTableContextMenu()
      stopTableDragSelection()
      exitTableCellEditMode()
      clearSelectedTableCells()
      clearSelectedImage()
      closeImagePanel()
      resetImageForm()
      imageInsertRange.value = null
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
    <EditorToolbar :active-tools="activeToolKeys" @action="handleAction" />
    <input ref="imageFileInput" class="image-file-input" type="file" accept="image/*" @change="onImageFileChange" />

    <section v-if="showImagePanel" class="image-panel" aria-label="Image settings">
      <header class="image-panel-header">Image</header>
      <form class="image-form" @submit.prevent="applyImageChanges">
        <label class="image-field">
          <span class="image-label">Image URL</span>
          <input v-model="imageForm.src" class="image-input" type="url"
            placeholder="https://example.com/your-image.png" />
        </label>

        <label class="image-field">
          <span class="image-label">Alt</span>
          <input v-model="imageForm.alt" class="image-input" type="text" placeholder="Describe this image" />
        </label>

        <label class="image-field">
          <span class="image-label">Title</span>
          <input v-model="imageForm.title" class="image-input" type="text" placeholder="Optional title" />
        </label>

        <div class="image-grid-two">
          <label class="image-field">
            <span class="image-label">Width</span>
            <input v-model="imageForm.width" class="image-input" type="number" min="1" max="4096" placeholder="auto" />
          </label>
          <label class="image-field">
            <span class="image-label">Height</span>
            <input v-model="imageForm.height" class="image-input" type="number" min="1" max="4096" placeholder="auto" />
          </label>
        </div>

        <label class="image-field">
          <span class="image-label">Align</span>
          <select v-model="imageForm.align" class="image-input image-select">
            <option v-for="option in imageAlignOptions" :key="option" :value="option">
              {{ option }}
            </option>
          </select>
        </label>

        <div class="image-actions">
          <button type="submit" class="image-btn primary">
            {{ selectedImage ? 'Update Image' : 'Insert Image' }}
          </button>
          <button type="button" class="image-btn" @click="triggerImageUpload">
            {{ selectedImage ? 'Replace Upload' : 'Upload Image' }}
          </button>
          <button v-if="selectedImage" type="button" class="image-btn danger" @click="removeSelectedImage">
            Delete Image
          </button>
          <button type="button" class="image-btn ghost" @click="closeImagePanel">
            Close
          </button>
        </div>
      </form>
    </section>

    <article class="editor-pane" aria-label="Rich text editor">
      <!-- <header class="pane-header">Rich Text Draft</header> -->
      <div ref="editor" class="editor-input" contenteditable="true" role="textbox" aria-multiline="true"
        data-placeholder="Start writing your content..." @mousedown="onEditorMouseDown" @dblclick="onEditorDblClick"
        @input="onEditorInput" @keyup="onEditorCaretChange" @mouseup="onEditorCaretChange" @keydown="onEditorKeydown"
        @contextmenu="onEditorContextMenu"></div>

      <ul v-if="showMentionMenu" class="mention-menu" :style="mentionMenuStyle" role="listbox"
        aria-label="Variable mentions">
        <li v-for="(item, index) in filteredMentions" :key="item.key" class="mention-item">
          <button type="button" class="mention-btn" :class="{ active: index === mentionActiveIndex }" role="option"
            :aria-selected="index === mentionActiveIndex" @mousedown.prevent @click="selectMention(item)">
            <span class="mention-label">{{ item.label }}</span>
            <code class="mention-code">{{ formatVariableToken(item.key) }}</code>
          </button>
        </li>
      </ul>

      <ul v-if="showTableContextMenu" class="table-context-menu" :style="tableContextMenuStyle" role="menu"
        aria-label="Table quick actions">
        <li v-for="item in tableContextActions" :key="item.key" class="table-context-item">
          <button type="button" class="table-context-btn" role="menuitem"
            :disabled="isTableContextActionDisabled(item.key)" @mousedown.prevent
            @click="onTableContextAction(item.key)">
            {{ item.label }}
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

    <!-- <EditorStatusBar :words="words" :characters="characters" :lines="lines" :last-saved-label="lastSavedLabel" /> -->
  </main>
</template>

<style scoped>
.workspace {
  /* width: min(1140px, 100%);
  margin: 0 auto;
  display: grid;
  gap: 1rem; */
  /* border-radius: 5rem; */
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
  /* border: 1px solid var(--editor-border); */
  border-top: 1px solid var(--editor-border);
  /* border-radius: 1rem; */
  background: #ffffff;
  overflow: hidden;
  border-bottom-left-radius: 0.5rem;
  border-bottom-right-radius: 0.5rem;
}

/* .pane-header {
  padding: 0.8rem 1rem;
  border-bottom: 1px solid var(--editor-border);
  color: var(--editor-muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 0.96rem;
} */

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

.editor-input :deep(img) {
  display: block;
  max-width: 100%;
  height: auto;
  margin: 0.7rem auto;
  border-radius: 0.45rem;
}

.editor-input :deep(img[data-align='left']) {
  margin-left: 0;
  margin-right: auto;
}

.editor-input :deep(img[data-align='center']) {
  margin-left: auto;
  margin-right: auto;
}

.editor-input :deep(img[data-align='right']) {
  margin-left: auto;
  margin-right: 0;
}

.editor-input :deep(.editor-image-selected) {
  outline: 2px solid #0ea5e9;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px rgba(14, 165, 233, 0.16);
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
  user-select: none;
  cursor: cell;
}

.editor-input :deep(th) {
  background: #f3f7fb;
  font-weight: 600;
}

.editor-input :deep(.table-cell-selected) {
  background: #dbf5ff;
  box-shadow: inset 0 0 0 2px #38bdf8;
}

.editor-input :deep(.table-cell-editing) {
  user-select: text;
  cursor: text;
  background: #ffffff;
  box-shadow: inset 0 0 0 2px #0f766e;
}

.image-file-input {
  display: none;
}

.image-panel {
  border: 1px solid var(--editor-border);
  border-radius: 0.95rem;
  background: linear-gradient(160deg, #ffffff, #f7fbff 45%, #eefcf8);
  padding: 0.85rem 0.9rem;
  display: grid;
  gap: 0.75rem;
}

.image-panel-header {
  color: var(--editor-muted);
  text-transform: uppercase;
  letter-spacing: 0.08em;
  font-size: 0.8rem;
}

.image-form {
  display: grid;
  gap: 0.65rem;
}

.image-grid-two {
  display: grid;
  gap: 0.65rem;
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.image-field {
  display: grid;
  gap: 0.28rem;
}

.image-label {
  font-size: 0.78rem;
  color: var(--editor-muted);
}

.image-input {
  width: 100%;
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.6rem;
  font-size: 0.86rem;
  padding: 0.44rem 0.56rem;
  outline: none;
}

.image-input:focus {
  border-color: #0f766e;
  box-shadow: 0 0 0 2px rgba(15, 118, 110, 0.14);
}

.image-select {
  text-transform: capitalize;
}

.image-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.45rem;
}

.image-btn {
  border: 1px solid var(--editor-border);
  background: #ffffff;
  color: var(--editor-ink);
  border-radius: 0.58rem;
  font-size: 0.82rem;
  font-weight: 600;
  padding: 0.34rem 0.64rem;
  cursor: pointer;
}

.image-btn:hover {
  border-color: #0f766e;
}

.image-btn.primary {
  background: #0f766e;
  border-color: #0f766e;
  color: #ffffff;
}

.image-btn.danger {
  border-color: #fecaca;
  color: #b91c1c;
  background: #fff1f2;
}

.image-btn.ghost {
  background: #f8fafc;
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

.mention-item+.mention-item {
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

.table-context-menu {
  position: fixed;
  z-index: 45;
  margin: 0;
  padding: 0.35rem;
  list-style: none;
  width: min(190px, calc(100vw - 16px));
  border: 1px solid var(--editor-border);
  border-radius: 0.65rem;
  background: #ffffff;
  box-shadow: 0 14px 28px rgba(15, 23, 42, 0.18);
}

.table-context-item+.table-context-item {
  margin-top: 0.15rem;
}

.table-context-btn {
  width: 100%;
  border: none;
  background: transparent;
  color: var(--editor-ink);
  border-radius: 0.5rem;
  padding: 0.38rem 0.48rem;
  font-size: 0.84rem;
  text-align: left;
  cursor: pointer;
}

.table-context-btn:hover:not(:disabled) {
  background: #ecfeff;
}

.table-context-btn:disabled {
  color: #9aa8b8;
  cursor: not-allowed;
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

@media (max-width: 720px) {
  .image-grid-two {
    grid-template-columns: 1fr;
  }
}
</style>
