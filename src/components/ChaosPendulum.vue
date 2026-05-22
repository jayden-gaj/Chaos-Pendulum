<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'
import Matter from 'matter-js'

const sceneRef = ref(null)

let engine
let render
let runner
let trailCanvas
let trailCtx
let lastTip = null

const WIDTH = 800
const HEIGHT = 600
const ANCHOR = { x: WIDTH / 2, y: 120 }
const ROD_LENGTH = 160
const BOB_RADIUS = 18

function build() {
  const { Engine, Render, Runner, Bodies, Body, Composite, Constraint, Events } = Matter

  engine = Engine.create()
  engine.gravity.y = 1

  render = Render.create({
    element: sceneRef.value,
    engine,
    options: {
      width: WIDTH,
      height: HEIGHT,
      wireframes: false,
      background: '#0b0d12',
    },
  })

  const anchor = Bodies.circle(ANCHOR.x, ANCHOR.y, 6, {
    isStatic: true,
    render: { fillStyle: '#888' },
  })

  const bob1 = Bodies.circle(ANCHOR.x + ROD_LENGTH, ANCHOR.y, BOB_RADIUS, {
    density: 0.004,
    frictionAir: 0,
    render: { fillStyle: '#41b883' },
  })

  const bob2 = Bodies.circle(ANCHOR.x + ROD_LENGTH * 2, ANCHOR.y, BOB_RADIUS, {
    density: 0.004,
    frictionAir: 0,
    render: { fillStyle: '#ff7e5f' },
  })

  const rod1 = Constraint.create({
    bodyA: anchor,
    bodyB: bob1,
    length: ROD_LENGTH,
    stiffness: 1,
    damping: 0,
    render: { strokeStyle: '#e5e7eb', lineWidth: 2 },
  })

  const rod2 = Constraint.create({
    bodyA: bob1,
    bodyB: bob2,
    length: ROD_LENGTH,
    stiffness: 1,
    damping: 0,
    render: { strokeStyle: '#e5e7eb', lineWidth: 2 },
  })

  Composite.add(engine.world, [anchor, bob1, bob2, rod1, rod2])

  // Kick it off the equilibrium so chaos shows up.
  Body.setAngularVelocity(bob1, 0.05)

  // Trail layer rendered on top of Matter.js's canvas.
  trailCanvas = document.createElement('canvas')
  trailCanvas.width = WIDTH
  trailCanvas.height = HEIGHT
  trailCanvas.style.position = 'absolute'
  trailCanvas.style.inset = '0'
  trailCanvas.style.pointerEvents = 'none'
  sceneRef.value.style.position = 'relative'
  sceneRef.value.appendChild(trailCanvas)
  trailCtx = trailCanvas.getContext('2d')

  Events.on(render, 'afterRender', () => {
    // Fade prior trail.
    trailCtx.fillStyle = 'rgba(11, 13, 18, 0.04)'
    trailCtx.fillRect(0, 0, WIDTH, HEIGHT)

    if (lastTip) {
      trailCtx.strokeStyle = '#ff7e5f'
      trailCtx.lineWidth = 1.5
      trailCtx.beginPath()
      trailCtx.moveTo(lastTip.x, lastTip.y)
      trailCtx.lineTo(bob2.position.x, bob2.position.y)
      trailCtx.stroke()
    }
    lastTip = { x: bob2.position.x, y: bob2.position.y }
  })

  Render.run(render)
  runner = Runner.create()
  Runner.run(runner, engine)
}

function reset() {
  teardown()
  lastTip = null
  build()
}

function teardown() {
  const { Render, Runner, Composite } = Matter
  if (render) {
    Render.stop(render)
    render.canvas?.remove()
    render.textures = {}
  }
  if (runner) Runner.stop(runner)
  if (engine) Composite.clear(engine.world, false)
  if (trailCanvas) trailCanvas.remove()
}

onMounted(build)
onBeforeUnmount(teardown)
</script>

<template>
  <div class="pendulum">
    <h1>Chaos Pendulum</h1>
    <p class="hint">A double pendulum — tiny changes in starting conditions produce wildly different paths.</p>
    <div ref="sceneRef" class="scene" />
    <button class="reset" @click="reset">Reset</button>
  </div>
</template>

<style scoped>
.pendulum {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  color: #e5e7eb;
  font-family: system-ui, sans-serif;
  padding: 1.5rem;
}
h1 {
  margin: 0;
  font-size: 1.5rem;
}
.hint {
  margin: 0;
  color: #9ca3af;
  font-size: 0.9rem;
}
.scene {
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}
.reset {
  background: #41b883;
  color: #0b0d12;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}
.reset:hover {
  background: #34a06f;
}
</style>
