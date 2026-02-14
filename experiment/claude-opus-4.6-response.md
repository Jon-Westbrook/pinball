# Constrained Pinball Physics: Approach Analysis

## Model: Claude Opus 4.6 (via GitHub Copilot)

---

This is a genuinely interesting engineering problem. The core tension is between three competing requirements:

1. **Deterministic outcome** — ball MUST hit specific obstacles and land in a specific bucket
2. **Physical realism** — ball must look like it obeys real physics
3. **Variable start position** — the pipe is always moving, so the initial x-position is unknown until drop time

Let me work through the solution space systematically.

---

## Exhaustive Approach Inventory

### 1. Pre-simulated Lookup Table (Offline Monte Carlo)

**Mechanism:** Before the exhibit runs, simulate millions of ball drops using a real physics engine (Matter.js, Box2D, etc.) from every possible pipe x-position (discretized to, say, 1px increments). For each starting position, record which obstacles are hit and which bucket the ball lands in. Store the results in a lookup table: `(start_x) → [(obstacle_set, bucket, initial_velocity)]`. At runtime, when the visitor makes a selection, wait until the pipe reaches an x-position where the lookup table has an entry matching the required `(obstacle_set, bucket)`. Apply the stored initial velocity and let the real physics engine run live.

**Feasibility:** High. The screen width might be ~1920px, so you'd have ~1920 starting positions. For each, you simulate thousands of drops with varying initial velocities (angle, speed). This is very tractable — a few million simulations that can run overnight.

**Realism:** Excellent. The ball is running through an actual physics engine at runtime with real collision response. Nothing is scripted. The physics are identical to the pre-simulation because the engine is deterministic given the same inputs.

**Constraint guarantee:** Yes, by construction — you only drop when you have a proven solution.

**Weakness:** Requires the physics engine to be deterministic (same inputs → same outputs). Most engines are, but floating-point differences across browsers/platforms can cause divergence. Mitigation: use a fixed-step physics loop with a specific engine version, or add subtle invisible guide forces as a safety net (see approach 5).

---

### 2. Real-Time Rapid Simulation (Look-Ahead)

**Mechanism:** When the visitor makes their selection, don't drop the ball immediately. Instead, on every frame, run a fast-forward physics simulation from the pipe's *current* position — simulate the full ball trajectory at, say, 100x speed in a hidden/offscreen world. Test a spread of initial velocities. If any trajectory satisfies the constraints (correct obstacles + correct bucket), immediately drop the ball with that velocity in the visible world. The pipe is still moving, so you're testing a new batch every frame until one works.

**Feasibility:** Medium-high. A single pinball trajectory might take ~2-3 seconds of simulated time. At 60Hz physics with 100x speedup, that's ~180 physics steps per candidate trajectory. If you test 50 velocity variants per frame, that's 9,000 physics steps per frame — tight but doable with an optimized engine (Box2D in WASM, for instance). You could also spread this across web workers.

**Realism:** Excellent — identical to approach 1, since the runtime ball uses the real engine.

**Constraint guarantee:** Yes, same as approach 1.

**Weakness:** Latency. There may be a delay (a few frames to a few hundred ms) between the visitor's input and the ball dropping, while the system searches for a valid trajectory. This is actually fine UX — the pipe is moving anyway, and a short "charging" animation can mask it.

---

### 3. Invisible Guide Rails / Force Fields (Runtime Correction)

**Mechanism:** Drop the ball with real physics immediately. During the simulation, apply small invisible forces to nudge the ball toward required obstacles and away from wrong paths. Essentially a PID controller or potential field that gently steers the ball while the physics engine handles collisions, gravity, and bounce behavior.

**Feasibility:** High — straightforward to implement.

**Realism:** Risky. Small corrections can look fine. But if the ball starts far from the needed path, the corrections become large and visible — balls curving mid-air, unnatural lateral movement, etc. This is essentially what "looks scripted" means. The further the start position is from the ideal, the more aggressive the steering, and the worse it looks.

**Constraint guarantee:** Probabilistic, not guaranteed. Strong corrections guarantee the path but kill realism. Weak corrections preserve realism but may miss constraints.

**Weakness:** This is a spectrum with no sweet spot. It's the approach most teams try first and most clients reject. Your junior dev's "scripted animation paths" was likely a variant of this.

---

### 4. Dynamic Obstacle Reconfiguration

**Mechanism:** Instead of steering the ball, move the world. When the ball is dropped, subtly reposition obstacles in real-time to create a valid path from the current start position. Pegs shift a few pixels, bumper angles adjust slightly, funnel walls tilt — all animated smoothly enough to look intentional (like the board is "alive"). The ball runs through real physics on the modified layout.

**Feasibility:** Medium. Requires pre-computing valid layouts for each start position (similar to approach 1 but in layout-space rather than velocity-space). The animation of obstacles moving must look intentional, not glitchy.

**Realism of ball physics:** Excellent — the ball interacts with real obstacles via real physics. But the moving obstacles might look strange if not carefully art-directed.

**Constraint guarantee:** Yes, if the layout solver is correct.

**Weakness:** Visually distracting. Visitors may notice pegs shifting. Could work if the exhibit's art direction embraces it (e.g., the board is "geological layers" that shift), but it's a design risk.

---

### 5. Hybrid: Pre-Simulated Lookup + Runtime Safety Net

**Mechanism:** Combine approaches 1 and 3. Use the pre-simulated lookup table to pick a start position and initial velocity that's proven to work. Run the real physics engine. But also monitor the ball's trajectory at runtime — if floating-point drift or browser differences cause it to deviate from the expected path, apply tiny corrective forces (< 1% of gravity) to nudge it back on track.

**Feasibility:** High.

**Realism:** Excellent. The corrections are so small they're imperceptible because the ball is already on approximately the right path. You're correcting millimeters, not centimeters.

**Constraint guarantee:** Yes — the lookup provides the plan, the safety net ensures execution.

**Weakness:** Slightly more complex implementation. But this is the engineering-grade solution.

---

### 6. Teleport / Cut Camera (Cinematic Cheat)

**Mechanism:** When the ball is in a visually busy area (bouncing off a cluster of pegs, behind a bumper, in a particle effect), briefly obscure it and reposition it onto a known-good path. Think of it like a movie cut — the audience doesn't notice the discontinuity.

**Feasibility:** Medium. Requires careful identification of "occlusion moments" and pre-planned safe positions.

**Realism:** Can work if the visual effects are good enough. But on a large museum screen, sharp-eyed visitors (especially kids) will notice teleportation. Risky.

**Constraint guarantee:** Yes, at the moment of teleportation.

**Weakness:** Fragile. Depends on visual cover that may not always be available at the right moment.

---

### 7. Deferred Drop Timing (Wait for the Right Moment)

**Mechanism:** Don't drop the ball until the pipe reaches a position where a known-good trajectory exists. The pipe moves continuously, so it will eventually reach a valid x-position. Use the pre-simulated lookup table from approach 1 to know which positions are valid. When the pipe hits one, drop immediately.

**Feasibility:** High — this is actually a simplification of approach 1.

**Realism:** Excellent. The only question is whether visitors notice the delay.

**Constraint guarantee:** Yes.

**Weakness:** Latency depends on how many valid start positions exist for the required path. If the board is well-designed, there should be valid positions every ~50-100px of pipe travel, meaning < 0.5s delay at typical pipe speeds. If valid positions are sparse, the delay could be noticeable.

---

### 8. Board Design as Primary Constraint Solver

**Mechanism:** Design the physical board layout so that the constraint problem is easy. Use funnels, channels, and converging geometry so that balls from a wide range of start positions naturally converge toward the same obstacle sequences. Think of a Galton board, but with intentional asymmetric funnels. If the board geometry does most of the work, the start position matters less, and simple velocity selection handles the rest.

**Feasibility:** High, but requires iterative board design. This is a design-time investment, not a runtime algorithm.

**Realism:** Excellent — the board literally *is* real physics. Funnels and channels are physically plausible.

**Constraint guarantee:** Not standalone, but dramatically reduces the difficulty of approaches 1/2/7.

**Weakness:** Can't solve the problem alone (some start positions will still produce wrong paths), but it's a force multiplier for other approaches.

---

## Approaches I Considered and Rejected

- **Machine learning / reinforcement learning**: Overkill, non-deterministic, hard to guarantee constraints. A lookup table is strictly better for a finite, pre-computable problem.
- **Spline/keyframe animation with physics-like easing**: This is what the junior dev tried. It's nearly impossible to make hand-authored paths look like real physics, especially on bounces and collisions.
- **Reverse simulation (start from bucket, simulate backward)**: Physics engines don't reverse well — collisions are lossy and non-invertible. Energy dissipation in bounces means there's no unique reverse path.

---

## Final Ranking

### Rank 1: Pre-Simulated Lookup + Deferred Drop + Safety Net (Approaches 1 + 7 + 5)

This is the synthesis. Here's how it works end-to-end:

1. **Offline (build time):** Simulate millions of drops across all pipe positions and a range of initial velocities. Store a table: `Map<start_x, Array<{velocity, obstacles_hit, bucket}>>`.
2. **Board design (design time):** Use approach 8 — design the board with funnels/converging geometry so that most start positions have valid solutions. This makes the lookup table dense.
3. **Runtime — visitor selects:** Look up which `(obstacle_set, bucket)` combo is required.
4. **Runtime — wait for valid position:** Each frame, check if the pipe's current x has a matching entry in the lookup table. When it does, drop the ball with the stored velocity.
5. **Runtime — physics runs live:** The real physics engine (Matter.js, Rapier, etc.) simulates the ball with actual collisions.
6. **Runtime — safety net:** Monitor the ball's position against the expected trajectory. If deviation exceeds a threshold, apply micro-corrections (tiny forces, < 0.5% of gravity magnitude) to converge back.

**Why this is the best:**

- Ball physics are real and look real — visitors can't distinguish it from a true random drop
- Constraints are guaranteed by construction (proven trajectories) with a safety net for floating-point drift
- Latency is minimal if the board is well-designed (~100-300ms typical)
- Implementation is straightforward — a physics engine + a JSON lookup table + a simple force controller
- Robust to browser/platform differences thanks to the safety net

### Rank 2: Real-Time Look-Ahead (Approach 2)

Use this if the board layout needs to change dynamically (e.g., different exhibit modes) and pre-simulation isn't practical. The tradeoff is higher runtime compute cost and slightly more implementation complexity (web workers, offscreen physics worlds). But it eliminates the need for offline pre-computation.

### Rank 3: Board Design + Simple Deferred Drop (Approaches 8 + 7)

The minimal-engineering solution. Invest heavily in board design so that convergent geometry does most of the work, then use a simple lookup table with deferred drops. Skip the safety net. This works if you can guarantee deterministic physics (fixed-step simulation, single target platform like Chrome on a kiosk machine — which a museum exhibit likely is).

---

## Implementation Note

Since this is a museum kiosk (single browser, single machine, controlled environment), physics determinism is highly achievable. You control the browser version, OS, and hardware. This means approach 1+7 alone (without the safety net) is probably sufficient in practice. Add the safety net if you want belt-and-suspenders reliability, which I'd recommend for a client-facing exhibit.
