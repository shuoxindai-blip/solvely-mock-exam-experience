<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import HighlightablePassage from './components/HighlightablePassage.vue'

type Question = {
  number: number
  prompt: string
  passage: string
  options: string[]
  graph?: boolean
}

type HighlightColor = 'yellow' | 'pink' | 'blue'

type TextHighlight = {
  id: string
  start: number
  end: number
  color: HighlightColor
}

const questions: Record<number, Question> = {
  10: {
    number: 10,
    graph: true,
    prompt: 'Which choice most effectively uses data from the graph to complete the statement?',
    passage:
      'Researchers studying urban work habits hypothesized that the density of nearby coffee shops might predict how long people use their laptops each day. After measuring laptop usage across 60 sites in both summer and winter, the researchers noted that usage tended to increase as more coffee shops were located nearby. Examining the lines of best fit, the researchers concluded that this pattern was especially strong in summer, noting that ______.',
    options: [
      'around five hours of laptop use were observed at a site with three coffee shops within one kilometer, while around eleven hours were observed at a site with six coffee shops within one kilometer.',
      'around three hours of laptop use were observed at a site with ten coffee shops within one kilometer, while around six hours were observed at a site with thirty coffee shops within one kilometer.',
      'fewer than five hours of laptop use were observed at a site with ten coffee shops within one kilometer, while more than ten hours were observed at a site with thirty coffee shops within one kilometer.',
      'fewer than ten hours of laptop use were observed at a site with five coffee shops within one kilometer, while more than thirty hours were observed at a site with ten coffee shops within one kilometer.',
    ],
  },
  11: {
    number: 11,
    prompt: 'Which choice most logically completes the text?',
    passage:
      'Historians long believed that large-scale metalworking was nearly impossible before 5,000 years ago because early communities lacked the furnaces and tools needed to safely reach and control extremely high temperatures. The oldest confirmed example of complex metal smelting was a 4,500-year-old copper workshop discovered in the Balkans. Recently, however, archaeologists uncovered a remote cave site in the Caucasus Mountains containing a 12,000-year-old hearth lined with heat-resistant clay and traces of purified metal droplets. Chemical analysis indicates that the droplets were produced intentionally rather than by natural fire events. Thus, ______.',
    options: [
      'early metalworkers must have possessed written instructions that guided their smelting practices.',
      'working with purified metal would have required protective equipment that early humans likely did not have.',
      'there is no compelling evidence that complex metalworking was rare before 5,000 years ago.',
      'early human groups may have developed more sophisticated heat-control methods than scholars previously assumed.',
    ],
  },
}

const currentNumber = ref(10)
const answers = reactive<Record<number, number | null>>({ 10: null, 11: null })
const eliminated = reactive<Record<number, Set<number>>>({ 10: new Set(), 11: new Set() })
const review = reactive(new Set<number>())
const highlights = reactive<Record<number, TextHighlight[]>>({ 10: [], 11: [] })
const highlighterEnabled = ref(false)
const navigatorOpen = ref(false)
const directionsOpen = ref(false)
const moreOpen = ref(false)
const timerVisible = ref(true)
const reportToast = ref(false)
const timeRemaining = ref(27 * 60 + 56)
const leftWidth = ref(47.25)
const passageScroller = ref<HTMLElement | null>(null)
const questionScroller = ref<HTMLElement | null>(null)
let countdownId: number | undefined
let toastId: number | undefined

const currentQuestion = computed(() => questions[currentNumber.value] ?? questions[11])
const timeLabel = computed(() => {
  const minutes = Math.floor(timeRemaining.value / 60)
  const seconds = timeRemaining.value % 60
  return `${minutes}:${String(seconds).padStart(2, '0')}`
})

const answeredNumbers = computed(() => {
  const numbers = new Set([1, 2, 5])
  Object.entries(answers).forEach(([number, answer]) => {
    if (answer !== null) numbers.add(Number(number))
  })
  return numbers
})

function selectAnswer(index: number) {
  answers[currentNumber.value] = index
}

function toggleEliminated(index: number) {
  const active = eliminated[currentNumber.value] ?? new Set<number>()
  if (!eliminated[currentNumber.value]) eliminated[currentNumber.value] = active
  active.has(index) ? active.delete(index) : active.add(index)
}

function goToQuestion(number: number) {
  currentNumber.value = Math.max(1, Math.min(27, number))
  navigatorOpen.value = false
  void nextTick(() => {
    passageScroller.value?.scrollTo({ top: 0 })
    questionScroller.value?.scrollTo({ top: 0 })
  })
}

function nextQuestion() {
  if (currentNumber.value < 27) goToQuestion(currentNumber.value + 1)
}

function previousQuestion() {
  if (currentNumber.value > 1) goToQuestion(currentNumber.value - 1)
}

function toggleReview() {
  review.has(currentNumber.value) ? review.delete(currentNumber.value) : review.add(currentNumber.value)
}

function toggleHighlightMode() {
  highlighterEnabled.value = !highlighterEnabled.value
  moreOpen.value = false
}

function updateHighlights(value: TextHighlight[]) {
  highlights[currentNumber.value] = value
}

function showReportToast() {
  reportToast.value = true
  if (toastId) window.clearTimeout(toastId)
  toastId = window.setTimeout(() => (reportToast.value = false), 2400)
}

function beginResize(event: PointerEvent) {
  const onMove = (moveEvent: PointerEvent) => {
    leftWidth.value = Math.min(64, Math.max(35, (moveEvent.clientX / window.innerWidth) * 100))
  }
  const onUp = () => {
    window.removeEventListener('pointermove', onMove)
    window.removeEventListener('pointerup', onUp)
    document.body.classList.remove('is-resizing')
  }
  document.body.classList.add('is-resizing')
  window.addEventListener('pointermove', onMove)
  window.addEventListener('pointerup', onUp)
  event.currentTarget && (event.currentTarget as HTMLElement).setPointerCapture?.(event.pointerId)
}

function onKeydown(event: KeyboardEvent) {
  const target = event.target as HTMLElement | null
  if (['1', '2', '3', '4'].includes(event.key)) {
    event.preventDefault()
    selectAnswer(Number(event.key) - 1)
  }
  if (event.key === 'Enter') {
    if (target?.closest('button, .user-highlight')) return
    event.preventDefault()
    nextQuestion()
  }
  if (event.key === 'Escape') {
    navigatorOpen.value = false
    directionsOpen.value = false
    moreOpen.value = false
  }
}

onMounted(() => {
  countdownId = window.setInterval(() => {
    if (timeRemaining.value > 0) timeRemaining.value -= 1
  }, 1000)
  window.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  if (countdownId) window.clearInterval(countdownId)
  if (toastId) window.clearTimeout(toastId)
  window.removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <main class="exam-app" :style="{ '--split': `${leftWidth}%` }">
    <header class="exam-header">
      <div class="header-left">
        <button class="directions-button" type="button" :aria-expanded="directionsOpen" @click="directionsOpen = !directionsOpen">
          Directions
          <svg viewBox="0 0 16 16" aria-hidden="true"><path d="m4 6 4 4 4-4" /></svg>
        </button>
        <div v-if="directionsOpen" class="header-popover directions-popover">
          <strong>Reading and Writing</strong>
          <p>Choose the best answer to each question. You can return to any question before time expires.</p>
        </div>
      </div>

      <div class="timer-wrap">
        <strong v-if="timerVisible" class="timer" aria-live="polite">{{ timeLabel }}</strong>
        <span v-else class="timer-placeholder">Timer hidden</span>
        <button type="button" @click="timerVisible = !timerVisible">{{ timerVisible ? 'Hide' : 'Show' }}</button>
      </div>

      <div class="header-tools">
        <button
          class="tool-button"
          :class="{ active: highlighterEnabled }"
          type="button"
          :aria-pressed="highlighterEnabled"
          :aria-label="highlighterEnabled ? 'Turn off highlight mode' : 'Turn on highlight mode'"
          @click="toggleHighlightMode"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="m5 16 9.8-9.8a2 2 0 0 1 2.8 0l.2.2a2 2 0 0 1 0 2.8L8 19H5v-3Z" />
            <path d="M13.5 7.5 16.5 10.5M4 21h16" />
          </svg>
          <span>Highlight</span>
        </button>
        <div class="more-wrap">
          <button class="tool-button" type="button" :aria-expanded="moreOpen" @click="moreOpen = !moreOpen">
            <svg viewBox="0 0 24 24" aria-hidden="true"><circle cx="12" cy="5" r="1.3" fill="currentColor" /><circle cx="12" cy="12" r="1.3" fill="currentColor" /><circle cx="12" cy="19" r="1.3" fill="currentColor" /></svg>
            <span>More</span>
          </button>
          <div v-if="moreOpen" class="header-popover more-popover">
            <button type="button">Line reader</button>
            <button type="button">Question text size</button>
            <button type="button">Keyboard shortcuts</button>
          </div>
        </div>
      </div>
    </header>

    <section class="exam-workspace">
      <article ref="passageScroller" class="passage-panel" aria-label="Reading passage">
        <div class="passage-inner" :class="{ 'graph-question': currentNumber === 10 }">
          <template v-if="currentNumber === 10">
            <figure class="graph-figure" aria-label="Coffee shop density and laptop usage graph">
              <figcaption>Coffee Shop Density and<br />Average Laptop Usage Time</figcaption>
              <svg class="graph" viewBox="0 0 500 420" role="img" aria-labelledby="graph-title">
                <title id="graph-title">Coffee Shop Density and Average Laptop Usage Time</title>
                <g class="plot-area">
                <g class="grid-lines">
                  <path d="M95 30H430M95 73.3H430M95 116.7H430M95 160H430M95 203.3H430M95 246.7H430M95 290H430" />
                </g>
                <path class="axis" d="M95 21V290H440M89 290H101M89 246.7H101M89 203.3H101M89 160H101M89 116.7H101M89 73.3H101M89 30H101" />
                <g class="axis-labels">
                  <text x="83" y="296">0</text><text x="74" y="252">2</text><text x="74" y="209">4</text><text x="74" y="166">6</text><text x="74" y="123">8</text><text x="65" y="80">10</text><text x="65" y="36">12</text>
                  <text x="91" y="315">0</text><text x="199" y="315">10</text><text x="309" y="315">20</text><text x="420" y="315">30</text>
                  <text class="x-title" x="264" y="346" text-anchor="middle">Number of coffee shops within 1 kilometer</text>
                  <text class="y-title" x="23" y="160" text-anchor="middle" transform="rotate(-90 23 160)">Average daily laptop</text>
                  <text class="y-title" x="42" y="160" text-anchor="middle" transform="rotate(-90 42 160)">usage per person (hours)</text>
                </g>
                <polyline class="winter-line" points="95,225 205,214 315,182 425,160" />
                <rect class="winter-dot" x="90" y="220" width="10" height="10" /><rect class="winter-dot" x="200" y="209" width="10" height="10" /><rect class="winter-dot" x="310" y="177" width="10" height="10" /><rect class="winter-dot" x="420" y="155" width="10" height="10" />
                <polyline class="summer-line" points="95,247 205,203 315,138 425,52" />
                <path class="summer-dot" d="m95 240 7 7-7 7-7-7Z" /><path class="summer-dot" d="m205 196 7 7-7 7-7-7Z" /><path class="summer-dot" d="m315 131 7 7-7 7-7-7Z" /><path class="summer-dot" d="m425 45 7 7-7 7-7-7Z" />
                </g>
                <g class="legend">
                  <rect x="25" y="367" width="310" height="36" />
                  <path class="winter-line" d="M46 385h50" /><rect class="winter-dot" x="66" y="380" width="10" height="10" /><text x="108" y="391">Winter</text>
                  <path class="summer-line" d="M183 385h50" /><path class="summer-dot" d="m208 378 7 7-7 7-7-7Z" /><text x="246" y="391">Summer</text>
                </g>
              </svg>
            </figure>
          </template>
          <HighlightablePassage
            :key="currentNumber"
            :text="currentQuestion.passage"
            :enabled="highlighterEnabled"
            :model-value="highlights[currentNumber] ?? []"
            :extra-class="currentNumber === 10 ? 'graph-copy' : ''"
            @update:model-value="updateHighlights"
          />
        </div>
      </article>

      <div class="splitter" role="separator" aria-orientation="vertical" :aria-valuenow="Math.round(leftWidth)" tabindex="0" @pointerdown="beginResize">
        <span><i /><i /><i /></span>
      </div>

      <article ref="questionScroller" class="question-panel" aria-label="Answer choices">
        <div class="question-shell">
          <div class="question-toolbar">
            <span class="number-badge">{{ currentNumber }}</span>
            <button class="review-button" :class="{ active: review.has(currentNumber) }" type="button" @click="toggleReview">
              <svg viewBox="0 0 18 22" aria-hidden="true"><path d="M3 2.5h12v17l-6-4-6 4v-17Z" /></svg>
              Mark for Review
            </button>
            <button class="report-button" type="button" @click="showReportToast">
              <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 21V4m1 1h11l-2.2 4L17 13H6" /></svg>
              Report
            </button>
            <button class="accessibility-button" type="button" aria-label="Accessibility settings">
              <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M7 8h7a4 4 0 0 1 0 8H9m2-11L7 8l4 3m2 8 4-3-4-3" /></svg>
            </button>
          </div>

          <h1>{{ currentQuestion.prompt }}</h1>

          <div class="choices" role="radiogroup" :aria-label="currentQuestion.prompt">
            <div
              v-for="(option, index) in currentQuestion.options"
              :key="`${currentNumber}-${index}`"
              class="choice-row"
              :class="{ selected: answers[currentNumber] === index, eliminated: eliminated[currentNumber]?.has(index) }"
            >
              <button class="choice-card" type="button" role="radio" :aria-checked="answers[currentNumber] === index" @click="selectAnswer(index)">
                <span class="choice-letter">{{ String.fromCharCode(65 + index) }}</span>
                <span class="choice-copy">{{ option }}</span>
              </button>
              <button class="eliminate-button" type="button" :aria-label="`Cross out answer ${String.fromCharCode(65 + index)}`" @click="toggleEliminated(index)">
                <span>{{ String.fromCharCode(65 + index) }}</span>
              </button>
            </div>
          </div>

          <p class="keyboard-tip">
            Tip:&nbsp; press
            <kbd>1</kbd><kbd>2</kbd><kbd>3</kbd><kbd>4</kbd>
            to pick an answer, then <kbd class="enter-key">Enter</kbd> to go to the next question
          </p>
        </div>
      </article>
    </section>

    <footer class="exam-footer">
      <button class="question-count" type="button" :aria-expanded="navigatorOpen" @click="navigatorOpen = !navigatorOpen">
        {{ currentNumber }} of 27
        <svg viewBox="0 0 18 18" aria-hidden="true"><path :d="navigatorOpen ? 'm4 11 5-5 5 5' : 'm4 7 5 5 5-5'" /></svg>
      </button>

      <div v-if="navigatorOpen" class="navigator-card">
        <button class="navigator-close" type="button" aria-label="Close question navigator" @click="navigatorOpen = false">×</button>
        <h2>Section 1, Module 1:<br />Reading and Writing</h2>
        <div class="navigator-rule" />
        <div class="navigator-legend">
          <span><i class="unanswered-key" />Unanswered</span>
          <span><i class="review-key" />For Review</span>
        </div>
        <div class="question-grid">
          <button
            v-for="number in 27"
            :key="number"
            type="button"
            :class="{
              answered: answeredNumbers.has(number),
              current: currentNumber === number,
              review: review.has(number),
            }"
            @click="goToQuestion(number)"
          >
            {{ number }}
          </button>
        </div>
      </div>

      <div class="footer-actions">
        <button type="button" :disabled="currentNumber <= 1" @click="previousQuestion">Previous</button>
        <button type="button" :disabled="currentNumber >= 27" @click="nextQuestion">Next</button>
      </div>
    </footer>

    <div v-if="reportToast" class="toast" role="status">Thanks. This question has been reported for review.</div>
  </main>
</template>
