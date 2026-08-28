<script setup lang="ts">
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type CalculatorMode = 'graphing' | 'scientific'

type GeoGebraApplet = {
  inject: (elementId: string) => void
}

declare global {
  interface Window {
    GGBApplet?: new (parameters: Record<string, unknown>, useBrowserForJS?: boolean) => GeoGebraApplet
  }
}

const emit = defineEmits<{
  close: []
}>()

const mode = ref<CalculatorMode>('graphing')
const status = ref<'loading' | 'ready' | 'error'>('loading')
const host = ref<HTMLElement | null>(null)
const hostId = `geogebra-${Math.random().toString(36).slice(2)}`
let mounted = false

function loadGeoGebra() {
  if (window.GGBApplet) return Promise.resolve()
  const existing = document.querySelector<HTMLScriptElement>('script[data-geogebra-deploy]')
  if (existing) {
    return new Promise<void>((resolve, reject) => {
      existing.addEventListener('load', () => resolve(), { once: true })
      existing.addEventListener('error', () => reject(new Error('GeoGebra failed to load')), { once: true })
    })
  }

  return new Promise<void>((resolve, reject) => {
    const script = document.createElement('script')
    script.src = 'https://www.geogebra.org/apps/deployggb.js'
    script.async = true
    script.dataset.geogebraDeploy = 'true'
    script.addEventListener('load', () => resolve(), { once: true })
    script.addEventListener('error', () => reject(new Error('GeoGebra failed to load')), { once: true })
    document.head.appendChild(script)
  })
}

async function mountApplet() {
  status.value = 'loading'
  await nextTick()
  if (!mounted || !host.value) return
  host.value.replaceChildren()

  try {
    await loadGeoGebra()
    if (!mounted || !host.value || !window.GGBApplet) return
    const width = Math.max(420, Math.round(host.value.clientWidth))
    const height = Math.max(520, Math.round(host.value.clientHeight))
    const applet = new window.GGBApplet(
      {
        appName: mode.value,
        width,
        height,
        language: 'en',
        showToolBar: false,
        showMenuBar: false,
        showAlgebraInput: mode.value === 'graphing',
        showKeyboardOnFocus: false,
        showResetIcon: false,
        enableShiftDragZoom: true,
        enableLabelDrags: false,
        enableRightClick: false,
        appletOnLoad() {
          if (!mounted) return
          status.value = 'ready'
        },
      },
      true,
    )
    applet.inject(hostId)
  } catch {
    status.value = 'error'
  }
}

function popOut() {
  const destination = mode.value === 'graphing' ? 'https://www.geogebra.org/graphing' : 'https://www.geogebra.org/scientific'
  window.open(destination, '_blank', 'noopener,noreferrer')
}

watch(mode, mountApplet)

onMounted(() => {
  mounted = true
  void mountApplet()
})

onBeforeUnmount(() => {
  mounted = false
  host.value?.replaceChildren()
})
</script>

<template>
  <section class="calculator-shell" aria-label="GeoGebra calculator">
    <div class="calculator-titlebar">
      <strong>Calculator</strong>
      <label class="calculator-mode">
        <span class="sr-only">Calculator mode</span>
        <select v-model="mode">
          <option value="graphing">Graphing</option>
          <option value="scientific">Scientific</option>
        </select>
        <svg viewBox="0 0 16 16" aria-hidden="true"><path d="m4 6 4 4 4-4" /></svg>
      </label>
      <div class="calculator-title-actions">
        <button type="button" @click="popOut">
          <svg viewBox="0 0 20 20" aria-hidden="true"><path d="M11 3h6v6M10 10l7-7M16 11v5H4V4h5" /></svg>
          Pop Out
        </button>
        <button class="calculator-close" type="button" aria-label="Close calculator" @click="emit('close')">×</button>
      </div>
    </div>
    <div :id="hostId" ref="host" class="geogebra-host" />
    <div v-if="status !== 'ready'" class="calculator-status" role="status">
      <span v-if="status === 'loading'" class="calculator-spinner" />
      <p v-if="status === 'loading'">Loading GeoGebra…</p>
      <template v-else>
        <strong>GeoGebra couldn’t load.</strong>
        <p>Check the network connection, then try again.</p>
        <button type="button" @click="mountApplet">Retry</button>
      </template>
    </div>
  </section>
</template>

<style scoped>
.calculator-shell {
  position: relative;
  display: grid;
  grid-template-rows: 54px minmax(0, 1fr);
  width: 100%;
  height: 100%;
  min-height: 0;
  border: 1px solid #d2d2d2;
  background: #fff;
}

.calculator-titlebar {
  position: relative;
  z-index: 2;
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  padding: 0 16px;
  border-bottom: 1px solid #cfcfcf;
  background: #fff;
  font-family: Arial, Helvetica, sans-serif;
}

.calculator-titlebar strong {
  font-size: 16px;
}

.calculator-mode {
  position: relative;
  display: flex;
  align-items: center;
}

.calculator-mode select {
  min-width: 144px;
  height: 38px;
  padding: 0 36px 0 14px;
  border: 0;
  border-radius: 9px;
  appearance: none;
  background: #f1f1f1;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
}

.calculator-mode svg {
  position: absolute;
  right: 12px;
  width: 16px;
  height: 16px;
  fill: none;
  stroke: #777;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 1.8;
  pointer-events: none;
}

.calculator-title-actions {
  display: flex;
  justify-content: flex-end;
  gap: 13px;
}

.calculator-title-actions button {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 6px;
  border: 0;
  background: transparent;
  color: #666;
  cursor: pointer;
}

.calculator-title-actions svg {
  width: 17px;
  height: 17px;
  fill: none;
  stroke: currentColor;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 1.6;
}

.calculator-title-actions .calculator-close {
  font-size: 29px;
  font-weight: 300;
  line-height: 1;
}

.geogebra-host {
  width: 100%;
  height: 100%;
  min-height: 520px;
  overflow: hidden;
}

.calculator-status {
  position: absolute;
  inset: 55px 0 0;
  z-index: 1;
  display: grid;
  place-content: center;
  justify-items: center;
  padding: 24px;
  background: #fff;
  color: #666;
  text-align: center;
}

.calculator-status p {
  margin: 8px 0 0;
  font-size: 14px;
}

.calculator-status button {
  margin-top: 14px;
  padding: 8px 16px;
  border: 0;
  border-radius: 7px;
  background: #0877d1;
  color: #fff;
  cursor: pointer;
}

.calculator-spinner {
  width: 28px;
  height: 28px;
  border: 3px solid #d8d8d8;
  border-top-color: #0bbce8;
  border-radius: 50%;
  animation: calculator-spin 700ms linear infinite;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

@keyframes calculator-spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 760px) {
  .calculator-titlebar {
    grid-template-columns: auto 1fr auto;
    gap: 8px;
    padding: 0 10px;
  }

  .calculator-mode select {
    min-width: 116px;
  }

  .calculator-title-actions button:first-child {
    display: none;
  }
}

@media (prefers-reduced-motion: reduce) {
  .calculator-spinner { animation: none; }
}
</style>
