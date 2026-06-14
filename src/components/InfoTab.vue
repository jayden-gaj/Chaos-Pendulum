<script setup>
import { onBeforeUnmount, onMounted, ref, watch } from 'vue'

function typesetMath() {
  if (typeof window === 'undefined' || !window.MathJax?.typesetPromise) return

  const target = infoPanelRef.value || document.querySelector('.info-panel')
  if (!target) return

  window.MathJax.typesetPromise([target]).catch(() => {})
}

const infoPanelRef = ref(null)
const canvasRef = ref(null)
const numPendulums = ref(2)
const angleSpread = ref(.001)
const isRunning = ref(false)

const WIDTH = 360
const HEIGHT = 280
const DPR = Math.min(window.devicePixelRatio || 1, 2)
const GRAVITY = 820
const ROD_LEN_1 = 78
const ROD_LEN_2 = 62
const BOB_RADIUS = 7

let ctx = null
let rafId = null
let pendulums = []

function derivs(state) {
  const [theta1, omega1, theta2, omega2] = state
  const m1 = 4
  const m2 = 4
  const L1 = ROD_LEN_1
  const L2 = ROD_LEN_2
  const g = GRAVITY
  const d = theta1 - theta2
  const denom = 2 * m1 + m2 - m2 * Math.cos(2 * theta1 - 2 * theta2)

  const a1 =
    (-g * (2 * m1 + m2) * Math.sin(theta1) -
      m2 * g * Math.sin(theta1 - 2 * theta2) -
      2 * Math.sin(d) * m2 * (omega2 * omega2 * L2 + omega1 * omega1 * L1 * Math.cos(d))) /
    (L1 * denom)

  const a2 =
    (2 *
      Math.sin(d) *
      (omega1 * omega1 * L1 * (m1 + m2) +
        g * (m1 + m2) * Math.cos(theta1) +
        omega2 * omega2 * L2 * m2 * Math.cos(d))) /
    (L2 * denom)

  return [omega1, a1, omega2, a2]
}

function rk4(state, dt) {
  const k1 = derivs(state)
  const s2 = state.map((value, i) => value + 0.5 * dt * k1[i])
  const k2 = derivs(s2)
  const s3 = state.map((value, i) => value + 0.5 * dt * k2[i])
  const k3 = derivs(s3)
  const s4 = state.map((value, i) => value + dt * k3[i])
  const k4 = derivs(s4)

  return state.map((value, i) => value + (dt / 6) * (k1[i] + 2 * k2[i] + 2 * k3[i] + k4[i]))
}

function createPendulums() {
  const count = Math.max(1, Math.min(9, numPendulums.value))
  const startAngle = (90 * Math.PI) / 180
  const spread = Math.max(0, angleSpread.value)
  const center = (count - 1) / 2

  pendulums = Array.from({ length: count }, (_, index) => {
    const phase = (index - center) * (spread * Math.PI) / 180

    return {
      theta1: startAngle + phase,
      omega1: 0,
      theta2: startAngle + phase * 0.85,
      omega2: 0,
      hue: 150 + index * 18,
    }
  })
}

function stepPendulums(dt) {
  pendulums = pendulums.map((item) => {
    const next = rk4([item.theta1, item.omega1, item.theta2, item.omega2], dt)
    return {
      ...item,
      theta1: next[0],
      omega1: next[1],
      theta2: next[2],
      omega2: next[3],
    }
  })
}

function drawPendulum(item) {
  const L1 = ROD_LEN_1 * 0.92
  const L2 = ROD_LEN_2 * 0.92
  const anchorX = WIDTH / 2
  const anchorY = 46

  const bob1 = {
    x: anchorX + L1 * Math.sin(item.theta1),
    y: anchorY + L1 * Math.cos(item.theta1),
  }

  const bob2 = {
    x: bob1.x + L2 * Math.sin(item.theta2),
    y: bob1.y + L2 * Math.cos(item.theta2),
  }

  ctx.strokeStyle = `hsla(${item.hue}, 85%, 68%, 0.35)`
  ctx.lineWidth = 1.5
  ctx.beginPath()
  ctx.moveTo(anchorX, anchorY)
  ctx.lineTo(bob1.x, bob1.y)
  ctx.lineTo(bob2.x, bob2.y)
  ctx.stroke()

  ctx.fillStyle = `hsla(${item.hue}, 70%, 62%, 0.95)`
  ctx.beginPath()
  ctx.arc(anchorX, anchorY, 3.2, 0, Math.PI * 2)
  ctx.fill()

  ctx.beginPath()
  ctx.arc(bob1.x, bob1.y, BOB_RADIUS, 0, Math.PI * 2)
  ctx.fill()

  ctx.fillStyle = `hsla(${item.hue + 15}, 85%, 72%, 0.95)`
  ctx.beginPath()
  ctx.arc(bob2.x, bob2.y, BOB_RADIUS, 0, Math.PI * 2)
  ctx.fill()
}

function render() {
  if (!ctx) return

  ctx.clearRect(0, 0, WIDTH, HEIGHT)
  ctx.fillStyle = '#0b0d12'
  ctx.fillRect(0, 0, WIDTH, HEIGHT)

  ctx.strokeStyle = 'rgba(148, 163, 184, 0.18)'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.moveTo(0, HEIGHT * 0.72)
  ctx.lineTo(WIDTH, HEIGHT * 0.72)
  ctx.stroke()

  pendulums.forEach((item) => drawPendulum(item))
}

function loop() {
  if (isRunning.value) {
    stepPendulums(1 / 60)
  }
  render()
  rafId = window.requestAnimationFrame(loop)
}

function togglePreview() {
  isRunning.value = !isRunning.value
}

function resetPreview() {
  isRunning.value = true
  createPendulums()
  render()
}

onMounted(() => {
  requestAnimationFrame(() => typesetMath())
  window.addEventListener('load', typesetMath)

  const canvas = canvasRef.value
  ctx = canvas?.getContext('2d')
  if (!ctx || !canvas) return

  canvas.width = Math.floor(WIDTH * DPR)
  canvas.height = Math.floor(HEIGHT * DPR)
  canvas.style.width = `${WIDTH}px`
  canvas.style.height = `${HEIGHT}px`
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0)
  ctx.imageSmoothingEnabled = true

  createPendulums()
  render()
  rafId = window.requestAnimationFrame(loop)
})

onBeforeUnmount(() => {
  window.removeEventListener('load', typesetMath)
  if (rafId) window.cancelAnimationFrame(rafId)
  rafId = null
  ctx = null
})

watch([numPendulums, angleSpread], () => {
  createPendulums()
  render()
})
</script>

<template>
  <section ref="infoPanelRef" class="info-panel">
    <h2 style="font-size: 1.8rem;">Physics Behind the Simulation</h2>

    
    <!-- § 5 -->
    <div class="section-block">
      <h3>Chaos Theory</h3>
      <p>
        The double pendulum is one of the simplest examples of a chaotic system. Even though
        the equations are perfectly deterministic, two trajectories with nearly identical starting
        angles diverge exponentially fast. This is the pinnacle of chaos: sensitivity to initial conditions.
      </p>
      <div class="mini-sim-card">
      <div class="mini-sim-copy">
        <h3 style="font-size: 1.2rem;">Mini Simulation</h3>
        <p>
          This shows several double-pendulum copies on top of one another with a very small
          offset so you can see the chaotic motion.
        </p>
      </div>
      <div class="mini-sim-grid">
        <canvas ref="canvasRef" :width="WIDTH" :height="HEIGHT" class="mini-canvas" />
        <div class="mini-controls">
          <label>
            <span>Angle spread (°)</span>
            <strong>{{ angleSpread.toFixed(3) }}°</strong>
            <input v-model.number="angleSpread" type="range" min="0.001" max="1" step="0.001" />
          </label>
          <label>
            <span>Pendulums</span>
            <strong>{{ numPendulums }}</strong>
            <input v-model.number="numPendulums" type="range" min="1" max="9" step="1" />
          </label>
          <div class="mini-actions">
            <button type="button" class="mini-reset" @click="togglePreview">{{ isRunning ? 'Stop' : 'Start' }}</button>
            <button type="button" class="mini-reset" @click="resetPreview">Reset view</button>
          </div>
        </div>
      </div>
      </div>
      <p>
        This divergence rate is quantified by the Lyapunov
        exponent λ:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$|\delta y(t)| \approx |\delta y_0| e^{\lambda t}$$</span>
      </div>
      
      <p>
        δy₀ is the initial separation between two trajectories. Because λ &gt; 0, tiny
        measurement errors grow without bound. This causes long-term prediction to be fundamentally impossible
        regardless of computing power. 
      </p>
      <p class="callout">
        Chaos does not mean randomness. The system is still fully deterministic: given exact initial
        conditions, the trajectory is perfectly defined. Chaos means sensitivity to initial conditions.
        In reality, we can never know positions infinitely precisely, so long-term behaviour becomes
        unpredictable.
      </p>
      
    </div>

    <!-- § 1 -->
    <div class="section-block">
      <h3>1 — System Setup &amp; Coordinates</h3>
      <p>
        A double pendulum consists of two rigid, massless rods. The first rod of length L₁ has a point
        mass m₁ at its tip; a second rod of length L₂ with mass m₂ hangs from that tip.
      </p>
      <p>
        We describe the system using two generalised coordinates: θ₁ and θ₂, the angles each rod makes
        with the vertical. The Cartesian positions of each mass follow directly:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$x_1 = L_1 \sin\theta_1$$</span>
        <span class="mathjax">$$y_1 = -L_1 \cos\theta_1$$</span>
        <span class="mathjax">$$x_2 = x_1 + L_2 \sin\theta_2$$</span>
        <span class="mathjax">$$y_2 = y_1 - L_2 \cos\theta_2$$</span>
      </div>
      
    </div>

    <!-- § 2 -->
    <div class="section-block">
      <h3>2 — The Lagrangian</h3>
      <p>
        Instead of resolving forces like Newton's approach, Lagrangian mechanics builds equations from
        energy alone. This is especially powerful for constrained systems like pendulums, where
        constraint forces (rod tension) do no work and drop out automatically.
      </p>

      
      <div class="eq-block">
        <span class="mathjax">$$\mathcal{L} = T - V$$</span>
      </div>

      <table class="var-table">
        <thead><tr><th>Symbol</th><th>Meaning</th><th>Units</th></tr></thead>
        <tbody>
          
          <tr><td>T</td><td>Total kinetic energy</td><td>J</td></tr>
          <tr><td>V</td><td>Total potential energy</td><td>J</td></tr>
          <tr><td>ℒ</td><td>Lagrangian</td><td>J</td></tr>
        </tbody>
      </table>
      <p style="font-size: .1em;"> &nbsp;</p>
      <p>
        Differentiating the position equations gives velocities. The first mass has only one velocity component, but the second mass's velocity depends on both θ₁ and θ₂:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$v_1^2 = L_1^2 \dot{\theta}_1^2$$</span>
      </div>
      <div class="eq-block">
        <span class="mathjax">$$v_2^2 = L_1^2 \dot{\theta}_1^2 + L_2^2 \dot{\theta}_2^2 + 2L_1L_2 \dot{\theta}_1\dot{\theta}_2 \cos(\theta_1-\theta_2)$$</span>
      </div>
      <p>
        After some algebra, the total kinetic energy for both masses is:
      </p>
  
      
      <div class="eq-block">
        <span class="mathjax">$$T = \tfrac12 (m_1+m_2)L_1^2 \dot{\theta}_1^2 + \tfrac12 m_2L_2^2 \dot{\theta}_2^2 + m_2L_1L_2 \dot{\theta}_1\dot{\theta}_2 \cos(\theta_1-\theta_2)$$</span>
      </div>
      <p>Taking the pivot as the reference height, total potential energy:</p>
      <div class="eq-block">
        <span class="mathjax">$$V = -(m_1+m_2)gL_1 \cos\theta_1 - m_2gL_2 \cos\theta_2$$</span>
      </div>


      <p>The Lagrangian ℒ = T − V:</p>
      <div class="eq-block">
        <span class="mathjax">$$\mathcal{L} = \tfrac12(m_1+m_2)L_1^2 \dot{\theta}_1^2 + \tfrac12 m_2L_2^2 \dot{\theta}_2^2+ m_2L_1L_2 \dot{\theta}_1\dot{\theta}_2 \cos(\theta_1-\theta_2) + (m_1+m_2)gL_1 \cos\theta_1 + m_2gL_2 \cos\theta_2$$</span>
      </div>
    </div>

    <!-- § 3 -->
    <div class="section-block">
      <h3>3 — Equations of Motion</h3>
      <p>The Euler–Lagrange equation is the foundation of Lagrangian mechanics. It uses the system's energy to derive the equations of motion, allowing us to calculate the angular accelerations of the double pendulum at every moment in time.</p>
      <p>The equation is applied separately for each generalised coordinate qᵢ (in our case that is θ₁ and θ₂):</p>
      <div class="eq-block">
        <span class="mathjax">$$\frac{d}{dt}\left(\frac{\partial \mathcal{L}}{\partial \dot q_i}\right) - \frac{\partial \mathcal{L}}{\partial q_i} = 0$$</span>
      </div>
      <p class="callout">
        A partial derivative is just a derivative where you change one variable at a time and pretend all the others are constant.
      </p>
      <p>
        Applying this for θ₁ and θ₂ and solving for the angular accelerations gives two coupled,
        nonlinear ODEs:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$\ddot{\theta}_1 = \frac{-g(2m_1+m_2)\sin\theta_1 - m_2g\sin(\theta_1-2\theta_2) - 2\sin(\theta_1-\theta_2)\,m_2(L_2\dot{\theta}_2^2 + L_1\dot{\theta}_1^2\cos(\theta_1-\theta_2))}{L_1(2m_1+m_2-m_2\cos(2\theta_1-2\theta_2))}$$</span>
      </div>
      <div class="eq-block">
        <span class="mathjax">$$\ddot{\theta}_2 = \frac{2\sin(\theta_1-\theta_2)\Big((m_1+m_2)(L_1\dot{\theta}_1^2 + g\cos\theta_1) + L_2m_2\dot{\theta}_2^2\cos(\theta_1-\theta_2)\Big)}{L_2(2m_1+m_2-m_2\cos(2\theta_1-2\theta_2))}$$</span>
      </div>
      <p>
        These equations are coupled (θ̈₁ depends on θ₂ and θ̇₂, and vice versa) and nonlinear
        (products and trig functions of the angles appear throughout). No closed-form analytical
        solution exists, so we must integrate them numerically.
      </p>
      <div class="eq-block">
        <span class="mathjax">$$\ddot{\theta}_1 = f_1\!\left(\theta_1,\theta_2,\dot{\theta}_1,\dot{\theta}_2\right)$$</span>
        <span class="mathjax">$$\ddot{\theta}_2 = f_2\!\left(\theta_1,\theta_2,\dot{\theta}_1,\dot{\theta}_2\right)$$</span>
      </div>
    </div>

    <!-- § 4 -->
    <div class="section-block">
      <h3>4 — Numerical Integration</h3>
      <p>
        Numerical integration turns the continuous equations of motion into a sequence of small
        updates. For the double pendulum, the two second-order ODEs (Ordinary Differential Equations) are rewritten as four first-order
        ODEs by introducing the angular velocities as extra state variables.
      </p>

      <div class="eq-block">
        <span class="mathjax">$$y =
\begin{bmatrix}
\theta_1 \\
\theta_2 \\
\omega_1 \\
\omega_2
\end{bmatrix},
\qquad \omega_i = \dot{\theta}_i$$</span>
      </div>

      <p>
        With this definition, the system becomes \(\dot y = f(y)\), where the derivative returns
        \([\omega_1,\omega_2,\alpha_1,\alpha_2]\). The accelerations \(\alpha_i = \ddot{\theta}_i\)
        come directly from the coupled expressions in Section 3.
      </p>

      <p class="callout">
        An ODE is an equation that relates a function to one or more of its derivatives. A state vector
        stores everything the future evolution depends on at a given instant: the two angles and the
        two angular velocities.
      </p>

      <p>
        <strong>Explicit Euler</strong> is the simplest update rule. Given a state yₙ at time tₙ and a
        timestep h, it advances the whole state in one forward step:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$y_{n+1} = y_n + h\,f(y_n)$$</span>
      </div>
      <p>
        In component form this means the next angles and angular velocities are estimated from the
        current derivative. This is easy to implement, but for a chaotic system such as the double
        pendulum it drifts quickly and tends to gain energy over long runs.
      </p>

      <p>
        <strong>Euler–Cromer</strong> (also called semi-implicit Euler) improves the stability of the
        simple method by updating the velocity first and then using that improved velocity to update the
        position. For the pendulum variables this becomes:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$\omega_{1,n+1} = \omega_{1,n} + h\,\alpha_1(y_n)$$</span>
        <span class="mathjax">$$\theta_{1,n+1} = \theta_{1,n} + h\,\omega_{1,n+1}$$</span>
        <span class="mathjax">$$\omega_{2,n+1} = \omega_{2,n} + h\,\alpha_2(y_n)$$</span>
        <span class="mathjax">$$\theta_{2,n+1} = \theta_{2,n} + h\,\omega_{2,n+1}$$</span>
      </div>
      <p>
        This is often much more stable because the position update
        uses the latest velocity rather than the old one. It is not as accurate as RK4, but it is a useful middle ground between simplicity and robustness.
      </p>

      <p>
        <strong>Runge–Kutta 4 (RK4)</strong> is the standard choice for most physics simulations. It
        samples the derivative at four intermediate points and combines them with weighted averages:
      </p>
      <div class="eq-block">
        <span class="mathjax">$$k_1 = f(y_n)$$</span>
        <span class="mathjax">$$k_2 = f\left(y_n + \frac h2 k_1\right)$$</span>
        <span class="mathjax">$$k_3 = f\left(y_n + \frac h2 k_2\right)$$</span>
        <span class="mathjax">$$k_4 = f(y_n + h k_3)$$</span>
        <span class="mathjax">$$y_{n+1} = y_n + \frac h6 (k_1 + 2k_2 + 2k_3 + k_4)$$</span>
      </div>
      <p>
        RK4 is much more accurate than either
        Euler variant. In this app, the animation loop starts from the chosen
        initial angles and angular velocities, computes the accelerations from Section 3, takes one RK4
        step, converts the updated angles into Cartesian coordinates for drawing, and repeats the cycle
        for each frame.
      </p>
    </div>

    <!-- § 5 -->
    <div class="section-block">
      <h3>5 — The Code</h3>
      <p>
        The simulator uses the same ideas directly in code:
        derive the accelerations, choose an update rule, and repeat the step for every frame.
      </p>

      <p>
        First, the current state \([\theta_1, \omega_1, \theta_2, \omega_2]\) is turned into the
        derivative vector \([\omega_1, \alpha_1, \omega_2, \alpha_2]\):
      </p>
      <pre class="code-snippet"><code>function derivs(state) {
  const [t1, w1, t2, w2] = state
  const d = t1 - t2
  const denom = 2 * m1 + m2 - m2 * Math.cos(2 * t1 - 2 * t2)

  const a1 = (-g * (2 * m1 + m2) * Math.sin(t1) -
    m2 * g * Math.sin(t1 - 2 * t2) -
    2 * Math.sin(d) * m2 * (w2 * w2 * L2 + w1 * w1 * L1 * Math.cos(d))) /
    (L1 * denom)

  const a2 = (2 * Math.sin(d) *
    (w1 * w1 * L1 * (m1 + m2) +
      g * (m1 + m2) * Math.cos(t1) +
      w2 * w2 * L2 * m2 * Math.cos(d))) /
    (L2 * denom)

  return [w1, a1, w2, a2]
}</code></pre>

      <p>
        Next, the integrator uses that derivative to advance the state:
      </p>
      <pre class="code-snippet"><code>function euler(state, dt) {
  const k = derivs(state)
  return state.map((x, i) => x + dt * k[i])
}</code></pre>

      <pre class="code-snippet"><code>function rk4(state, dt) {
  const k1 = derivs(state)
  const s2 = state.map((x, i) => x + 0.5 * dt * k1[i])
  const k2 = derivs(s2)
  const s3 = state.map((x, i) => x + 0.5 * dt * k2[i])
  const k3 = derivs(s3)
  const s4 = state.map((x, i) => x + dt * k3[i])
  const k4 = derivs(s4)

  return state.map((x, i) =>
    x + (dt / 6) * (k1[i] + 2 * k2[i] + 2 * k3[i] + k4[i])
  )
}</code></pre>

      <pre class="code-snippet"><code>function eulerCromer(state, dt) {
  const [t1, w1, t2, w2] = state
  const [, a1, , a2] = derivs(state)
  const nw1 = w1 + dt * a1
  const nw2 = w2 + dt * a2

  return [t1 + dt * nw1, nw1, t2 + dt * nw2, nw2]
}</code></pre>

      <p>
        Finally, the main simulation loop simply chooses the integrator and applies it frame by frame:
      </p>
      <pre class="code-snippet"><code>function step(state, dt) {
  switch (integrator.value) {
    case 'euler':
      return euler(state, dt)
    case 'euler-cromer':
      return eulerCromer(state, dt)
    default:
      return rk4(state, dt)
  }
}</code></pre>

      <p>
        Once the state has been updated, the same variables are used to draw the pendulum arms and
        bobs on screen. This is where the maths becomes visible animation:
      </p>
      <pre class="code-snippet"><code>function getPositions() {
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
}</code></pre>

      <p class="callout">
        This is the direct bridge from the mathematics to the animation: the equations define the
        derivatives, the integrator updates the state, and the render loop draws the new positions.
      </p>
    </div>

  </section>
</template>

<style scoped>
.info-panel {
  max-width: 900px;
  margin: 1rem auto 0;
  padding: 1rem 1.1rem;
  background: #11141b;
  border: 1px solid #1f2937;
  border-radius: 10px;
  color: #e5e7eb;
}

.info-panel h2 {
  margin-top: 0;
  font-size: 1rem;
}

.info-panel p {
  color: #d1d5db;
  line-height: 1.45;
}

.section-block {
  margin-top: 1.6rem;
  padding-top: 1.4rem;
  border-top: 1px solid #1f2937;
}

.section-block:first-of-type {
  border-top: none;
  margin-top: 0.8rem;
  padding-top: 0;
}

h3 {
  font-size: 1.3rem;
  font-weight: 600;
  color: #41b883;
  margin: 0 0 0.7rem 0;
  letter-spacing: 0.02em;
}

.eq-block {
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
  background: #0d1017;
  border-left: 3px solid #41b883;
  border-radius: 0 6px 6px 0;
  padding: 0.65rem 1rem;
  margin: 0.75rem 0;
  font-family: 'JetBrains Mono', 'Fira Mono', 'Courier New', monospace;
  font-size: 1.2rem;
  color: #e2e8f0;
  overflow-x: auto;
}

.var-table {
  width: 100%;
  border-collapse: collapse;
  margin: 0.8rem 0;
  font-size: 0.82rem;
}

.var-table th {
  text-align: left;
  padding: 0.4rem 0.7rem;
  border-bottom: 1px solid #374151;
  color: #9ca3af;
  font-weight: 500;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.var-table td {
  padding: 0.38rem 0.7rem;
  border-bottom: 1px solid #1f2937;
  color: #d1d5db;
  vertical-align: top;
}

.var-table td:first-child {
  font-family: 'JetBrains Mono', 'Fira Mono', monospace;
  font-size: 0.8rem;
  color: #41b883;
}

.var-table tr:last-child td {
  border-bottom: none;
}

.callout {
  background: #0f172a;
  border: 1px solid #1e3a5f;
  border-radius: 6px;
  padding: 0.65rem 0.9rem;
  color: #93c5fd;
  font-size: 0.88rem;
  line-height: 1.5;
  margin: 0.75rem 0;
}

.code-snippet {
  background: #090d13;
  border: 1px solid #1f2937;
  border-radius: 8px;
  padding: 0.75rem 0.85rem;
  margin: 0.75rem 0;
  overflow-x: auto;
  font-family: 'JetBrains Mono', 'Fira Mono', 'Courier New', monospace;
  font-size: 0.82rem;
  line-height: 1.45;
  color: #dbe4ee;
}

strong {
  color: #f9fafb;
}

.mini-sim-card {
  margin: 1rem 0 1.2rem;
  padding: 0.95rem;
  border: 1px solid #1f2937;
  border-radius: 10px;
  background: linear-gradient(180deg, #111827 0%, #0b0f17 100%);
}

.mini-sim-copy h3 {
  margin: 0 0 0.35rem;
  color: #41b883;
  font-size: 0.95rem;
}

.mini-sim-copy p {
  margin: 0 0 0.9rem;
  color: #d1d5db;
  font-size: 0.88rem;
  line-height: 1.45;
}

.mini-sim-grid {
  display: grid;
  grid-template-columns: 1fr 220px;
  gap: 0.9rem;
  align-items: center;
}

.mini-canvas {
  width: 360px;
  max-width: 100%;
  height: auto;
  border-radius: 8px;
  border: 1px solid #1f2937;
  background: #090d13;
}

.mini-controls {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.mini-controls label {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  color: #e5e7eb;
  font-size: 0.82rem;
}

.mini-controls strong {
  color: #41b883;
  font-size: 0.9rem;
}

.mini-controls input[type='range'] {
  accent-color: #41b883;
}

.mini-actions {
  display: flex;
  gap: 0.45rem;
}

.mini-reset {
  border: 1px solid #374151;
  border-radius: 6px;
  background: #111827;
  color: #e5e7eb;
  padding: 0.45rem 0.6rem;
  font: inherit;
  font-size: 0.82rem;
  cursor: pointer;
}

.mini-reset:hover {
  background: #18212d;
}

@media (max-width: 840px) {
  .mini-sim-grid {
    grid-template-columns: 1fr;
  }
}
</style>