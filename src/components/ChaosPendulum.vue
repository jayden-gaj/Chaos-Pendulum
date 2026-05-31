<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const sceneRef = ref(null)
const chartRef = ref(null)
const running = ref(false)

const WIDTH = 600
const HEIGHT = 600
const ANCHOR = { x: WIDTH / 2, y: HEIGHT / 2 }
const BOB_RADIUS = 14
const SCENE_PAD = 12 // gap between max reach and canvas edge
const GRAVITY = 800 // px/s²

const CHART_W = 260
const CHART_H = 200
const THETA_RANGE = Math.PI

// User-tunable parameters. Applied on Reset.
const rodLen1 = ref(160)
const rodLen2 = ref(160)
const mass1 = ref(4)
const mass2 = ref(4)
const startAngleDeg = ref(90)

// Simulator state — angles from downward vertical, angular velocities in rad/s.
let theta1 = 0
let omega1 = 0
let theta2 = 0
let omega2 = 0

// Render bits.
let mainCanvas
let mainCtx
let trailCanvas
let trailCtx
let chartCtx
let rafId = null
let lastTip = null

// Drag state.
let dragging = null // 'bob1' | 'bob2' | null

// Phase plot data.
const SAMPLE_LIMIT = 20000
const phaseSamples = [] // { theta1, omega1, theta2, omega2 }
let omegaScale = 0.5

// ─────────────────────────────────────────────────────────────
// Physics — double pendulum equations of motion, RK4-integrated.
// ─────────────────────────────────────────────────────────────

function derivs(state) {
  const [t1, w1, t2, w2] = state
  const m1 = mass1.value
  const m2 = mass2.value
  const L1 = rodLen1.value
  const L2 = rodLen2.value
  const g = GRAVITY
  const d = t1 - t2
  const denom = 2 * m1 + m2 - m2 * Math.cos(2 * t1 - 2 * t2)

  const a1 =
    (-g * (2 * m1 + m2) * Math.sin(t1) -
      m2 * g * Math.sin(t1 - 2 * t2) -
      2 * Math.sin(d) * m2 * (w2 * w2 * L2 + w1 * w1 * L1 * Math.cos(d))) /
    (L1 * denom)

  const a2 =
    (2 *
      Math.sin(d) *
      (w1 * w1 * L1 * (m1 + m2) +
        g * (m1 + m2) * Math.cos(t1) +
        w2 * w2 * L2 * m2 * Math.cos(d))) /
    (L2 * denom)

  return [w1, a1, w2, a2]
}

function rk4(state, dt) {
  const k1 = derivs(state)
  const s2 = state.map((x, i) => x + 0.5 * dt * k1[i])
  const k2 = derivs(s2)
  const s3 = state.map((x, i) => x + 0.5 * dt * k2[i])
  const k3 = derivs(s3)
  const s4 = state.map((x, i) => x + dt * k3[i])
  const k4 = derivs(s4)
  return state.map(
    (x, i) => x + (dt / 6) * (k1[i] + 2 * k2[i] + 2 * k3[i] + k4[i])
  )
}

function getRenderScale() {
  // Shrink the drawing if total rod length would exceed the canvas, so the
  // whole reachable disc is always visible.
  const maxReach = rodLen1.value + rodLen2.value + BOB_RADIUS + SCENE_PAD
  const halfMin = Math.min(WIDTH, HEIGHT) / 2
  return Math.min(1, halfMin / maxReach)
}

function getPositions() {
  const s = getRenderScale()
  const L1 = rodLen1.value * s
  const L2 = rodLen2.value * s
  const bob1 = {
    x: ANCHOR.x + L1 * Math.sin(theta1),
    y: ANCHOR.y + L1 * Math.cos(theta1),
  }
  const bob2 = {
    x: bob1.x + L2 * Math.sin(theta2),
    y: bob1.y + L2 * Math.cos(theta2),
  }
  return { bob1, bob2 }
}

// ─────────────────────────────────────────────────────────────
// Build / teardown
// ─────────────────────────────────────────────────────────────

function build() {
  // Main scene canvas.
  mainCanvas = document.createElement('canvas')
  mainCanvas.width = WIDTH
  mainCanvas.height = HEIGHT
  mainCanvas.style.display = 'block'
  mainCanvas.style.background = '#0b0d12'
  sceneRef.value.style.position = 'relative'
  sceneRef.value.appendChild(mainCanvas)
  mainCtx = mainCanvas.getContext('2d')

  // Trail layer.
  trailCanvas = document.createElement('canvas')
  trailCanvas.width = WIDTH
  trailCanvas.height = HEIGHT
  trailCanvas.style.position = 'absolute'
  trailCanvas.style.inset = '0'
  trailCanvas.style.pointerEvents = 'none'
  sceneRef.value.appendChild(trailCanvas)
  trailCtx = trailCanvas.getContext('2d')

  chartCtx = chartRef.value.getContext('2d')

  // Initial state from chosen start angle.
  const a = (startAngleDeg.value * Math.PI) / 180
  theta1 = a
  theta2 = a
  omega1 = 0
  omega2 = 0
  lastTip = null

  mainCanvas.addEventListener('pointerdown', onPointerDown)
  mainCanvas.addEventListener('pointermove', onPointerMove)
  window.addEventListener('pointerup', onPointerUp)

  drawCharts()
  loop()
}

function teardown() {
  if (rafId !== null) cancelAnimationFrame(rafId)
  rafId = null
  if (mainCanvas) {
    mainCanvas.removeEventListener('pointerdown', onPointerDown)
    mainCanvas.removeEventListener('pointermove', onPointerMove)
    mainCanvas.remove()
    mainCanvas = null
    mainCtx = null
  }
  if (trailCanvas) {
    trailCanvas.remove()
    trailCanvas = null
    trailCtx = null
  }
  window.removeEventListener('pointerup', onPointerUp)
  chartCtx = null
  lastTip = null
}

// ─────────────────────────────────────────────────────────────
// Main loop
// ─────────────────────────────────────────────────────────────

const SUBSTEPS = 8
const FRAME_DT = 1 / 60

function loop() {
  if (running.value && !dragging) {
    const dt = FRAME_DT / SUBSTEPS
    for (let i = 0; i < SUBSTEPS; i++) {
      const next = rk4([theta1, omega1, theta2, omega2], dt)
      theta1 = next[0]
      omega1 = next[1]
      theta2 = next[2]
      omega2 = next[3]
    }
    samplePhase()
  }
  render()
  rafId = requestAnimationFrame(loop)
}

function samplePhase() {
  phaseSamples.push({ theta1, omega1, theta2, omega2 })
  if (phaseSamples.length > SAMPLE_LIMIT) phaseSamples.shift()
  omegaScale = Math.max(
    omegaScale,
    Math.abs(omega1) * 1.05,
    Math.abs(omega2) * 1.05
  )
  drawCharts()
}

function render() {
  const { bob1, bob2 } = getPositions()

  mainCtx.fillStyle = '#0b0d12'
  mainCtx.fillRect(0, 0, WIDTH, HEIGHT)

  // Rods.
  mainCtx.strokeStyle = '#e5e7eb'
  mainCtx.lineWidth = 2
  mainCtx.beginPath()
  mainCtx.moveTo(ANCHOR.x, ANCHOR.y)
  mainCtx.lineTo(bob1.x, bob1.y)
  mainCtx.lineTo(bob2.x, bob2.y)
  mainCtx.stroke()

  // Anchor.
  mainCtx.fillStyle = '#888'
  mainCtx.beginPath()
  mainCtx.arc(ANCHOR.x, ANCHOR.y, 6, 0, Math.PI * 2)
  mainCtx.fill()

  // Bobs.
  mainCtx.fillStyle = '#41b883'
  mainCtx.beginPath()
  mainCtx.arc(bob1.x, bob1.y, BOB_RADIUS, 0, Math.PI * 2)
  mainCtx.fill()

  mainCtx.fillStyle = '#ff7e5f'
  mainCtx.beginPath()
  mainCtx.arc(bob2.x, bob2.y, BOB_RADIUS, 0, Math.PI * 2)
  mainCtx.fill()

  // Trail (only while simulating).
  if (running.value && !dragging) {
    trailCtx.globalCompositeOperation = 'destination-out'
    trailCtx.fillStyle = 'rgba(0, 0, 0, 0.04)'
    trailCtx.fillRect(0, 0, WIDTH, HEIGHT)
    trailCtx.globalCompositeOperation = 'source-over'

    if (lastTip) {
      trailCtx.strokeStyle = '#ff7e5f'
      trailCtx.lineWidth = 1.5
      trailCtx.beginPath()
      trailCtx.moveTo(lastTip.x, lastTip.y)
      trailCtx.lineTo(bob2.x, bob2.y)
      trailCtx.stroke()
    }
    lastTip = { x: bob2.x, y: bob2.y }
  } else {
    lastTip = null
  }
}

// ─────────────────────────────────────────────────────────────
// Mouse interaction
// ─────────────────────────────────────────────────────────────

function onPointerDown(e) {
  if (!mainCanvas) return
  const rect = mainCanvas.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  const { bob1, bob2 } = getPositions()
  const d1 = Math.hypot(x - bob1.x, y - bob1.y)
  const d2 = Math.hypot(x - bob2.x, y - bob2.y)
  if (d2 <= BOB_RADIUS && d2 <= d1) dragging = 'bob2'
  else if (d1 <= BOB_RADIUS) dragging = 'bob1'
}

function onPointerMove(e) {
  if (!dragging || !mainCanvas) return
  const rect = mainCanvas.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  if (dragging === 'bob1') {
    theta1 = Math.atan2(x - ANCHOR.x, y - ANCHOR.y)
  } else {
    const { bob1 } = getPositions()
    theta2 = Math.atan2(x - bob1.x, y - bob1.y)
  }
  omega1 = 0
  omega2 = 0
}

function onPointerUp() {
  if (!dragging) return
  omega1 = 0
  omega2 = 0
  lastTip = null
  trailCtx?.clearRect(0, 0, WIDTH, HEIGHT)
  phaseSamples.length = 0
  drawCharts()
  dragging = null
}

// ─────────────────────────────────────────────────────────────
// Controls
// ─────────────────────────────────────────────────────────────

function toggleRun() {
  running.value = !running.value
  // Pause/resume only — no state clearing, graph and trail persist.
}

function reset() {
  const a = (startAngleDeg.value * Math.PI) / 180
  theta1 = a
  theta2 = a
  omega1 = 0
  omega2 = 0
  lastTip = null
  trailCtx?.clearRect(0, 0, WIDTH, HEIGHT)
  phaseSamples.length = 0
  omegaScale = 0.5
  running.value = false
  drawCharts()
}

// ─────────────────────────────────────────────────────────────
// Phase chart
// ─────────────────────────────────────────────────────────────

function projectTheta(theta) {
  return ((theta + THETA_RANGE) / (2 * THETA_RANGE)) * CHART_W
}
function projectOmega(omega) {
  const norm = (omega + omegaScale) / (2 * omegaScale)
  return CHART_H - norm * CHART_H
}

function drawCharts() {
  if (!chartCtx) return
  const ctx = chartCtx

  ctx.fillStyle = '#0b0d12'
  ctx.fillRect(0, 0, CHART_W, CHART_H)

  ctx.strokeStyle = '#374151'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.moveTo(CHART_W / 2, 0)
  ctx.lineTo(CHART_W / 2, CHART_H)
  ctx.moveTo(0, CHART_H / 2)
  ctx.lineTo(CHART_W, CHART_H / 2)
  ctx.stroke()

  ctx.strokeStyle = '#1f2937'
  ctx.beginPath()
  for (let i = 1; i < 4; i++) {
    if (i === 2) continue
    ctx.moveTo((CHART_W / 4) * i, 0)
    ctx.lineTo((CHART_W / 4) * i, CHART_H)
    ctx.moveTo(0, (CHART_H / 4) * i)
    ctx.lineTo(CHART_W, (CHART_H / 4) * i)
  }
  ctx.stroke()

  if (phaseSamples.length > 1) {
    drawTrajectory(ctx, 'theta1', 'omega1', [65, 184, 131])
    drawTrajectory(ctx, 'theta2', 'omega2', [255, 126, 95])
  }

  ctx.font = '11px system-ui, sans-serif'
  ctx.fillStyle = '#9ca3af'
  ctx.fillText('θ', 8, 16)
  ctx.save()
  ctx.translate(14, CHART_H / 2)
  ctx.rotate(-Math.PI / 2)
  ctx.textAlign = 'center'
  ctx.fillText('dθ/dt', 0, -4)
  ctx.restore()

  ctx.textAlign = 'right'
  ctx.fillStyle = '#41b883'
  ctx.fillText('■ upper', CHART_W - 56, 16)
  ctx.fillStyle = '#ff7e5f'
  ctx.fillText('■ lower', CHART_W - 8, 16)
  ctx.textAlign = 'left'
}

function drawTrajectory(ctx, thetaKey, omegaKey, rgb) {
  ctx.strokeStyle = `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`
  ctx.lineWidth = 1.2
  ctx.beginPath()
  for (let i = 0; i < phaseSamples.length; i++) {
    const s = phaseSamples[i]
    const x = projectTheta(s[thetaKey])
    const y = projectOmega(s[omegaKey])
    if (i === 0) ctx.moveTo(x, y)
    else ctx.lineTo(x, y)
  }
  ctx.stroke()
  const latest = phaseSamples[phaseSamples.length - 1]
  ctx.fillStyle = `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`
  ctx.beginPath()
  ctx.arc(
    projectTheta(latest[thetaKey]),
    projectOmega(latest[omegaKey]),
    3,
    0,
    Math.PI * 2
  )
  ctx.fill()
}

onMounted(build)
onBeforeUnmount(teardown)
</script>

<template>
  <div class="pendulum">
    <h1>Chaos Pendulum</h1>
    <div class="stage">
      <div ref="sceneRef" class="scene" :style="{ width: WIDTH + 'px', height: HEIGHT + 'px' }" />
      <aside class="sidebar">
        <div class="panel">
          <h2>Phase space</h2>
          <div class="chart">
            <canvas ref="chartRef" :width="CHART_W" :height="CHART_H" />
          </div>
          <p class="panel-note">dθ/dt vs θ — upper (green), lower (orange).</p>
        </div>
        <div class="params">
          <fieldset>
            <legend>Upper arm</legend>
            <label>Rod length <input type="number" v-model.number="rodLen1" min="20" max="200" step="5"></label>
            <label>Ball mass <input type="number" v-model.number="mass1" min="0.1" max="50" step="0.5"></label>
          </fieldset>
          <fieldset>
            <legend>Lower arm</legend>
            <label>Rod length <input type="number" v-model.number="rodLen2" min="20" max="200" step="5"></label>
            <label>Ball mass <input type="number" v-model.number="mass2" min="0.1" max="50" step="0.5"></label>
          </fieldset>
          <fieldset>
            <legend>Start angle (deg)</legend>
            <label>θ <input type="number" v-model.number="startAngleDeg" min="-180" max="180" step="1"></label>
          </fieldset>
        </div>
        <div class="controls">
          <button class="btn primary" @click="toggleRun">{{ running ? 'Pause' : 'Start' }}</button>
          <button class="btn" @click="reset">Reset</button>
        </div>
        <p class="subtle">Drag a bob to reposition. Parameter changes apply on Reset.</p>
      </aside>
    </div>
  </div>
</template>

<style scoped>
.pendulum {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  color: #e5e7eb;
  font-family: system-ui, sans-serif;
  padding: 0.75rem;
}
h1 {
  margin: 0;
  font-size: 1.25rem;
}
.stage {
  display: flex;
  gap: 1.5rem;
  align-items: flex-start;
  flex-wrap: wrap;
  justify-content: center;
}
.sidebar {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  min-width: 280px;
  max-width: 300px;
  flex-shrink: 0;
}
.scene {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  cursor: grab;
  flex-shrink: 0;
}
.scene:active {
  cursor: grabbing;
}
.panel {
  background: #11141b;
  border: 1px solid #1f2937;
  border-radius: 8px;
  padding: 0.5rem 0.6rem;
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}
.panel h2 {
  margin: 0;
  font-size: 0.85rem;
  color: #e5e7eb;
}
.chart {
  border: 1px solid #1f2937;
  border-radius: 4px;
  overflow: hidden;
  line-height: 0;
}
.chart canvas {
  display: block;
}
.panel-note {
  margin: 0;
  color: #6b7280;
  font-size: 0.7rem;
}
.params {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}
.params fieldset {
  border: 1px solid #1f2937;
  border-radius: 6px;
  padding: 0.3rem 0.6rem 0.4rem;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  background: #11141b;
  margin: 0;
}
.params legend {
  color: #9ca3af;
  font-size: 0.7rem;
  padding: 0 0.25rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.params label {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  font-size: 0.8rem;
  color: #e5e7eb;
}
.params input[type="number"] {
  width: 70px;
  background: #0b0d12;
  color: #e5e7eb;
  border: 1px solid #374151;
  border-radius: 4px;
  padding: 0.15rem 0.4rem;
  font: inherit;
  font-size: 0.8rem;
}
.params input[type="number"]:focus {
  outline: none;
  border-color: #41b883;
}
.controls {
  display: flex;
  gap: 0.4rem;
}
.btn {
  background: #1f2937;
  color: #e5e7eb;
  border: 1px solid #374151;
  padding: 0.4rem 0.75rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.85rem;
  cursor: pointer;
}
.btn:hover {
  background: #2a3645;
}
.btn.primary {
  background: #41b883;
  color: #0b0d12;
  border-color: transparent;
}
.btn.primary:hover {
  background: #34a06f;
}
.subtle {
  font-size: 0.72rem;
  color: #6b7280;
  margin: 0;
  line-height: 1.3;
}
</style>
