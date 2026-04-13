# Asteroids Game — Design & Architecture

The game is built as a **single-file HTML5 application** using the Canvas 2D API. It follows a classic **game loop architecture** with clear separation of concerns across modules.

---

## Diagrams

### 1. Entity Relationship Diagram

```mermaid
erDiagram
    GAME_STATE {
        number score
        number lives
        number level
        boolean gameRunning
        boolean gameOver
    }

    SHIP {
        number x
        number y
        number angle
        number vx
        number vy
        boolean alive
        number invincible
        boolean thrusting
    }

    BULLET {
        number x
        number y
        number vx
        number vy
        number life
    }

    ASTEROID {
        number x
        number y
        number vx
        number vy
        number rot
        number rotSpeed
        string size
        number radius
        array shape
    }

    PARTICLE {
        number x
        number y
        number vx
        number vy
        number life
        number maxLife
        string color
        number size
    }

    STAR {
        number x
        number y
        number r
        number alpha
    }

    GAME_STATE ||--o| SHIP : "controls"
    GAME_STATE ||--o{ BULLET : "tracks"
    GAME_STATE ||--o{ ASTEROID : "tracks"
    GAME_STATE ||--o{ PARTICLE : "tracks"
    GAME_STATE ||--o{ STAR : "renders"
    SHIP ||--o{ BULLET : "fires"
    SHIP ||--o{ PARTICLE : "emits exhaust"
    BULLET }o--o{ ASTEROID : "collides with"
    ASTEROID ||--o{ PARTICLE : "spawns on hit"
    ASTEROID ||--o{ ASTEROID : "splits into"
```

---

### 2. Game Loop Sequence Diagram

```mermaid
sequenceDiagram
    participant RAF as requestAnimationFrame
    participant GL as gameLoop()
    participant Canvas as Canvas 2D
    participant Ship
    participant Bullets
    participant Asteroids
    participant Particles
    participant Collision

    RAF->>GL: tick
    GL->>Canvas: fillRect (motion blur)
    GL->>Canvas: drawStars()
    GL->>Particles: updateParticles()
    GL->>Canvas: drawParticles()
    GL->>Ship: updateShip()
    Note over Ship: read keys[], apply thrust/rotate
    Ship-->>Particles: spawnThrustParticles()
    Ship-->>Bullets: fireBullet() (if Space)
    GL->>Canvas: drawShip()
    GL->>Bullets: updateBullets()
    GL->>Canvas: drawBullets()
    GL->>Asteroids: updateAsteroids()
    GL->>Canvas: drawAsteroids()
    GL->>Collision: checkCollisions()
    Collision-->>Bullets: remove hit bullet
    Collision-->>Asteroids: remove hit asteroid
    Collision-->>Asteroids: spawn 2× children
    Collision-->>Particles: spawnExplosion()
    Collision-->>Ship: destroyShip() (if hit)
    GL->>GL: updateBgBeat()
    GL->>GL: level-up check
    GL->>RAF: requestAnimationFrame(gameLoop)
```

---

### 3. Ship State Machine

```mermaid
stateDiagram-v2
    [*] --> Alive : createShip()

    Alive --> Thrusting : ArrowUp / W pressed
    Thrusting --> Alive : key released

    Alive --> Rotating : ArrowLeft / ArrowRight pressed
    Rotating --> Alive : key released

    Alive --> Firing : Space pressed + cooldown=0
    Firing --> Alive : bullet spawned

    Alive --> Hyperspace : Shift pressed
    Hyperspace --> Alive : teleport success (95%)
    Hyperspace --> Dead : random death (5%)

    Alive --> Dead : asteroid collision
    Dead --> Invincible : respawn (after 3s delay)
    Invincible --> Alive : invincibility expires
    Dead --> [*] : lives = 0 → Game Over
```

---

### 4. Asteroid Lifecycle

```mermaid
flowchart TD
    A([spawnAsteroids at level start]) --> B[Large asteroid created\nwith random shape & velocity]
    B --> C{Bullet hit?}
    C -- No --> D[Move & wrap around screen]
    D --> C
    C -- Yes --> E[Score += 20\nspawnExplosion\nplaySound explodeLarge]
    E --> F[Spawn 2× Medium asteroids]
    F --> G{Bullet hit?}
    G -- No --> H[Move & wrap]
    H --> G
    G -- Yes --> I[Score += 50\nspawnExplosion\nplaySound explodeMedium]
    I --> J[Spawn 2× Small asteroids]
    J --> K{Bullet hit?}
    K -- No --> L[Move & wrap]
    L --> K
    K -- Yes --> M[Score += 100\nspawnExplosion\nplaySound explodeSmall]
    M --> N{asteroids.length === 0?}
    N -- Yes --> O([Level Up! Spawn more asteroids])
    N -- No --> P([Continue game])
```

---

### 5. Collision Detection Flow

```mermaid
flowchart TD
    CD([checkCollisions called each frame])
    CD --> BL[For each bullet]
    BL --> AL[For each asteroid]
    AL --> D1{distance < asteroid.radius × 0.85?}
    D1 -- No --> AL
    D1 -- Yes --> HIT[Bullet hits asteroid]
    HIT --> SC[Add score]
    HIT --> EX[spawnExplosion]
    HIT --> SP{asteroid.size?}
    SP -- large --> S1[spawn 2× medium]
    SP -- medium --> S2[spawn 2× small]
    SP -- small --> S3[no children]
    HIT --> RM1[remove bullet]
    HIT --> RM2[remove asteroid]
    CD --> SH{ship alive\n& not invincible?}
    SH -- No --> END([done])
    SH -- Yes --> SA[For each asteroid]
    SA --> D2{distance < asteroid.radius×0.7\n+ ship.size×0.6?}
    D2 -- No --> SA
    D2 -- Yes --> DS[destroyShip]
    DS --> LV{lives > 0?}
    LV -- Yes --> RS[respawnTimer = 180\nrespawn after 3s]
    LV -- No --> GO([Game Over])
```

---

### 6. Audio Synthesis Flow

```mermaid
flowchart LR
    EV[Game Event] --> PS[playSound type]
    PS --> OC[Create OscillatorNode]
    PS --> GN[Create GainNode]
    OC -->|connect| GN
    GN -->|connect| DEST[AudioContext.destination\nspeakers]
    OC --> FQ[Set frequency curve\nexponentialRamp]
    GN --> GA[Set gain envelope\nexponentialRamp to ~0]
    FQ & GA --> ST[o.start / o.stop\nfire-and-forget]

    subgraph Sound Types
        SH2[shoot → sawtooth 880→110Hz]
        EL[explodeLarge → square 60→20Hz]
        EM[explodeMedium → square 100→30Hz]
        ES[explodeSmall → sawtooth 200→50Hz]
        TH[thrust → sawtooth noise burst]
        DT[death → triangle 440→55Hz]
        HP[hyperspace → sine 200→2000Hz]
        LU[levelup → 3 sine chord]
    end
```

---

### 7. Rendering Layer Stack

```mermaid
flowchart BT
    A[🖤 Motion-blur fill\nrect rgba 0,0,0,0.25] --> B
    B[⭐ Stars\nstatic white dots] --> C
    C[✨ Particles\nexplosion & thrust sparks] --> D
    D[🚀 Ship\nneon cyan polygon] --> E
    E[🔶 Bullets\nyellow glowing dots] --> F
    F[🪨 Asteroids\norange jagged polygons]

    style A fill:#111,color:#aaa
    style B fill:#111,color:#fff
    style C fill:#111,color:#fa0
    style D fill:#111,color:#0ff
    style E fill:#111,color:#ff0
    style F fill:#111,color:#f80
```

---

### 8. Input Handling Architecture

```mermaid
flowchart TD
    KB[keydown / keyup events] -->|set keys-code = true/false| KM[keys map object]
    TC[touchstart / touchend events\nmobile buttons] -->|set keys-code = true/false| KM
    KM -->|polled every frame| US[updateShip]
    US --> R{keys ArrowLeft\nor A?}
    R -- Yes --> ROT[ship.angle -= TURN_SPEED]
    US --> T{keys ArrowUp\nor W?}
    T -- Yes --> THR[apply thrust vector\nspawnThrustParticles]
    US --> F{keys Space\n& cooldown=0?}
    F -- Yes --> FIR[fireBullet\nshootCooldown = 10]
    US --> H{keys Shift\n& not used?}
    H -- Yes --> HYP[hyperspace teleport]
```

---

## High-Level Structure

```
index.html
├── <style>        — UI layout & overlay styling (CSS)
├── <canvas>       — rendering surface
├── <div#ui>       — HUD (score, level, lives) — fixed overlay, pointer-events: none
├── <div#overlay>  — title/game-over screen
└── <script>       — all game logic (~500 lines of JS)
    ├── Canvas setup & resize
    ├── Audio engine (Web Audio API)
    ├── Constants
    ├── State variables
    ├── Ship module
    ├── Bullet module
    ├── Asteroid module
    ├── Particle system
    ├── Star field
    ├── Collision detection
    ├── Background beat
    ├── Game loop
    ├── Game control (start / end)
    └── Input handling (keyboard + touch)
```

---

## Core Pattern: Entity–Update–Draw

Every game object follows the same **3-step lifecycle** each frame:

```
update*(entity)  →  mutates position/velocity/state
draw*(entity)    →  renders to canvas
```

| Module    | Update fn           | Draw fn           |
|-----------|---------------------|-------------------|
| Ship      | `updateShip()`      | `drawShip(s)`     |
| Bullets   | `updateBullets()`   | `drawBullets()`   |
| Asteroids | `updateAsteroids()` | `drawAsteroids()` |
| Particles | `updateParticles()` | `drawParticles()` |

---

## Game Loop (`gameLoop`)

```
requestAnimationFrame(gameLoop)
  │
  ├── Clear canvas (motion blur via semi-transparent fill)
  ├── drawStars()
  ├── updateParticles() → drawParticles()
  ├── updateShip()      → drawShip()
  ├── updateBullets()   → drawBullets()
  ├── updateAsteroids() → drawAsteroids()
  ├── checkCollisions()
  ├── updateBgBeat()
  └── Level-up check (asteroids.length === 0)
```

The **motion blur** effect is achieved by filling the canvas each frame with a semi-transparent black (`rgba(0,0,0,0.25)`) instead of fully clearing it — previous frames fade out naturally.

---

## State Management

All mutable game state lives in **module-level variables**:

```js
let ship, bullets, asteroids, particles;  // entity arrays
let score, lives, level;                  // game stats
let keys = {};                            // keyboard state map
let shootCooldown, respawnTimer;          // timers
```

The `keys` object is a simple **boolean map** (`{ ArrowUp: true, Space: false, ... }`) updated by `keydown`/`keyup` listeners. Input is *polled* inside `updateShip()` each frame rather than being event-driven — this is the standard game-loop input pattern.

---

## Asteroid Splitting Logic

```
bullet hits asteroid
        │
        ├── size === 'large'  → spawn 2× 'medium'
        ├── size === 'medium' → spawn 2× 'small'
        └── size === 'small'  → no children (destroyed)
```

Each asteroid's shape is a **procedurally generated polygon** — a fixed number of vertices placed at random radii around a circle, giving every asteroid a unique jagged look.

---

## Particle System

A **single shared `particles[]` array** handles both explosion debris and engine exhaust. Each particle is a plain object:

```js
{ x, y, vx, vy, life, maxLife, color, size }
```

Alpha is computed as `life / maxLife` — particles fade out linearly. This avoids any class overhead and keeps the system simple.

---

## Audio (Web Audio API)

No audio files are used. Every sound is **synthesized in real-time**:

- Oscillator type (`sine` / `sawtooth` / `square`) sets the timbre
- `frequency.exponentialRampToValueAtTime` creates pitch sweeps
- `gain.exponentialRampToValueAtTime` creates natural fade-outs
- Each call creates and immediately disposes its own oscillator node (fire-and-forget pattern)

The **background heartbeat** uses the same approach — two alternating low-frequency tones whose interval shrinks as fewer asteroids remain, creating tension.

---

## Wrapping / Screen Edges

All entities use a single `wrap(v, max)` helper:

```js
function wrap(v, max) {
  if (v < 0)   return v + max;
  if (v > max) return v - max;
  return v;
}
```

Applied to both `x` and `y` each frame — entities seamlessly cross screen edges.

---

## Rendering Layers (painter's algorithm, back-to-front)

```
1. Motion-blur fill (semi-transparent black)
2. Stars (static background)
3. Particles (behind everything)
4. Ship
5. Bullets
6. Asteroids
```

Neon glow is achieved with Canvas `shadowBlur` + `shadowColor` — no WebGL required.