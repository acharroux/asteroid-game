# 🚀 Asteroid Game

A classic Asteroids arcade game built as a **single-file HTML5 application** using the Canvas 2D API and the Web Audio API — no dependencies, no build step.

## 🎮 Play

Just open `index.html` in any modern browser.

## Controls

| Action | Keyboard | Mobile |
|---|---|---|
| Rotate left | `←` / `A` | On-screen button |
| Rotate right | `→` / `D` | On-screen button |
| Thrust | `↑` / `W` | On-screen button |
| Fire | `Space` | On-screen button |
| Hyperspace | `Shift` | On-screen button |

## Features

- 🪨 **Asteroid splitting** — large → medium → small, each with unique procedurally generated shapes
- 🔊 **Synthesized audio** — all sounds generated in real-time via Web Audio API (no audio files)
- ✨ **Particle system** — explosion debris and engine exhaust
- 💫 **Neon glow rendering** — Canvas `shadowBlur` effects, no WebGL required
- 🌌 **Motion blur** — semi-transparent fill creates a cinematic trail effect
- 📱 **Mobile support** — touch controls with on-screen buttons
- ❤️ **Lives & levels** — difficulty increases each level
- 🌀 **Hyperspace** — teleport to a random position (5% chance of death)

## Scoring

| Event | Points |
|---|---|
| Destroy large asteroid | 20 |
| Destroy medium asteroid | 50 |
| Destroy small asteroid | 100 |

## Architecture

The game follows a classic **Entity–Update–Draw** game loop pattern. See [ARCHITECTURE.md](ARCHITECTURE.md) for full design documentation including entity diagrams, state machines, collision detection flow, and rendering layer stack.

## Tech Stack

- **HTML5 Canvas 2D** — rendering
- **Web Audio API** — real-time sound synthesis
- **Vanilla JavaScript** — no frameworks or dependencies
- Single file: `index.html` (~500 lines of JS)