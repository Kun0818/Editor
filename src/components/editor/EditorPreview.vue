<script setup>
import { computed } from 'vue'

const props = defineProps({
  content: {
    type: String,
    required: true,
  },
})

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
  'a',
  'time',
  'br',
  'hr',
])

const allowedAttrs = {
  a: new Set(['href', 'title', 'target', 'rel']),
  time: new Set(['datetime']),
}

const isSafeHref = (href) =>
  /^(https?:|mailto:|tel:|\/|#)/i.test(href.trim())

const sanitizeHtml = (rawHtml) => {
  const parser = new DOMParser()
  const doc = parser.parseFromString(rawHtml, 'text/html')

  const allNodes = Array.from(doc.body.querySelectorAll('*'))
  for (const node of allNodes) {
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

const renderedHtml = computed(() => sanitizeHtml(props.content))
</script>

<template>
  <section class="preview" aria-label="Live preview">
    <header class="preview-header">
      <h2>Live Preview</h2>
    </header>

    <div class="preview-body">
      <div v-if="renderedHtml.trim().length" v-html="renderedHtml"></div>
      <p v-else class="placeholder">Start typing to see a formatted preview.</p>
    </div>
  </section>
</template>

<style scoped>
.preview {
  border: 1px solid var(--editor-border);
  border-radius: 1rem;
  background: #ffffff;
  min-height: 320px;
  display: flex;
  flex-direction: column;
}

.preview-header {
  padding: 0.8rem 1rem;
  border-bottom: 1px solid var(--editor-border);
}

.preview-header h2 {
  margin: 0;
  font-size: 0.96rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: var(--editor-muted);
}

.preview-body {
  padding: 1rem;
  overflow: auto;
}

.preview-body h1,
.preview-body h2,
.preview-body h3 {
  margin: 0 0 0.75rem;
  color: #0b3b35;
}

.preview-body h1 {
  font-size: 1.42rem;
}

.preview-body h2 {
  font-size: 1.22rem;
}

.preview-body h3 {
  font-size: 1.06rem;
}

.preview-body p,
.preview-body li,
.preview-body blockquote {
  margin: 0 0 0.7rem;
  line-height: 1.55;
}

.preview-body ul {
  margin: 0 0 0.85rem;
  padding-inline-start: 1.2rem;
}

.preview-body blockquote {
  padding: 0.55rem 0.85rem;
  border-left: 4px solid var(--editor-brand);
  border-radius: 0 0.6rem 0.6rem 0;
  background: #effcf9;
}

.preview-body code {
  font-family: var(--editor-mono);
  font-size: 0.88rem;
  background: #f2f6fa;
  padding: 0.12rem 0.38rem;
  border-radius: 0.38rem;
}

.preview-body pre {
  margin: 0 0 0.85rem;
  background: #0f172a;
  color: #d9e6ff;
  border-radius: 0.72rem;
  padding: 0.85rem;
  overflow: auto;
}

.preview-body pre code {
  background: transparent;
  color: inherit;
  padding: 0;
  font-size: 0.82rem;
}

.placeholder {
  color: var(--editor-muted);
}
</style>
