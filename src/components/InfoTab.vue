<script setup> </script>

<template>
  <section class="info-panel">
    <h2>Physics Behind the Simulation</h2>

    <!-- § 1 -->
    <div class="section-block">
      <h3>§ 1 — System Setup &amp; Coordinates</h3>
      <p>
        A double pendulum consists of two rigid, massless rods. The first rod of length L₁ has a point
        mass m₁ at its tip; a second rod of length L₂ with mass m₂ hangs from that tip. The pivot is
        fixed to a ceiling.
      </p>
      <p>
        We describe the system using two generalised coordinates: θ₁ and θ₂, the angles each rod makes
        with the vertical. The Cartesian positions of each mass follow directly:
      </p>
      <div class="eq-block">
        <span>x₁ = L₁ sin θ₁ &nbsp;&nbsp; y₁ = −L₁ cos θ₁</span>
        <span>x₂ = x₁ + L₂ sin θ₂ &nbsp;&nbsp; y₂ = y₁ − L₂ cos θ₂</span>
      </div>
      <table class="var-table">
        <thead><tr><th>Symbol</th><th>Meaning</th><th>Units</th></tr></thead>
        <tbody>
          <tr><td>θ₁, θ₂</td><td>Angles from vertical for each bob</td><td>rad</td></tr>
          <tr><td>L₁, L₂</td><td>Rod lengths</td><td>m</td></tr>
          <tr><td>m₁, m₂</td><td>Point masses</td><td>kg</td></tr>
          <tr><td>g</td><td>Gravitational acceleration</td><td>m/s²</td></tr>
          <tr><td>T</td><td>Total kinetic energy</td><td>J</td></tr>
          <tr><td>V</td><td>Total potential energy</td><td>J</td></tr>
          <tr><td>ℒ</td><td>Lagrangian = T − V</td><td>J</td></tr>
        </tbody>
      </table>
    </div>

    <!-- § 2 -->
    <div class="section-block">
      <h3>§ 2 — The Lagrangian</h3>
      <p>
        Instead of resolving forces like Newton's approach, Lagrangian mechanics builds equations from
        energy alone. This is especially powerful for constrained systems like pendulums, where
        constraint forces (rod tension) do no work and drop out automatically.
      </p>
      <p>
        Differentiating the position equations gives velocities. The speed of m₁ is simply
        v₁² = L₁²θ̇₁². For m₂, the velocity involves contributions from both rods:
      </p>
      <div class="eq-block">
        <span>v₂² = L₁²θ̇₁² + L₂²θ̇₂² + 2L₁L₂ θ̇₁θ̇₂ cos(θ₁ − θ₂)</span>
      </div>
      <p>
        The cross-term 2L₁L₂ θ̇₁θ̇₂ cos(θ₁ − θ₂) is the key coupling between the two bobs — it is
        what makes this system interesting and what prevents any clean analytical solution.
      </p>
      <p>Total kinetic energy:</p>
      <div class="eq-block">
        <span>T = ½(m₁+m₂)L₁²θ̇₁² + ½m₂L₂²θ̇₂² + m₂L₁L₂ θ̇₁θ̇₂ cos(θ₁−θ₂)</span>
      </div>
      <p>Taking the pivot as the reference height, total potential energy:</p>
      <div class="eq-block">
        <span>V = −(m₁+m₂)gL₁ cos θ₁ − m₂gL₂ cos θ₂</span>
      </div>
      <p>The Lagrangian ℒ = T − V:</p>
      <div class="eq-block">
        <span>ℒ = ½(m₁+m₂)L₁²θ̇₁² + ½m₂L₂²θ̇₂²</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;+ m₂L₁L₂ θ̇₁θ̇₂ cos(θ₁−θ₂)</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;+ (m₁+m₂)gL₁ cos θ₁ + m₂gL₂ cos θ₂</span>
      </div>
    </div>

    <!-- § 3 -->
    <div class="section-block">
      <h3>§ 3 — Equations of Motion</h3>
      <p>The Euler–Lagrange equation is applied separately for each generalised coordinate qᵢ:</p>
      <div class="eq-block">
        <span>d/dt (∂ℒ/∂q̇ᵢ) − ∂ℒ/∂qᵢ = 0</span>
      </div>
      <p>
        Applying this for θ₁ and θ₂ and solving for the angular accelerations gives two coupled,
        nonlinear ODEs:
      </p>
      <div class="eq-block">
        <span>θ̈₁ = [ −g(2m₁+m₂) sin θ₁ − m₂g sin(θ₁−2θ₂)</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;− 2 sin(θ₁−θ₂) · m₂(L₂θ̇₂² + L₁θ̇₁² cos(θ₁−θ₂)) ]</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;/ [ L₁(2m₁+m₂ − m₂ cos(2θ₁−2θ₂)) ]</span>
      </div>
      <div class="eq-block">
        <span>θ̈₂ = [ 2 sin(θ₁−θ₂) · ((m₁+m₂)(L₁θ̇₁² + g cos θ₁)</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;+ L₂m₂θ̇₂² cos(θ₁−θ₂)) ]</span>
        <span>&nbsp;&nbsp;&nbsp;&nbsp;/ [ L₂(2m₁+m₂ − m₂ cos(2θ₁−2θ₂)) ]</span>
      </div>
      <p>
        These equations are coupled (θ̈₁ depends on θ₂ and θ̇₂, and vice versa) and nonlinear
        (products and trig functions of the angles appear throughout). No closed-form analytical
        solution exists, so we must integrate them numerically.
      </p>
      <p class="callout">
        You could use Newton's laws instead, but you would need to track tension forces in both rods
        and resolve them in two dimensions each. The Lagrangian approach lets you ignore constraint
        forces entirely — you only ever differentiate one scalar function.
      </p>
    </div>

    <!-- § 4 -->
    <div class="section-block">
      <h3>§ 4 — Numerical Integration</h3>
      <p>
        To simulate the system, we convert the two second-order ODEs into a system of four first-order
        ODEs by introducing state variables. Define the state vector:
      </p>
      <div class="eq-block">
        <span>y = [θ₁, ω₁, θ₂, ω₂]ᵀ &nbsp; where ωᵢ = θ̇ᵢ</span>
      </div>
      <p>
        Then ẏ = f(y), where f uses the θ̈ expressions from §3. We step this forward in time.
      </p>
      <p>
        <strong>Euler's Method</strong> is the simplest integrator. Given state y at time t, advance
        by timestep h:
      </p>
      <div class="eq-block">
        <span>y(t + h) = y(t) + h · f(y(t))</span>
      </div>
      <p>
        It is simple but accumulates error — energy is not conserved over long runs. For chaotic
        systems like this, errors compound quickly.
      </p>
      <p>
        <strong>Runge–Kutta 4 (RK4)</strong> is the standard for most physics simulations. It
        evaluates the derivative at four intermediate points per step and takes a weighted average:
      </p>
      <div class="eq-block">
        <span>k₁ = f(yₙ)</span>
        <span>k₂ = f(yₙ + h/2 · k₁)</span>
        <span>k₃ = f(yₙ + h/2 · k₂)</span>
        <span>k₄ = f(yₙ + h · k₃)</span>
        <span>&nbsp;</span>
        <span>yₙ₊₁ = yₙ + (h/6)(k₁ + 2k₂ + 2k₃ + k₄)</span>
      </div>
      <p>
        RK4 has local truncation error O(h⁵) and global error O(h⁴), which is massively better than
        Euler's O(h²). For the same accuracy you can use a much larger timestep. The simulation loop
        works as follows: initialise [θ₁, ω₁, θ₂, ω₂] from starting conditions, compute θ̈ from the
        equations of motion, run one RK4 step, convert θ₁ and θ₂ to Cartesian coordinates to draw
        the frame, then repeat.
      </p>
    </div>

    <!-- § 5 -->
    <div class="section-block">
      <h3>§ 5 — Chaos &amp; Sensitivity</h3>
      <p>
        The double pendulum is one of the simplest examples of a chaotic dynamical system. Even though
        the equations are perfectly deterministic, two trajectories with nearly identical starting
        angles diverge exponentially fast. This divergence rate is quantified by the Lyapunov
        exponent λ:
      </p>
      <div class="eq-block">
        <span>|δy(t)| ≈ |δy₀| · e^(λt)</span>
      </div>
      <p>
        Here δy₀ is the initial separation between two trajectories. Because λ &gt; 0, tiny
        measurement errors grow without bound — long-term prediction is fundamentally impossible
        regardless of computing power.
      </p>
      <p class="callout">
        Chaos does not mean randomness. The system is still fully deterministic: given exact initial
        conditions, the trajectory is perfectly defined. Chaos means sensitivity to initial conditions.
        In practice, we can never know positions infinitely precisely, so long-term behaviour becomes
        unpredictable.
      </p>
      <p>
        Because there is no friction or damping, total mechanical energy E = T + V must remain
        constant throughout the simulation. Monitoring energy drift is the standard way to check
        integrator accuracy:
      </p>
      <div class="eq-block">
        <span>εᵣₑₗ(t) = |E(t) − E(0)| / |E(0)| ≪ 1</span>
      </div>
      <p>
        If εᵣₑₗ grows over time, your timestep is too large or your integrator is too low-order. RK4
        with h ≈ 0.01 s typically keeps relative energy error below 10⁻⁵ for several hundred seconds
        of simulation.
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
  font-size: 0.9rem;
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
  font-size: 0.82rem;
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

strong {
  color: #f9fafb;
}
</style>