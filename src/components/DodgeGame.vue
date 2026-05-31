<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const sceneRef = ref(null)
const score = ref(0)
const best = ref(0)
const state = ref('idle') // 'idle' | 'playing' | 'dead'
const stage = ref(1) // 1 = double, 2 = triple
const health = ref(HEALTH_MAX)
const healthActive = ref(false)

const WIDTH = 720
const HEIGHT = 520
const BOB_RADIUS = 14
const CURSOR_RADIUS = 7
const GRAVITY = 800
const SUBSTEPS = 8
const FRAME_DT = 1 / 60
const UPGRADE_AT = 10 // seconds
const WALLS_AT = 20 // seconds
const WALL_PERIOD = 8 // seconds per in-and-out cycle
const WALL_MAX = 90 // max intrusion in px
const HEALTH_AT = 30 // seconds — pickup system activates
const HEALTH_MAX = 5 // seconds of health per pickup
const PICKUP_RADIUS = 10

const CENTER = { x: WIDTH / 2, y: HEIGHT / 2 }

// Pivots arranged as an equilateral triangle around the arena center.
const PIVOT_RADIUS = 150
const PIVOT_ANGLES = [-Math.PI / 2, Math.PI / 6, (5 * Math.PI) / 6]
function pivotAt(i) {
  return {
    ax: CENTER.x + PIVOT_RADIUS * Math.cos(PIVOT_ANGLES[i]),
    ay: CENTER.y + PIVOT_RADIUS * Math.sin(PIVOT_ANGLES[i]),
  }
}
const CONFIGS = [
  { ...pivotAt(0), L: [90, 90], m: [3, 3] },
  { ...pivotAt(1), L: [120, 110], m: [4, 4] },
  { ...pivotAt(2), L: [80, 95], m: [3, 3] },
]
const THIRD_SEGMENT = { L: 70, m: 2 }

let canvas
let ctx
let rafId = null
let cursor = { x: WIDTH / 2, y: HEIGHT - 60, active: false }
let startTime = 0
let lastFrameTime = 0
let pendulums = []
let upgraded = false
let pickup = null // { x, y } or null

// ─────────────────────────────────────────────────────────────
// N-pendulum physics (Lagrangian → mass matrix → solve → RK4).
// ─────────────────────────────────────────────────────────────

function solveLinear(A, b) {
  const n = b.length
  // Copy in place since we'll mutate.
  const M = A.map((r) => r.slice())
  const v = b.slice()
  for (let i = 0; i < n; i++) {
    // Partial pivot.
    let maxRow = i
    for (let r = i + 1; r < n; r++) {
      if (Math.abs(M[r][i]) > Math.abs(M[maxRow][i])) maxRow = r
    }
    if (maxRow !== i) {
      ;[M[i], M[maxRow]] = [M[maxRow], M[i]]
      ;[v[i], v[maxRow]] = [v[maxRow], v[i]]
    }
    const piv = M[i][i]
    for (let r = i + 1; r < n; r++) {
      const f = M[r][i] / piv
      for (let c = i; c < n; c++) M[r][c] -= f * M[i][c]
      v[r] -= f * v[i]
    }
  }
  const x = new Array(n)
  for (let i = n - 1; i >= 0; i--) {
    let s = v[i]
    for (let j = i + 1; j < n; j++) s -= M[i][j] * x[j]
    x[i] = s / M[i][i]
  }
  return x
}

function derivs(p, st) {
  const n = p.L.length
  const theta = st.slice(0, n)
  const omega = st.slice(n)

  // S[i] = sum of masses at indices >= i.
  const S = new Array(n)
  S[n - 1] = p.m[n - 1]
  for (let i = n - 2; i >= 0; i--) S[i] = S[i + 1] + p.m[i]

  const M = []
  for (let i = 0; i < n; i++) {
    const row = new Array(n)
    for (let j = 0; j < n; j++) {
      row[j] =
        p.L[i] * p.L[j] * Math.cos(theta[i] - theta[j]) * S[Math.max(i, j)]
    }
    M.push(row)
  }

  const b = new Array(n)
  for (let i = 0; i < n; i++) {
    let bi = -GRAVITY * p.L[i] * Math.sin(theta[i]) * S[i]
    for (let j = 0; j < n; j++) {
      if (i === j) continue
      bi -=
        p.L[i] *
        p.L[j] *
        Math.sin(theta[i] - theta[j]) *
        S[Math.max(i, j)] *
        omega[j] *
        omega[j]
    }
    b[i] = bi
  }

  const alpha = solveLinear(M, b)
  return [...omega, ...alpha]
}

function rk4Step(p, dt) {
  const n = p.L.length
  const s0 = [...p.theta, ...p.omega]
  const k1 = derivs(p, s0)
  const s2 = s0.map((x, i) => x + 0.5 * dt * k1[i])
  const k2 = derivs(p, s2)
  const s3 = s0.map((x, i) => x + 0.5 * dt * k2[i])
  const k3 = derivs(p, s3)
  const s4 = s0.map((x, i) => x + dt * k3[i])
  const k4 = derivs(p, s4)
  for (let i = 0; i < 2 * n; i++) {
    const inc = (dt / 6) * (k1[i] + 2 * k2[i] + 2 * k3[i] + k4[i])
    if (i < n) p.theta[i] = s0[i] + inc
    else p.omega[i - n] = s0[i] + inc
  }
}

function getBobs(p) {
  const out = []
  let x = p.ax
  let y = p.ay
  for (let i = 0; i < p.L.length; i++) {
    x += p.L[i] * Math.sin(p.theta[i])
    y += p.L[i] * Math.cos(p.theta[i])
    out.push({ x, y })
  }
  return out
}

function collides(p) {
  const r = BOB_RADIUS + CURSOR_RADIUS
  for (const b of getBobs(p)) {
    if (Math.hypot(cursor.x - b.x, cursor.y - b.y) < r) return true
  }
  return false
}

// ─────────────────────────────────────────────────────────────
// Game lifecycle
// ─────────────────────────────────────────────────────────────

function makePendulum(cfg, seed) {
  return {
    ax: cfg.ax,
    ay: cfg.ay,
    L: cfg.L.slice(),
    m: cfg.m.slice(),
    // Spread initial angles around the unit circle so they don't all start aligned.
    theta: [seed * 2.1 + 0.4, seed * 1.6 - 1.2],
    omega: [0, 0],
  }
}

const PICKUP_SPEED = 90

function spawnPickup() {
  const margin = 30
  const xMin = WALL_MAX + margin
  const xMax = WIDTH - WALL_MAX - margin
  const yMin = WALL_MAX + margin
  const yMax = HEIGHT - WALL_MAX - margin
  let pos = null
  for (let tries = 0; tries < 80; tries++) {
    const x = xMin + Math.random() * (xMax - xMin)
    const y = yMin + Math.random() * (yMax - yMin)
    let ok = true
    for (const cfg of CONFIGS) {
      const reach = cfg.L.reduce((a, b) => a + b, 0) + (upgraded ? THIRD_SEGMENT.L : 0) + 35
      if (Math.hypot(x - cfg.ax, y - cfg.ay) < reach) { ok = false; break }
    }
    if (ok && Math.hypot(x - cursor.x, y - cursor.y) < 80) ok = false
    if (ok) { pos = { x, y }; break }
  }
  if (!pos) pos = { x: WALL_MAX + margin + 10, y: HEIGHT - WALL_MAX - margin - 10 }
  // Random direction.
  const angle = Math.random() * Math.PI * 2
  return {
    x: pos.x,
    y: pos.y,
    vx: Math.cos(angle) * PICKUP_SPEED,
    vy: Math.sin(angle) * PICKUP_SPEED,
  }
}

function updatePickup(dt, intrusion) {
  if (!pickup) return
  const margin = PICKUP_RADIUS + 2
  pickup.x += pickup.vx * dt
  pickup.y += pickup.vy * dt
  // Bounce off the playable boundary (which contracts as walls intrude).
  if (pickup.x - margin < intrusion) {
    pickup.x = intrusion + margin
    pickup.vx = Math.abs(pickup.vx)
  } else if (pickup.x + margin > WIDTH - intrusion) {
    pickup.x = WIDTH - intrusion - margin
    pickup.vx = -Math.abs(pickup.vx)
  }
  if (pickup.y - margin < intrusion) {
    pickup.y = intrusion + margin
    pickup.vy = Math.abs(pickup.vy)
  } else if (pickup.y + margin > HEIGHT - intrusion) {
    pickup.y = HEIGHT - intrusion - margin
    pickup.vy = -Math.abs(pickup.vy)
  }
}

function wallIntrusion() {
  if (state.value !== 'playing' || score.value < WALLS_AT) return 0
  const t = score.value - WALLS_AT
  return WALL_MAX * 0.5 * (1 - Math.cos((2 * Math.PI * t) / WALL_PERIOD))
}

function collidesWalls(intrusion) {
  if (intrusion <= 0) return false
  const r = CURSOR_RADIUS
  return (
    cursor.x - r < intrusion ||
    cursor.x + r > WIDTH - intrusion ||
    cursor.y - r < intrusion ||
    cursor.y + r > HEIGHT - intrusion
  )
}

function initPendulums() {
  pendulums = CONFIGS.map((cfg, i) => makePendulum(cfg, i + 1))
  upgraded = false
  stage.value = 1
}

function upgradeToTriple() {
  for (const p of pendulums) {
    p.L.push(THIRD_SEGMENT.L)
    p.m.push(THIRD_SEGMENT.m)
    // Continue in line with the last segment, no initial velocity — chaos takes over fast.
    p.theta.push(p.theta[p.theta.length - 1])
    p.omega.push(0)
  }
  upgraded = true
  stage.value = 2
}

function startGame() {
  initPendulums()
  startTime = performance.now()
  lastFrameTime = startTime
  score.value = 0
  health.value = HEALTH_MAX
  healthActive.value = false
  pickup = null
  state.value = 'playing'
}

function loop() {
  const now = performance.now()
  const frameDt = (now - lastFrameTime) / 1000
  lastFrameTime = now

  if (state.value === 'playing') {
    const dt = FRAME_DT / SUBSTEPS
    for (let i = 0; i < SUBSTEPS; i++) {
      for (const p of pendulums) rk4Step(p, dt)
    }
    score.value = (now - startTime) / 1000

    if (!upgraded && score.value >= UPGRADE_AT) upgradeToTriple()

    // Activate the pickup/health system at 40s.
    if (!healthActive.value && score.value >= HEALTH_AT) {
      healthActive.value = true
      health.value = HEALTH_MAX
      pickup = spawnPickup()
    }

    const intrusion = wallIntrusion()

    // Drain health while the system is active.
    if (healthActive.value) {
      health.value = Math.max(0, health.value - frameDt)
      updatePickup(frameDt, intrusion)
      // Pickup collected?
      if (pickup) {
        const r = PICKUP_RADIUS + CURSOR_RADIUS
        if (Math.hypot(cursor.x - pickup.x, cursor.y - pickup.y) < r) {
          health.value = HEALTH_MAX
          pickup = spawnPickup()
        }
      }
    }
    let dead = collidesWalls(intrusion)
    if (!dead) {
      for (const p of pendulums) {
        if (collides(p)) { dead = true; break }
      }
    }
    if (!dead && healthActive.value && health.value <= 0) dead = true
    if (dead) {
      state.value = 'dead'
      if (score.value > best.value) best.value = score.value
    }
  }
  render()
  rafId = requestAnimationFrame(loop)
}

// ─────────────────────────────────────────────────────────────
// Rendering
// ─────────────────────────────────────────────────────────────

function render() {
  ctx.fillStyle = '#0b0d12'
  ctx.fillRect(0, 0, WIDTH, HEIGHT)

  // Arena border.
  ctx.strokeStyle = stage.value === 2 ? '#ff7e5f' : '#41b883'
  ctx.lineWidth = 3
  ctx.strokeRect(1.5, 1.5, WIDTH - 3, HEIGHT - 3)

  // Killing walls (closing/opening from each edge).
  const intrusion = wallIntrusion()
  if (intrusion > 0) {
    ctx.fillStyle = 'rgba(239, 68, 68, 0.85)'
    ctx.fillRect(0, 0, intrusion, HEIGHT)
    ctx.fillRect(WIDTH - intrusion, 0, intrusion, HEIGHT)
    ctx.fillRect(0, 0, WIDTH, intrusion)
    ctx.fillRect(0, HEIGHT - intrusion, WIDTH, intrusion)
    // Hatched inner edge for visual menace.
    ctx.strokeStyle = '#7f1d1d'
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.moveTo(intrusion + 0.5, 0)
    ctx.lineTo(intrusion + 0.5, HEIGHT)
    ctx.moveTo(WIDTH - intrusion - 0.5, 0)
    ctx.lineTo(WIDTH - intrusion - 0.5, HEIGHT)
    ctx.moveTo(0, intrusion + 0.5)
    ctx.lineTo(WIDTH, intrusion + 0.5)
    ctx.moveTo(0, HEIGHT - intrusion - 0.5)
    ctx.lineTo(WIDTH, HEIGHT - intrusion - 0.5)
    ctx.stroke()
  }

  const bobColors = ['#41b883', '#ff7e5f', '#fbbf24']

  for (const p of pendulums) {
    const bobs = getBobs(p)
    // Rods
    ctx.strokeStyle = '#374151'
    ctx.lineWidth = 2
    ctx.beginPath()
    ctx.moveTo(p.ax, p.ay)
    for (const b of bobs) ctx.lineTo(b.x, b.y)
    ctx.stroke()
    // Anchor pin
    ctx.fillStyle = '#888'
    ctx.beginPath()
    ctx.arc(p.ax, p.ay, 5, 0, Math.PI * 2)
    ctx.fill()
    // Bobs
    for (let i = 0; i < bobs.length; i++) {
      ctx.fillStyle = bobColors[i % bobColors.length]
      ctx.beginPath()
      ctx.arc(bobs[i].x, bobs[i].y, BOB_RADIUS, 0, Math.PI * 2)
      ctx.fill()
    }
  }

  // Pickup (drawn before cursor so cursor sits on top).
  if (pickup && state.value === 'playing') {
    const pulse = 0.6 + 0.4 * Math.sin(performance.now() / 180)
    ctx.fillStyle = `rgba(251, 191, 36, ${0.18 * pulse})`
    ctx.beginPath()
    ctx.arc(pickup.x, pickup.y, PICKUP_RADIUS + 12, 0, Math.PI * 2)
    ctx.fill()
    ctx.fillStyle = '#fbbf24'
    ctx.beginPath()
    ctx.arc(pickup.x, pickup.y, PICKUP_RADIUS, 0, Math.PI * 2)
    ctx.fill()
    ctx.fillStyle = '#0b0d12'
    ctx.font = 'bold 12px ui-monospace, monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('E', pickup.x, pickup.y + 1)
    ctx.textAlign = 'left'
    ctx.textBaseline = 'alphabetic'
  }

  // Health bar inside the canvas, shows up only after activation.
  if (healthActive.value) {
    const barW = 220
    const barH = 12
    const barX = (WIDTH - barW) / 2
    const barY = HEIGHT - 28
    ctx.fillStyle = '#11141b'
    ctx.fillRect(barX - 2, barY - 2, barW + 4, barH + 4)
    ctx.fillStyle = '#1f2937'
    ctx.fillRect(barX, barY, barW, barH)
    const ratio = health.value / HEALTH_MAX
    ctx.fillStyle = ratio > 0.4 ? '#41b883' : ratio > 0.2 ? '#fbbf24' : '#ef4444'
    ctx.fillRect(barX, barY, barW * ratio, barH)
    ctx.fillStyle = '#9ca3af'
    ctx.font = '11px system-ui, sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText(`HEALTH ${health.value.toFixed(1)}s — collect E`, WIDTH / 2, barY - 6)
    ctx.textAlign = 'left'
  }

  // Cursor
  if (cursor.active || state.value !== 'idle') {
    ctx.fillStyle = state.value === 'dead' ? '#ef4444' : '#e5e7eb'
    ctx.beginPath()
    ctx.arc(cursor.x, cursor.y, CURSOR_RADIUS, 0, Math.PI * 2)
    ctx.fill()
    ctx.strokeStyle = '#0b0d12'
    ctx.lineWidth = 2
    ctx.stroke()
  }

  // HUD overlay
  if (state.value === 'idle') {
    drawCenterText('Move into the arena to start', '#e5e7eb', 22)
    drawCenterText('Triple at 10s · Walls at 20s', '#9ca3af', 14, 28)
  } else if (state.value === 'playing') {
    if (!upgraded && UPGRADE_AT - score.value < 3 && UPGRADE_AT - score.value > 0) {
      drawCenterText(
        `Triple in ${(UPGRADE_AT - score.value).toFixed(1)}s…`,
        '#fbbf24',
        18,
        -HEIGHT / 2 + 50
      )
    }
    if (score.value < WALLS_AT && WALLS_AT - score.value < 3) {
      drawCenterText(
        `Walls in ${(WALLS_AT - score.value).toFixed(1)}s…`,
        '#ef4444',
        18,
        -HEIGHT / 2 + 50
      )
    }
    if (!healthActive.value && HEALTH_AT - score.value < 3 && HEALTH_AT - score.value > 0) {
      drawCenterText(
        `Pickups in ${(HEALTH_AT - score.value).toFixed(1)}s…`,
        '#fbbf24',
        18,
        -HEIGHT / 2 + 50
      )
    }
  } else if (state.value === 'dead') {
    drawCenterText(`You lasted ${score.value.toFixed(2)}s`, '#ef4444', 28, -16)
    drawCenterText('Click to retry', '#9ca3af', 16, 20)
  }
}

function drawCenterText(text, color, size, yOffset = 0) {
  ctx.fillStyle = color
  ctx.font = `${size}px system-ui, sans-serif`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(text, WIDTH / 2, HEIGHT / 2 + yOffset)
  ctx.textAlign = 'left'
  ctx.textBaseline = 'alphabetic'
}

// ─────────────────────────────────────────────────────────────
// Input + lifecycle
// ─────────────────────────────────────────────────────────────

function onPointerMove(e) {
  const rect = canvas.getBoundingClientRect()
  cursor.x = e.clientX - rect.left
  cursor.y = e.clientY - rect.top
  cursor.active = true
  if (state.value === 'idle') startGame()
}
function onPointerLeave() {
  cursor.active = false
}
function onPointerDown() {
  if (state.value === 'dead') startGame()
}

function build() {
  canvas = document.createElement('canvas')
  canvas.width = WIDTH
  canvas.height = HEIGHT
  canvas.style.display = 'block'
  canvas.style.background = '#0b0d12'
  canvas.style.cursor = 'none'
  sceneRef.value.appendChild(canvas)
  ctx = canvas.getContext('2d')

  canvas.addEventListener('pointermove', onPointerMove)
  canvas.addEventListener('pointerleave', onPointerLeave)
  canvas.addEventListener('pointerdown', onPointerDown)

  initPendulums()
  loop()
}

function teardown() {
  if (rafId !== null) cancelAnimationFrame(rafId)
  rafId = null
  if (canvas) {
    canvas.removeEventListener('pointermove', onPointerMove)
    canvas.removeEventListener('pointerleave', onPointerLeave)
    canvas.removeEventListener('pointerdown', onPointerDown)
    canvas.remove()
    canvas = null
    ctx = null
  }
}

onMounted(build)
onBeforeUnmount(teardown)
</script>

<template>
  <div class="game">
    <h1>Dodge the Pendulums</h1>
    <div class="hud">
      <div>Time: <span class="num">{{ score.toFixed(2) }}s</span></div>
      <div>Best: <span class="num">{{ best.toFixed(2) }}s</span></div>
      <div>Stage: <span class="num">{{ stage === 1 ? 'Double' : 'Triple' }}</span></div>
    </div>
    <div ref="sceneRef" class="scene" :style="{ width: WIDTH + 'px', height: HEIGHT + 'px' }" />
    <p class="hint">Move into the arena to start. Avoid bobs and walls. 10s: triple pendulums. 20s: walls breathe in and out. 30s: collect an <strong style="color:#fbbf24">E</strong> every 5s or your health drains.</p>
  </div>
</template>

<style scoped>
.game {
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
.hud {
  display: flex;
  gap: 1.5rem;
  font-size: 0.95rem;
  color: #9ca3af;
}
.num {
  color: #e5e7eb;
  font-family: ui-monospace, monospace;
}
.scene {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}
.hint {
  margin: 0;
  color: #6b7280;
  font-size: 0.8rem;
  max-width: 720px;
  text-align: center;
}
</style>
