# Mineral Chance

**A Monte Carlo physics pinball machine that guarantees outcomes while looking completely natural.**

[**Try the live demo**](https://jon-westbrook.github.io/pinball/)

![Mineral Chance Screenshot](https://img.shields.io/badge/status-live-00ff88?style=for-the-badge) ![Matter.js](https://img.shields.io/badge/physics-Matter.js-00f0ff?style=for-the-badge) ![Zero Dependencies](https://img.shields.io/badge/build-zero--deps-ff00aa?style=for-the-badge)

---

## The Problem

You're building an interactive pinball-style game. A ball drops from a moving pipe and falls through a board of obstacles into one of several buckets at the bottom.

Here's the catch:

- The ball **must** land in a specific target bucket
- The ball **must** hit specific labeled obstacles on the way down
- The ball **must** look like real physics — not scripted, not keyframed, not animated along a spline
- The pipe is always moving, so the drop position is unpredictable

Scripted animation paths look fake. Pure physics can't guarantee outcomes. So how do you get **real physics AND deterministic results**?

## The Solution

**Run the simulation hundreds of times before the ball ever drops.**

When the user triggers a drop, the engine:

1. Freezes the current pipe position
2. Spawns **800 invisible physics simulations** with micro-variations in initial velocity
3. Each simulation runs to completion in **headless mode** — no rendering, pure math, finished in milliseconds
4. Filters results for simulations where the ball hit all required obstacles AND landed in the correct bucket
5. Picks a valid simulation and **replays it visually** with the real physics engine

Because the physics engine is deterministic (same inputs = same outputs), the visual replay follows the exact same path as the headless simulation. The ball bounces, tumbles, and collides naturally — because it *is* natural. The only trick is that we already know this particular trajectory is a winner.

This is essentially **Monte Carlo constraint satisfaction** applied to real-time physics.

```
User presses DROP
        |
        v
[Freeze pipe position X]
        |
        v
[Run 800 headless simulations]  <-- ~50-200ms, invisible
  |  vx = random(-1.5, 1.5)
  |  vy = random(0, 0.5)
  |  → full physics to completion
  |  → record: obstacles hit, bucket landed
        |
        v
[Filter: correct obstacles + correct bucket?]
        |
    ┌───┴───┐
    |       |
  Found   None found
    |       |
    v       v
[Replay   [Wait, pipe moves,
 winner]   try again]
```

## How It Works

| Component | Detail |
|-----------|--------|
| **Physics** | Matter.js with fixed timestep (deterministic) |
| **Simulations** | 800 per drop, ~50-200ms total |
| **Board** | 8 labeled obstacles (A-H), scatter pegs, 4 destination buckets |
| **Constraints** | 27 unique selection combinations, each mapping to required obstacles + target bucket |
| **UI** | Three selectors (Rock Type, Depth, Temperature) drive the constraint rules |
| **Debug Panel** | Real-time stats: sims run, valid paths found, search time, selected simulation |

## Run It

It's a single HTML file. No build step. No dependencies to install.

```bash
# Clone and open
git clone https://github.com/Jon-Westbrook/pinball.git
open pinball/index.html
```

Or just visit the [live demo](https://jon-westbrook.github.io/pinball/).

## Why This Approach Works

| Approach | Real Physics? | Guarantees Constraints? | Verdict |
|----------|:---:|:---:|---------|
| Scripted keyframe paths | No | Yes | Looks fake |
| Pure physics, hope for the best | Yes | No | Unreliable |
| Invisible guide forces / PID steering | Kinda | Mostly | Uncanny valley |
| **Monte Carlo pre-simulation** | **Yes** | **Yes** | **Best of both worlds** |

The key insight: you don't need to *control* physics. You just need to *search* physics. Run it enough times with enough variation, filter for winners, and replay the winner. The audience sees real physics. The system guarantees the outcome.

## Tech Stack

- **Matter.js** — deterministic 2D physics engine (CDN, no install)
- **HTML Canvas** — rendering
- **Vanilla JS** — everything else
- **GitHub Pages** — hosting

Single file. No framework. No build tools. No npm. 806 lines.

## License

[MIT](LICENSE) — do whatever you want with it.
