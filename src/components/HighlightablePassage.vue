<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type HighlightColor = 'yellow' | 'pink' | 'blue'

type TextHighlight = {
  id: string
  start: number
  end: number
  color: HighlightColor
}

type TextSegment = {
  text: string
  highlight?: TextHighlight
}

const props = withDefaults(
  defineProps<{
    text: string
    enabled: boolean
    modelValue: TextHighlight[]
    extraClass?: string
  }>(),
  {
    extraClass: '',
  },
)

const emit = defineEmits<{
  'update:modelValue': [value: TextHighlight[]]
}>()

const passageRoot = ref<HTMLElement | null>(null)
const toolbarVisible = ref(false)
const activeHighlightId = ref<string | null>(null)
const toolbarLeft = ref(0)
const toolbarTop = ref(0)
const toolbarPlacement = ref<'above' | 'below'>('above')
let scrollOwner: HTMLElement | null = null

const palette: Array<{ color: HighlightColor; label: string }> = [
  { color: 'yellow', label: 'Yellow highlight' },
  { color: 'pink', label: 'Pink highlight' },
  { color: 'blue', label: 'Blue highlight' },
]

const activeHighlight = computed(() => props.modelValue.find((highlight) => highlight.id === activeHighlightId.value) ?? null)

const segments = computed<TextSegment[]>(() => {
  const ordered = [...props.modelValue]
    .filter((highlight) => highlight.start >= 0 && highlight.end > highlight.start && highlight.end <= props.text.length)
    .sort((a, b) => a.start - b.start)
  const output: TextSegment[] = []
  let cursor = 0

  ordered.forEach((highlight) => {
    if (highlight.start > cursor) output.push({ text: props.text.slice(cursor, highlight.start) })
    if (highlight.end > cursor) {
      const start = Math.max(cursor, highlight.start)
      output.push({ text: props.text.slice(start, highlight.end), highlight: { ...highlight, start } })
      cursor = highlight.end
    }
  })

  if (cursor < props.text.length) output.push({ text: props.text.slice(cursor) })
  return output
})

const toolbarStyle = computed(() => ({
  left: `${toolbarLeft.value}px`,
  top: `${toolbarTop.value}px`,
}))

function positionToolbar(rect: DOMRect) {
  const horizontalPadding = 86
  toolbarLeft.value = Math.min(window.innerWidth - horizontalPadding, Math.max(horizontalPadding, rect.left + rect.width / 2))
  if (rect.top >= 54) {
    toolbarPlacement.value = 'above'
    toolbarTop.value = rect.top - 8
  } else {
    toolbarPlacement.value = 'below'
    toolbarTop.value = rect.bottom + 8
  }
}

function clearNativeSelection() {
  window.getSelection()?.removeAllRanges()
}

function closeToolbar(clearSelection = true) {
  toolbarVisible.value = false
  activeHighlightId.value = null
  if (clearSelection) clearNativeSelection()
}

function textOffset(root: HTMLElement, container: Node, offset: number) {
  const prefix = document.createRange()
  prefix.selectNodeContents(root)
  prefix.setEnd(container, offset)
  return prefix.toString().length
}

function createHighlight(start: number, end: number, rect: DOMRect) {
  const overlaps = props.modelValue.filter((highlight) => highlight.start < end && highlight.end > start)
  const mergedStart = overlaps.reduce((value, highlight) => Math.min(value, highlight.start), start)
  const mergedEnd = overlaps.reduce((value, highlight) => Math.max(value, highlight.end), end)
  const id = `highlight-${currentId++}`
  const next = props.modelValue.filter((highlight) => !overlaps.includes(highlight))
  next.push({ id, start: mergedStart, end: mergedEnd, color: 'yellow' })
  next.sort((a, b) => a.start - b.start)
  emit('update:modelValue', next)
  activeHighlightId.value = id
  positionToolbar(rect)
  toolbarVisible.value = true
  void nextTick(clearNativeSelection)
}

let currentId = 1

function captureSelection() {
  if (!props.enabled || !passageRoot.value) return
  const selection = window.getSelection()
  if (!selection || selection.isCollapsed || selection.rangeCount === 0) return
  const range = selection.getRangeAt(0)
  if (!passageRoot.value.contains(range.commonAncestorContainer)) return

  const selectedText = range.toString()
  const trimmedText = selectedText.trim()
  if (!trimmedText) return

  const leadingWhitespace = selectedText.length - selectedText.trimStart().length
  const trailingWhitespace = selectedText.length - selectedText.trimEnd().length
  const rawStart = textOffset(passageRoot.value, range.startContainer, range.startOffset)
  const rawEnd = textOffset(passageRoot.value, range.endContainer, range.endOffset)
  const start = rawStart + leadingWhitespace
  const end = rawEnd - trailingWhitespace
  if (end <= start) return

  createHighlight(start, end, range.getBoundingClientRect())
}

function onPassageMouseup() {
  window.requestAnimationFrame(captureSelection)
}

function openExistingHighlight(event: MouseEvent | KeyboardEvent, id: string) {
  const target = event.currentTarget as HTMLElement
  activeHighlightId.value = id
  positionToolbar(target.getBoundingClientRect())
  toolbarVisible.value = true
  clearNativeSelection()
}

function setColor(color: HighlightColor) {
  if (!activeHighlightId.value) return
  emit(
    'update:modelValue',
    props.modelValue.map((highlight) => (highlight.id === activeHighlightId.value ? { ...highlight, color } : highlight)),
  )
}

function removeHighlight() {
  if (!activeHighlightId.value) return
  emit('update:modelValue', props.modelValue.filter((highlight) => highlight.id !== activeHighlightId.value))
  closeToolbar()
}

function onDocumentPointerdown(event: PointerEvent) {
  if (!toolbarVisible.value) return
  const target = event.target as HTMLElement | null
  if (target?.closest('.highlight-toolbar') || target?.closest('.user-highlight')) return
  closeToolbar(false)
}

function onEscape(event: KeyboardEvent) {
  if (event.key === 'Escape' && toolbarVisible.value) closeToolbar()
}

function onViewportChange() {
  if (toolbarVisible.value) closeToolbar(false)
}

watch(
  () => props.enabled,
  (enabled) => {
    if (!enabled) closeToolbar()
  },
)

watch(
  () => props.text,
  () => closeToolbar(),
)

onMounted(() => {
  scrollOwner = passageRoot.value?.closest('.passage-panel') as HTMLElement | null
  scrollOwner?.addEventListener('scroll', onViewportChange, { passive: true })
  window.addEventListener('resize', onViewportChange, { passive: true })
  document.addEventListener('pointerdown', onDocumentPointerdown)
  document.addEventListener('keydown', onEscape)
})

onBeforeUnmount(() => {
  scrollOwner?.removeEventListener('scroll', onViewportChange)
  window.removeEventListener('resize', onViewportChange)
  document.removeEventListener('pointerdown', onDocumentPointerdown)
  document.removeEventListener('keydown', onEscape)
})
</script>

<template>
  <p
    ref="passageRoot"
    class="passage-copy highlightable-passage"
    :class="[extraClass, { 'highlight-mode-active': enabled }]"
    @mouseup="onPassageMouseup"
  ><template v-for="(segment, index) in segments" :key="segment.highlight?.id ?? `text-${index}`"><mark
      v-if="segment.highlight"
      class="user-highlight"
      :class="`highlight-${segment.highlight.color}`"
      :data-highlight-id="segment.highlight.id"
      tabindex="0"
      aria-label="Highlighted text. Press Enter to edit."
      @click="openExistingHighlight($event, segment.highlight.id)"
      @keydown.enter.stop.prevent="openExistingHighlight($event, segment.highlight.id)"
      @keydown.space.stop.prevent="openExistingHighlight($event, segment.highlight.id)"
    >{{ segment.text }}</mark><span v-else>{{ segment.text }}</span></template></p>

  <Teleport to="body">
    <div
      v-if="toolbarVisible && activeHighlight"
      class="highlight-toolbar"
      :class="`placement-${toolbarPlacement}`"
      :style="toolbarStyle"
      role="toolbar"
      aria-label="Highlight colors"
    >
      <button
        v-for="item in palette"
        :key="item.color"
        class="highlight-swatch"
        :class="[`swatch-${item.color}`, { selected: activeHighlight.color === item.color }]"
        type="button"
        :aria-label="item.label"
        :aria-pressed="activeHighlight.color === item.color"
        :title="item.label"
        @pointerdown.prevent
        @click="setColor(item.color)"
      ><svg v-if="activeHighlight.color === item.color" viewBox="0 0 14 14" aria-hidden="true"><path d="m3 7 2.4 2.5L11 4" /></svg></button>
      <span class="toolbar-divider" />
      <button class="remove-highlight" type="button" aria-label="Remove highlight" title="Remove highlight" @pointerdown.prevent @click="removeHighlight">
        <svg viewBox="0 0 20 20" aria-hidden="true"><path d="M5 6h10M8 6V4h4v2m-6 2 1 8h6l1-8" /></svg>
      </button>
    </div>
  </Teleport>
</template>

<style scoped>
.highlightable-passage {
  -webkit-user-select: text;
  user-select: text;
}

.highlightable-passage.highlight-mode-active {
  cursor: text;
}

.highlightable-passage.highlight-mode-active::selection,
.highlightable-passage.highlight-mode-active :deep(*)::selection {
  background: rgba(255, 235, 52, 0.72);
  color: inherit;
}

.user-highlight {
  margin: 0;
  padding: 0;
  border-radius: 2px;
  color: inherit;
  cursor: pointer;
  box-decoration-break: clone;
  -webkit-box-decoration-break: clone;
  transition: filter 120ms ease, box-shadow 120ms ease;
}

.user-highlight:hover,
.user-highlight:focus-visible {
  filter: saturate(1.08);
  box-shadow: 0 0 0 2px rgba(40, 40, 40, 0.12);
  outline: none;
}

.highlight-yellow { background: #fff34c; }
.highlight-pink { background: #ff9abb; }
.highlight-blue { background: #83d5ff; }

.highlight-toolbar {
  position: fixed;
  z-index: 120;
  display: flex;
  align-items: center;
  gap: 7px;
  height: 34px;
  padding: 0 9px;
  border: 1px solid #d9d9d9;
  border-radius: 7px;
  background: #fff;
  box-shadow: 0 7px 18px rgba(0, 0, 0, 0.14);
  animation: toolbar-in 120ms ease-out;
}

.highlight-toolbar.placement-above {
  transform: translate(-50%, -100%);
}

.highlight-toolbar.placement-below {
  transform: translateX(-50%);
}

.highlight-toolbar::after {
  position: absolute;
  left: 50%;
  width: 8px;
  height: 8px;
  border-right: 1px solid #d9d9d9;
  border-bottom: 1px solid #d9d9d9;
  background: #fff;
  content: '';
}

.highlight-toolbar.placement-above::after {
  bottom: -5px;
  transform: translateX(-50%) rotate(45deg);
}

.highlight-toolbar.placement-below::after {
  top: -5px;
  transform: translateX(-50%) rotate(225deg);
}

.highlight-swatch,
.remove-highlight {
  position: relative;
  display: grid;
  width: 22px;
  height: 22px;
  padding: 0;
  place-items: center;
  border: 1px solid rgba(0, 0, 0, 0.09);
  border-radius: 50%;
  cursor: pointer;
}

.highlight-swatch:hover {
  transform: scale(1.08);
}

.highlight-swatch.selected {
  box-shadow: 0 0 0 2px #fff, 0 0 0 3px #343434;
}

.highlight-swatch svg {
  width: 13px;
  height: 13px;
  fill: none;
  stroke: rgba(0, 0, 0, 0.68);
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 1.8;
}

.swatch-yellow { background: #fff34c; }
.swatch-pink { background: #ff9abb; }
.swatch-blue { background: #83d5ff; }

.toolbar-divider {
  width: 1px;
  height: 19px;
  margin: 0 1px;
  background: #dedede;
}

.remove-highlight {
  border: 0;
  border-radius: 5px;
  background: transparent;
  color: #646464;
}

.remove-highlight:hover {
  background: #f1f1f1;
  color: #202020;
}

.remove-highlight svg {
  width: 17px;
  height: 17px;
  fill: none;
  stroke: currentColor;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 1.45;
}

@keyframes toolbar-in {
  from { opacity: 0; }
  to { opacity: 1; }
}

@media (prefers-reduced-motion: reduce) {
  .highlight-toolbar {
    animation: none;
  }
}
</style>
