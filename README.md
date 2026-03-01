# 🔬 STRM-OVR/unified — Real-Time Multi-Protocol Overlay Engine

[![Architecture: Event-Driven Microkernel](https://img.shields.io/badge/arch-event--driven%20microkernel-blueviolet)]()
[![Protocol: IRC/WSS + EventSub/WSS](https://img.shields.io/badge/protocol-IRC%2FWSS%20%2B%20EventSub%2FWSS-informational)]()
[![Rendering: Canvas2D + requestAnimationFrame Pipeline](https://img.shields.io/badge/render-Canvas2D%20%2B%20rAF%20pipeline-green)]()
[![Auth: OAuth2 PKCE + Fragment Isolation](https://img.shields.io/badge/auth-OAuth2%20PKCE%20%2B%20Fragment%20Isolation-critical)]()
[![State Machine: Hierarchical FSM with 17 Transition Tables](https://img.shields.io/badge/FSM-17%20hierarchical%20states-yellow)]()
[![License: OSS](https://img.shields.io/badge/license-OSS-lightgrey)]()

---

> **⚠️ NOTA DE COMPLEJIDAD:** Este sistema implementa un motor de overlay en tiempo real con un pipeline de renderizado a 60fps, gestión de estado mediante máquinas de estados finitos jerárquicas (HFSM), y orquestación multi-conexión sobre WebSocket con reconexión exponencial backoff y jitter. No es un "HTML con JavaScript". Es una arquitectura de microkernel event-driven desplegada como Single Page Application con zero-server-side state. Leer la documentación completa antes de intentar modificar cualquier cosa.

---

## 📐 Tabla de Contenidos

1. [Resumen de Arquitectura](#-resumen-de-arquitectura)
2. [Diagrama de Flujo del Sistema](#-diagrama-de-flujo-del-sistema)
3. [Modelo de Concurrencia y Event Loop](#-modelo-de-concurrencia-y-event-loop)
4. [Subsistema de Renderizado — Canvas2D Pipeline](#-subsistema-de-renderizado--canvas2d-pipeline)
5. [Protocolo de Comunicación Multi-Socket](#-protocolo-de-comunicación-multi-socket)
6. [Motor de Física y Simulación de Partículas](#-motor-de-física-y-simulación-de-partículas)
7. [Máquinas de Estados Finitos (FSM) — Tabla de Transiciones](#-máquinas-de-estados-finitos-fsm--tabla-de-transiciones)
8. [Pipeline de Internacionalización (i18n) — Traducción en Tiempo Real](#-pipeline-de-internacionalización-i18n--traducción-en-tiempo-real)
9. [Modelo de Seguridad — OAuth2 Fragment Isolation](#-modelo-de-seguridad--oauth2-fragment-isolation)
10. [Sistema de Audio — Web Audio API con Pooling](#-sistema-de-audio--web-audio-api-con-pooling)
11. [API de Extensión — Esquema de Recompensas](#-api-de-extensión--esquema-de-recompensas)
12. [Comandos de Protocolo — Parser y Dispatcher](#-comandos-de-protocolo--parser-y-dispatcher)
13. [Despliegue y Configuración](#-despliegue-y-configuración)
14. [Formato de Payload del Pixel Art](#-formato-de-payload-del-pixel-art)
15. [Parámetros de Configuración Runtime](#-parámetros-de-configuración-runtime)
16. [Depuración Avanzada](#-depuración-avanzada)
17. [Formato de Audio — Codec Pipeline](#-formato-de-audio--codec-pipeline)
18. [Licencia](#-licencia)

---

## 🏗 Resumen de Arquitectura

STRM-OVR/unified implementa un patrón **microkernel event-driven** donde el núcleo del sistema es un dispatcher central de eventos que coordina subsistemas desacoplados a través de un bus de mensajes interno. Cada subsistema opera de forma autónoma y se comunica exclusivamente mediante eventos tipados.

### Componentes del Microkernel

```
┌──────────────────────────────────────────────────────────────────┐
│                    STRM-OVR/unified Runtime                      │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │  Protocol    │  │ EventSub     │  │  Canvas2D Renderer     │  │
│  │  Parser      │  │ Orchestrator │  │  (rAF Pipeline)        │  │
│  │  (Dual-Conn) │  │              │  │                        │  │
│  │             │  │ ┌──────────┐ │  │ ┌────────────────────┐ │  │
│  │ ┌─────────┐ │  │ │ WSS      │ │  │ │ Entity Manager     │ │  │
│  │ │ Channel │ │  │ │ Handshake│ │  │ │ (Object Pool)      │ │  │
│  │ │ Socket  │ │  │ └──────────┘ │  │ └────────────────────┘ │  │
│  │ └─────────┘ │  │ ┌──────────┐ │  │ ┌────────────────────┐ │  │
│  │ ┌─────────┐ │  │ │ Sub      │ │  │ │ Physics Engine     │ │  │
│  │ │ Relay   │ │  │ │ Manager  │ │  │ │ (Verlet + Euler)   │ │  │
│  │ │ Socket  │ │  │ └──────────┘ │  │ └────────────────────┘ │  │
│  │ └─────────┘ │  └──────────────┘  │ ┌────────────────────┐ │  │
│  └─────────────┘                     │ │ Particle System    │ │  │
│                                      │ │ (Emitter Pool)     │ │  │
│  ┌─────────────┐  ┌──────────────┐  │ └────────────────────┘ │  │
│  │ Translation │  │ Audio        │  └────────────────────────┘  │
│  │ Pipeline    │  │ Subsystem    │                                │
│  │             │  │              │                                │
│  │ ┌─────────┐ │  │ ┌──────────┐ │  ┌────────────────────────┐  │
│  │ │ Lang    │ │  │ │ Sound    │ │  │  State Machine Engine  │  │
│  │ │ Detect  │ │  │ │ Pool     │ │  │  (Hierarchical FSM)    │  │
│  │ └─────────┘ │  │ └──────────┘ │  │                        │  │
│  │ ┌─────────┐ │  │ ┌──────────┐ │  │ ┌────────────────────┐ │  │
│  │ │ GTX     │ │  │ │ Playback │ │  │ │ 17 State Tables    │ │  │
│  │ │ Bridge  │ │  │ │ Sched.   │ │  │ │ + Transition Maps  │ │  │
│  │ └─────────┘ │  │ └──────────┘ │  │ └────────────────────┘ │  │
│  └─────────────┘  └──────────────┘  └────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Event Bus (Pub/Sub Dispatcher)                │  │
│  │  Channels: proto.message | eventsub.redeem | render.tick  │  │
│  │            physics.step | audio.trigger | fsm.transition   │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Stack Tecnológico

| Capa | Tecnología | Justificación |
|------|-----------|---------------|
| Renderizado | Canvas2D con `requestAnimationFrame` | Renderizado de bitmaps a nivel de píxel con composición alpha. No se usa WebGL para mantener compatibilidad con el compositor Chromium embebido (CEF 95+). |
| Networking | WebSocket (RFC 6455) sobre TLS 1.3 | Protocolo full-duplex con overhead mínimo (2-14 bytes header vs HTTP). Dos conexiones de protocolo + una de eventos. |
| Estado | Hierarchical Finite State Machine | 17 estados con tablas de transición explícitas, guards condicionales y acciones de entrada/salida. Previene race conditions en eventos concurrentes. |
| Audio | HTMLAudioElement con pooling manual | Pre-carga de buffers con `preload="auto"`. Pool de instancias para reproducción solapada sin Web Audio API overhead. |
| Auth | OAuth2 Implicit Grant + URI Fragment | Los tokens viajan en el hash (`#`) del URI, que según RFC 3986 §3.5 nunca se transmite al servidor HTTP. Zero server-side exposure. |
| i18n | Google Translate GTX endpoint | Endpoint con rate limiting dinámico. Detección de idioma automática + traducción bidireccional con fallback a no-op. |
| Física | Integración Euler semi-implícita + Verlet | Euler para entidades simples (O(n)), Verlet para cadenas de partículas con constraints (O(n·k) donde k = iteraciones del solver). |
| Deploy | GitHub Pages (Jekyll bypass) | SPA estática con `nojekyll`. CDN global con cache invalidation vía commit hash. |

---

## 🔄 Diagrama de Flujo del Sistema

```
                    ┌──────────────────┐
                    │   Host Runtime   │
                    │   (CEF/Chromium) │
                    └────────┬─────────┘
                             │ load
                             ▼
                    ┌──────────────────┐
                    │  Parse URI Hash  │
                    │  (Fragment API)  │
                    └────────┬─────────┘
                             │ config{}
                             ▼
              ┌──────────────────────────────┐
              │     Bootstrap Sequence       │
              │                              │
              │  1. Validate config schema   │
              │  2. Init Canvas2D context    │
              │  3. Pre-load audio buffers   │
              │  4. Connect WSS (channel)    │
              │  5. Connect WSS (relay)      │
              │  6. Connect EventSub WSS     │
              │  7. Subscribe reward events  │
              │  8. Start rAF render loop    │
              └──────────────┬───────────────┘
                             │
              ┌──────────────┼───────────────┐
              │              │               │
              ▼              ▼               ▼
     ┌──────────────┐ ┌───────────┐ ┌──────────────┐
     │  WSS.channel │ │ WSS.relay │ │  EventSub    │
     │  (read-only) │ │ (r/w)     │ │  (read-only) │
     └──────┬───────┘ └─────┬─────┘ └──────┬───────┘
            │               │               │
            ▼               │               ▼
     ┌──────────────┐      │        ┌──────────────┐
     │ Message      │      │        │ Redemption   │
     │ Parser       │      │        │ Dispatcher   │
     │              │      │        │              │
     │ MSG →        │      │        │ reward.title │
     │  extract:    │      │        │ → lookup in  │
     │  - user      │      │        │   reward map │
     │  - command   │      │        │ → dispatch   │
     │  - payload   │      │        │   FSM event  │
     └──────┬───────┘      │        └──────┬───────┘
            │               │               │
            ▼               │               ▼
     ┌──────────────────────────────────────────────┐
     │            Event Bus (Central Hub)            │
     │                                               │
     │  proto.message ──→ [Command Router]           │
     │                    ├─ !dibujar → PixelArt.add │
     │                    ├─ !test    → Debug.spawn   │
     │                    ├─ !limpiar → Entity.clear  │
     │                    └─ !redes   → Relay.reply   │
     │                                               │
     │  eventsub.redeem ──→ [FSM Trigger]            │
     │                       ├─ Pelea    → fight.fsm │
     │                       ├─ Boom     → explosion  │
     │                       ├─ Tornado  → vortex     │
     │                       ├─ Pachinko → minigame   │
     │                       └─ ... (17 total)       │
     │                                               │
     │  render.tick ──→ [Physics] → [Draw] → [Flip]  │
     └──────────────────────────────────────────────┘
            │                               │
            ▼                               ▼
     ┌──────────────┐              ┌──────────────┐
     │  Translation │              │   Canvas2D   │
     │  Pipeline    │              │   Composite  │
     │              │              │              │
     │  detect() →  │              │  clearRect() │
     │  translate()→│              │  drawImage() │
     │  format() → │              │  fillText()  │
     │  relay() →  │              │  composite() │
     └──────────────┘              └──────────────┘
```

---

## ⚙ Modelo de Concurrencia y Event Loop

El sistema opera dentro del event loop del navegador (CEF/Chromium) con las siguientes consideraciones de scheduling:

### Prioridades de Ejecución

| Cola | Prioridad | Subsistema | Latencia Máxima |
|------|-----------|-----------|-----------------|
| Microtask | Alta | Promise chains (fetch/translate) | < 1ms |
| rAF callback | Media-Alta | Render pipeline (60fps target = 16.67ms budget) | < 16.67ms |
| Macrotask | Media | WebSocket `onmessage` handlers | < 50ms |
| setTimeout | Baja | Reconnection timers, cleanup | Variable |

### Frame Budget Distribution

```
16.67ms total frame budget (60fps)
├── Physics step:     ~2ms   (Euler integration + collision)
├── State evaluation: ~0.5ms (FSM guard checks)
├── Entity update:    ~3ms   (position, animation frame, alpha)
├── Draw calls:       ~8ms   (clearRect + N×drawImage + text)
├── Particle render:  ~2ms   (emitter pool iteration)
└── Headroom:         ~1.17ms (GC, browser compositor)
```

> **⚠️ Performance Note:** Con >200 entidades activas + sistema de partículas, el frame budget puede exceder 16.67ms en hardware bajo. El renderer implementa frame dropping adaptativo — si `deltaTime > 33ms`, se skipea el frame de renderizado pero se ejecuta el physics step con dt acumulado para mantener coherencia de simulación.

---

## 🖥 Subsistema de Renderizado — Canvas2D Pipeline

### Pipeline de Render por Frame

```
requestAnimationFrame(callback)
│
├─ 1. calcDeltaTime()
│     └─ Clamp dt to [0, 100ms] to prevent spiral of death
│
├─ 2. updatePhysics(dt)
│     ├─ For each entity:
│     │   ├─ Apply gravity: vy += G * dt
│     │   ├─ Apply friction: vx *= FRICTION
│     │   ├─ Euler integration: x += vx * dt, y += vy * dt
│     │   ├─ Boundary collision (floor, walls)
│     │   └─ Entity-specific behavior (walk, jump, dance...)
│     └─ For active effects:
│         └─ Particle emitters: spawn, integrate, kill expired
│
├─ 3. updateAnimations(dt)
│     ├─ Sprite frame advancement (per-entity timer)
│     ├─ Alpha interpolation (fade in/out)
│     └─ Scale interpolation (zoom effects)
│
├─ 4. resolveStateTransitions()
│     └─ Check FSM guards → execute transitions → fire actions
│
├─ 5. render()
│     ├─ ctx.clearRect(0, 0, W, H)
│     ├─ Draw background effects (rain, volcano lava)
│     ├─ Sort entities by z-index
│     ├─ For each entity:
│     │   ├─ ctx.save()
│     │   ├─ ctx.globalAlpha = entity.alpha
│     │   ├─ ctx.translate(entity.x, entity.y)
│     │   ├─ ctx.scale(entity.scaleX, entity.scaleY)
│     │   ├─ ctx.drawImage(entity.sprite, 0, 0)  // Nearest-neighbor
│     │   ├─ ctx.fillText(entity.username)        // Label
│     │   └─ ctx.restore()
│     ├─ Draw foreground effects (explosions, text popups)
│     └─ Draw debug overlay (if enabled)
│
└─ 6. scheduleNextFrame()
      └─ requestAnimationFrame(callback)  // Loop
```

### Rendering Hints

El canvas opera con `imageSmoothingEnabled = false` para preservar la estética pixel-art. Cada sprite se renderiza sin interpolación bilineal, manteniendo bordes nítidos al escalar.

```
ctx.imageSmoothingEnabled = false;           // Chromium
ctx.mozImageSmoothingEnabled = false;        // Firefox (legacy)
ctx.webkitImageSmoothingEnabled = false;     // WebKit (legacy)
ctx.msImageSmoothingEnabled = false;         // Edge Legacy
```

---

## 🔌 Protocolo de Comunicación Multi-Socket

### Protocolo de Mensajería sobre WebSocket (RFC 1459 + IRCv3 Tags)

El sistema mantiene **dos conexiones de mensajería simultáneas** sobre WSS:

#### Conexión #1 — Channel Reader

```
→ PASS {AUTH_TOKEN}
→ NICK {channel_username}
→ CAP REQ :tags commands
→ JOIN #{channel}
← :server 001 channel_username :Welcome
← :server 376 channel_username :>
← @badge-info=...;display-name=User MSG #{channel} :!dibujar ABC123
```

**Capabilities solicitadas:**
- `tags` — Metadata IRCv3 (badges, color, emotes, user-id)
- `commands` — Notificaciones, clear events, room state

#### Conexión #2 — Relay Writer

```
→ PASS {RELAY_AUTH_TOKEN}
→ NICK {relay_username}
→ CAP REQ :tags commands
→ JOIN #{channel}
→ MSG #{channel} :🌐 User said: "Hello everyone"
```

> **Separación de concerns:** La conexión de lectura y escritura están separadas para evitar que mensajes de rate-limiting del servidor (20 msg/30s para cuentas estándar) afecten la recepción de comandos.

#### Protocolo de Reconexión

```
Reconnection strategy: Exponential backoff with jitter
──────────────────────────────────────────────────────
Attempt 1: wait  1000ms + random(0, 500)ms
Attempt 2: wait  2000ms + random(0, 500)ms
Attempt 3: wait  4000ms + random(0, 500)ms
Attempt 4: wait  8000ms + random(0, 500)ms
Attempt 5: wait 16000ms + random(0, 500)ms
...
Max wait:  60000ms (cap)
Reset:     After successful PING/PONG cycle
```

### EventSub sobre WebSocket

```
Client                          Event Server
  │                                    │
  │──── WSS CONNECT ──────────────────→│
  │                                    │
  │←── session_welcome ───────────────│
  │    { session: { id: "abc123" } }   │
  │                                    │
  │──── POST /api/eventsub/sub ──────→│ (via fetch, not WS)
  │     { type: "reward.redemption.add",
  │       transport: { session_id: "abc123" } }
  │                                    │
  │←── subscription confirmed ────────│
  │                                    │
  │←── notification ──────────────────│
  │    { event: {                      │
  │        reward: { title: "Pelea" }, │
  │        user_name: "viewer123"      │
  │    }}                              │
  │                                    │
  │←── session_keepalive ─────────────│ (every ~10s)
  │                                    │
```

**Keepalive timeout:** Si no se recibe un `session_keepalive` en el intervalo especificado por `keepalive_timeout_seconds` + margen de 5s, el cliente asume desconexión y reinicia el ciclo completo (nueva conexión WSS → nuevo session_id → nueva suscripción).

---

## 🌀 Motor de Física y Simulación de Partículas

### Constantes del Motor

| Constante | Valor | Unidad | Descripción |
|-----------|-------|--------|-------------|
| `GRAVITY` | 0.3 | px/frame² | Aceleración gravitatoria vertical |
| `FRICTION` | 0.95 | coeff | Factor de fricción horizontal por frame |
| `FLOOR_Y` | canvas.height - 60 | px | Coordenada Y del suelo |
| `BOUNCE` | -0.6 | coeff | Coeficiente de restitución en colisiones |
| `MAX_VX` | 8.0 | px/frame | Velocidad horizontal máxima |
| `MAX_VY` | 15.0 | px/frame | Velocidad vertical máxima |
| `WALK_SPEED` | 1.0 | px/frame | Velocidad base de caminata |
| `JUMP_FORCE` | -8.0 | px/frame | Impulso vertical del salto |

### Integración Numérica

Para entidades estándar, se usa **Euler semi-implícito** (symplectic Euler):

```
// Paso 1: Actualizar velocidad
vy(t+dt) = vy(t) + gravity * dt

// Paso 2: Aplicar fricción
vx(t+dt) = vx(t) * friction

// Paso 3: Actualizar posición con velocidad nueva
x(t+dt) = x(t) + vx(t+dt) * dt
y(t+dt) = y(t) + vy(t+dt) * dt

// Paso 4: Resolver constraints (floor collision)
if y(t+dt) > FLOOR_Y:
    y(t+dt) = FLOOR_Y
    vy(t+dt) = vy(t+dt) * BOUNCE
```

> **¿Por qué Euler semi-implícito y no RK4?** El error de orden O(dt) es aceptable para animaciones visuales no-críticas. RK4 cuadruplicaría las evaluaciones de fuerza sin beneficio perceptible a 60fps.

### Sistema de Partículas

Las partículas se gestionan mediante un **object pool** pre-alocado para evitar garbage collection spikes:

```
ParticlePool {
    pool: Array<Particle>[MAX_PARTICLES]    // Pre-allocated
    activeCount: number                      // Living particles
    
    spawn(config) → Particle | null          // O(1) allocation
    kill(index) → void                       // Swap-and-pop, O(1)
    update(dt) → void                        // Linear scan O(n)
    render(ctx) → void                       // Linear scan O(n)
}
```

Cada partícula tiene: `{ x, y, vx, vy, life, maxLife, color, size, alpha }`. Al morir (life ≤ 0), se intercambia con la última partícula activa y se decrementa `activeCount` — sin splice, sin GC.

---

## 🔀 Máquinas de Estados Finitos (FSM) — Tabla de Transiciones

El sistema de eventos opera mediante una **FSM jerárquica** donde el estado global del overlay determina qué comportamientos están activos.

### Estado Global del Overlay

```
                    ┌─────────┐
                    │  IDLE   │◄──────────────────────────────┐
                    └────┬────┘                               │
                         │                                    │
          ┌──────────────┼──────────────┐                    │
          │              │              │                    │
          ▼              ▼              ▼                    │
    ┌───────────┐ ┌───────────┐ ┌───────────┐              │
    │  EFFECT   │ │  MINIGAME │ │  GLOBAL   │              │
    │  ACTIVE   │ │  ACTIVE   │ │  MODIFIER │              │
    │           │ │           │ │           │              │
    │ • Boom    │ │ • Pelea   │ │ • Speed   │              │
    │ • Tornado │ │ • Pachinko│ │ • Rain    │              │
    │ • Pogo    │ │ • Carrera │ │ • Space   │              │
    │ • Volcan  │ │           │ │           │              │
    │ • Salto   │ └─────┬─────┘ └─────┬─────┘              │
    │ • Thriller│       │             │                     │
    │ • Taiko   │       │             │                     │
    │ • Terrem. │       │ timer       │ timer               │
    │ • Remolin.│       │ expires     │ expires             │
    └─────┬─────┘       │             │                     │
          │             └─────────────┘                     │
          │ timer                 │                          │
          │ expires               │                          │
          └───────────────────────┴──────────────────────────┘
```

### Tabla de Transiciones — Evento "Pelea" (Ejemplo Detallado)

| Estado Actual | Evento | Guard | Acción | Estado Siguiente |
|---|---|---|---|---|
| `IDLE` | `redeem:Pelea` | `entities.length >= 2` | `initFight()` | `FIGHT_SETUP` |
| `IDLE` | `redeem:Pelea` | `entities.length < 2` | `relay.send("No hay suficientes entidades")` | `IDLE` |
| `FIGHT_SETUP` | `timer:3s` | — | `assignTeams(); playSound('fight')` | `FIGHT_ACTIVE` |
| `FIGHT_ACTIVE` | `collision(red, blue)` | `random() < 0.5` | `eliminate(blue); playSound('sword')` | `FIGHT_ACTIVE` |
| `FIGHT_ACTIVE` | `collision(red, blue)` | `random() >= 0.5` | `eliminate(red); playSound('shield')` | `FIGHT_ACTIVE` |
| `FIGHT_ACTIVE` | `team.eliminated(X)` | `remaining(other) > 0` | `declareWinner(); playSound('win')` | `FIGHT_END` |
| `FIGHT_END` | `timer:5s` | — | `cleanup(); restoreEntities()` | `IDLE` |

> Las 17 recompensas tienen tablas de transición similares. Cada una opera como un sub-estado dentro de la HFSM con guards, actions, y timers propios.

---

## 🌐 Pipeline de Internacionalización (i18n) — Traducción en Tiempo Real

### Flujo de Procesamiento de Mensajes

```
Message received via WSS
│
├─ 1. Filtros de exclusión:
│     ├─ message.length < 3          → SKIP (too short)
│     ├─ letterCount < 2             → SKIP (no real text)
│     ├─ startsWithCommand('!')      → SKIP (command, not message)
│     ├─ user === relayNick          → SKIP (own message)
│     └─ user === channel            → ROUTE to ownerPipeline
│
├─ 2a. Pipeline OWNER (canal → inglés):
│     ├─ detectLanguage(text)
│     │   └─ if detected === 'en'    → SKIP (already english)
│     ├─ translate(text, 'en')
│     │   └─ GET https://translate.googleapis.com/translate_a/single
│     │       ?client=gtx&sl=auto&tl=en&dt=t&q={encoded_text}
│     ├─ formatResponse("🌐 {user}: {translation}")
│     └─ relay.send(formatted)
│
├─ 2b. Pipeline VIEWER (viewer → español):
│     ├─ detectLanguage(text)
│     │   └─ if detected === 'es'    → SKIP (already spanish)
│     ├─ translate(text, 'es')
│     │   └─ GET (same endpoint, tl=es)
│     ├─ formatResponse("🌐 {user}: {translation}")
│     └─ relay.send(formatted)
│
└─ 3. Error handling:
      └─ on fetch error → relay.send("⚠️ Traducción no disponible")
```

### Google Translate GTX Endpoint

```
Endpoint: https://translate.googleapis.com/translate_a/single
Method:   GET
Params:
  client = gtx          // Client identifier
  sl     = auto         // Source language (auto-detect)
  tl     = {target}     // Target language code (ISO 639-1)
  dt     = t            // Data type: translation
  q      = {text}       // URL-encoded text to translate

Response: JSON array
  [0][0][0] = translated text
  [0][0][1] = original text
  [2]       = detected source language code
```

> **Rate Limiting:** El endpoint GTX aplica rate limiting dinámico basado en IP. En uso moderado (~1 msg/seg), no se alcanza el límite. Si se excede, la respuesta HTTP cambia a 429 y el pipeline ejecuta el fallback de error.

---

## 🔐 Modelo de Seguridad — OAuth2 Fragment Isolation

### Threat Model

```
┌──────────────────────────────────────────────────┐
│                  TRUST BOUNDARY                   │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │           Static CDN Host                   │ │
│  │                                             │ │
│  │  Serves: index.html, *.ogg                  │ │
│  │  CANNOT access: URI fragment (#...)         │ │
│  │  Access logs contain: path + query only     │ │
│  │  HTTPS enforced (HSTS)                      │ │
│  └─────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────┐ │
│  │        Host Runtime (CEF/Chromium)          │ │
│  │                                             │ │
│  │  CAN access: URI fragment via location.hash │ │
│  │  Runs: JavaScript client-side only          │ │
│  │  Isolation: Chromium sandbox                │ │
│  │  Storage: None (no localStorage/cookies)    │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘

Tokens flow:
  URL config ──hash──→ location.hash ──JS──→ WSS connections
                       (never sent to         (TLS encrypted,
                        HTTP server)           server only)
```

### RFC 3986 §3.5 — Fragment Identifier

> El componente fragment de un URI es procesado exclusivamente por el user-agent después de que la referencia de URI es dereferencida. El fragment **no se envía** en la petición HTTP al servidor. Esto garantiza que los tokens OAuth almacenados en el hash nunca sean expuestos al CDN host.

### Scopes OAuth2 Requeridos

| Token | Scope | Propósito | Riesgo si es filtrado |
|-------|-------|----------|----------------------|
| Primary | `read` | Lectura de mensajes | Bajo — solo lectura |
| Primary | `read:redemptions` | Recibir eventos de recompensas | Bajo — solo lectura |
| Relay | `read` | Detección de idioma | Bajo — solo lectura |
| Relay | `write` | Enviar traducciones | **Medio** — puede enviar mensajes |

> **Principio de mínimo privilegio:** Ningún token tiene scopes de administración o moderación. El peor caso de leak es lectura de mensajes (públicos) o escritura como el relay.

---

## 🔊 Sistema de Audio — Web Audio API con Pooling

### Catálogo de Assets de Audio

| Asset | Archivo | Codec | Trigger | Latencia Target |
|-------|---------|-------|---------|-----------------|
| Welcome ping | `ping.ogg` | Opus@64k | Primer mensaje de usuario nuevo | < 100ms |
| Sword hit | `sword.ogg` | Opus@64k | Colisión en pelea (eliminación) | < 50ms |
| Shield block | `shield.ogg` | Opus@64k | Colisión en pelea (supervivencia) | < 50ms |
| Wilhelm scream | `wilhelmscream.ogg` | Opus@64k | Eliminación dramática | < 50ms |
| Victory | `win.ogg` | Opus@64k | Fin de pelea / pachinko win | < 100ms |
| Fight start | `fight.ogg` | Opus@64k | Inicio de batalla | < 100ms |
| Gold hit | `gold.ogg` | Opus@64k | Acierto en pachinko | < 50ms |
| Bad hit | `bad.ogg` | Opus@64k | Fallo en pachinko | < 50ms |

### Formato de Audio — Codec Pipeline

```
Source audio (any format)
│
├─ ffmpeg transcode:
│   ffmpeg -i input.mp3 -c:a libopus -b:a 64k output.ogg
│
├─ Container: Ogg (.ogg)
├─ Codec: Opus (IETF RFC 6716)
├─ Bitrate: 64 kbps CBR
├─ Sample rate: 48000 Hz (Opus native)
├─ Channels: Mono/Stereo (auto)
│
└─ Justificación:
    • Opus: mejor calidad/bitrate que MP3, Vorbis, AAC a 64k
    • Ogg: container libre, sin patentes
    • 64k: suficiente para SFX cortos, minimiza tamaño
    • Soporte nativo en Chromium (CEF) sin decodificador extra
```

> **Volumen:** Todos los assets se reproducen a `volume = 1.0`. El control de volumen se delega al mixer del host runtime mediante monitorización de audio. Esto permite al operador ajustar el balance sin modificar el código.

---

## 🎁 API de Extensión — Esquema de Recompensas

Las recompensas se mapean por nombre exacto (`reward.title`) a funciones del motor. El matching es **case-sensitive** y **sin normalización Unicode**.

### Registro de Recompensas

| `reward.title` (exacto) | Handler | Duración | Entidades Afectadas | Prioridad FSM |
|---|---|---|---|---|
| `Haz Bailar tus Dibujos` | `handleDance(user)` | 8s | Solo del usuario | Normal |
| `Borrar mi Dibujo` | `handleDelete(user)` | Instant | Solo del usuario | Normal |
| `Velocidad` | `handleSpeed()` | 60s | Todas | Modifier |
| `Lluvia` | `handleRain()` | 30s | Todas | Effect |
| `Pelea` | `handleFight()` | Variable | Todas | Minigame |
| `Boom` | `handleBoom()` | 5s | Todas | Effect |
| `Tornado` | `handleTornado()` | 15s | Todas | Effect |
| `Pogo` | `handlePogo()` | 15s | Todas | Effect |
| `Volcan` | `handleVolcano()` | 20s | Todas | Effect |
| `Salto` | `handleJump()` | 10s | Todas | Effect |
| `Thriller` | `handleThriller()` | 15s | Todas | Effect |
| `Taiko` | `handleTaiko()` | 15s | Todas | Effect |
| `Terremoto` | `handleEarthquake()` | 10s | Todas | Effect |
| `Carrera` | `handleRace()` | 20s | Todas | Minigame |
| `Espacio` | `handleSpace()` | 30s | Todas | Modifier |
| `Remolino` | `handleVortex()` | 15s | Todas | Effect |
| `Pachinko` | `handlePachinko()` | Variable | Todas | Minigame |

> **⚠️ IMPORTANTE:** Los nombres de las recompensas deben coincidir **exactamente** con los strings de la tabla. Un espacio extra, una tilde faltante, o un cambio de mayúscula causará un `reward_not_found` silencioso en el dispatcher.

---

## 💬 Comandos de Protocolo — Parser y Dispatcher

### Anatomía de un Comando

```
Protocol Message:
@badge-info=subscriber/24;badges=subscriber/24;color=#FF4500;
display-name=Viewer123;emotes=;flags=;id=abc-def-ghi;mod=0;
room-id=12345678;subscriber=1;sent-ts=1700000000000;
turbo=0;user-id=87654321;user-type=
:viewer123@host MSG #{channel} :!dibujar A1B2C3

Parsed:
  tags.display-name = "Viewer123"
  tags.mod          = "0"
  tags.subscriber   = "1"
  prefix            = "viewer123@host"
  command           = "MSG"
  channel           = "#{channel}"
  message           = "!dibujar A1B2C3"
```

### Tabla de Comandos

| Comando | Regex | Permisos | Handler | Respuesta |
|---------|-------|----------|---------|-----------|
| `!dibujar {payload}` | `/^!dibujar\s+(.+)$/i` | Todos | `pixelArt.create(user, payload)` | Entidad spawneada en canvas |
| `!test` | `/^!test$/i` | Admin only | `debug.spawnTestEntities(25)` | 25 entidades de prueba |
| `!limpiar` | `/^!limpiar$/i` | Admin only | `entityManager.clearAll()` | Todas las entidades eliminadas |
| `!limpiar {user}` | `/^!limpiar\s+(\w+)$/i` | Admin only | `entityManager.clearByUser(user)` | Entidades del usuario eliminadas |
| `!redes` | `/^!redes$/i` | Todos | `relay.sendLinks('all')` | Links a redes sociales |
| `!discord` | `/^!discord$/i` | Todos | `relay.sendLinks('discord')` | Link a Discord |
| `!instagram` | `/^!instagram$/i` | Todos | `relay.sendLinks('instagram')` | Link a Instagram |
| `!youtube` | `/^!youtube$/i` | Todos | `relay.sendLinks('youtube')` | Link a YouTube |
| `!pixel` | `/^!pixel$/i` | Todos | `relay.sendLinks('pixel')` | Link al editor |

> **Admin Detection:** Se determina mediante `tags['display-name'].toLowerCase() === config.channel`. No se usa el flag `mod` porque el owner no siempre tiene badge de moderador en su propio canal.

---

## 🎨 Formato de Payload del Pixel Art

El comando `!dibujar` acepta un string codificado que representa una grilla de píxeles. Cada carácter del payload mapea a un color de una paleta predefinida.

### Esquema de Codificación

```
Payload: String de caracteres alfanuméricos
Dimensiones: Definidas por el grid del editor (típicamente 16×16)
Encoding: 1 char = 1 pixel
Palette: Mapa char → color hex

Ejemplo:
  !dibujar 0000111100001111000011110000111100001111

  Donde:
    '0' = transparente
    '1' = color #FFFFFF (blanco)
    ... (paleta completa definida en el código)
```

### Pipeline de Renderizado de Sprites

```
Payload string
│
├─ 1. Parse: split into grid rows
├─ 2. Create offscreen canvas (pixel dimensions)
├─ 3. For each pixel:
│     ├─ Lookup color from palette
│     └─ fillRect(x, y, 1, 1)
├─ 4. Scale: drawImage to display size (nearest-neighbor)
├─ 5. Cache: store in entity.sprite
└─ 6. Render: drawImage in main render loop
```

---

## 🚀 Despliegue y Configuración

### Prerequisitos

- Cuenta de GitHub con GitHub Pages habilitado
- Dos cuentas de plataforma (canal + relay)
- Client ID de la aplicación registrada
- Channel ID numérico del canal

### Paso 1 — Obtención de Credenciales OAuth2

Se requieren dos tokens OAuth2 con scopes diferenciados y un Client ID de aplicación registrada. La generación de tokens debe realizarse mediante el flujo OAuth2 Implicit Grant (RFC 6749 §4.2) de la plataforma target.

#### 1.1 Client ID

El Client ID identifica la aplicación registrada ante el Authorization Server (RFC 6749 §2.2). Si se usan generadores de tokens de terceros, el Client ID puede estar integrado automáticamente.

#### 1.2 Token #1 — Primary

**Parámetro:** `channel_token=oauth:XXXXXXXX`

Scopes: lectura de mensajes + lectura de eventos de recompensas.

#### 1.3 Token #2 — Relay

**Parámetro:** `bot_token=oauth:XXXXXXXX`

Scopes: lectura de mensajes + escritura de mensajes.

> **Justificación de separación:** El relay actúa como proxy de escritura para aislar el historial de mensajes del canal de traducciones automáticas y posibles mensajes maliciosos re-traducidos.

#### 1.4 Channel ID

ID numérico del canal, obtenible mediante la API de la plataforma o herramientas de conversión username→ID.

### Paso 2 — Deploy a GitHub Pages

```bash
git init
git add index.html *.ogg
git commit -m "initial deploy"
git remote add origin https://github.com/USER/REPO.git
git push -u origin main

# Settings → Pages → Source: Deploy from branch → main
```

### Paso 3 — Configurar Host Runtime

**Dimensiones:** 1920×1080

**URL Template:**
```
https://{USER}.github.io/{REPO}/#{PARAMS}
```

**Construcción del Fragment:**
```
channel={CHANNEL}&channel_id={CHANNEL_ID}&channel_token=oauth:{TOKEN1}&bot_token=oauth:{TOKEN2}&bot_nick={RELAY_NICK}&pixel=true&traductor=true[&client_id={CLIENT_ID}]
```

### Parámetros de Configuración Runtime

| Parámetro | Tipo | Obligatorio | Default | Descripción |
|---|---|---|---|---|
| `channel` | `string` | ✅ | — | Nombre del canal (lowercase) |
| `channel_id` | `string` | ✅ | — | ID numérico del canal |
| `channel_token` | `string` | ✅ | — | Token OAuth primary (incluir `oauth:`) |
| `bot_token` | `string` | ✅ | — | Token OAuth relay (incluir `oauth:`) |
| `bot_nick` | `string` | ✅ | — | Username de la cuenta relay (lowercase) |
| `client_id` | `string` | ❌ | (built-in) | Client ID de la aplicación registrada |
| `pixel` | `boolean` | ❌ | `true` | Activar subsistema de pixel art |
| `traductor` | `boolean` | ❌ | `true` | Activar subsistema de traducción |

**Configuración del Host:**
- ✅ Activar routing de audio hacia el mixer del host
- ❌ Desactivar shutdown automático cuando no es visible (mantener conexiones WSS activas)

---

## 🔧 Depuración Avanzada

### Acceso a la Consola de Desarrollo

1. Interactuar con el runtime embebido
2. Abrir DevTools: `F12` o `Ctrl+Shift+I`
3. Pestaña **Console**

### Namespaces de Log

| Namespace | Ejemplo de Output | Indica |
|---|---|---|
| `[eventsub]` | `✅ WebSocket conectado` | Estado de la conexión de eventos |
| `[eventsub]` | `🎉 session_id: abc123` | Sesión WSS activa |
| `[eventsub]` | `✅ suscripto a reward events` | Suscripción confirmada |
| `[eventsub]` | `🎯 redemption: Pelea por user` | Evento de recompensa procesado |
| `[eventsub]` | `❌ error al suscribirse` | Falla de autenticación o Client ID inválido |
| `[traductor]` | `🌐 Traducción enviada` | Pipeline de traducción ejecutado |
| `[traductor]` | `⚠️ Error de traducción` | Falla en el endpoint GTX |

### Diagnóstico de Problemas Comunes

| Síntoma | Log Esperado | Causa Probable | Solución |
|---|---|---|---|
| No se reciben eventos | `❌ error al suscribirse` | Token o Client ID inválido/expirado | Regenerar tokens y verificar Client ID |
| No traduce | Sin logs de `[traductor]` | `traductor=false` en URL o falta `bot_token` | Verificar parámetros en el hash |
| Sin sonido | — | Audio routing desactivado en el host | Activar routing de audio |
| Desconexiones frecuentes | `Reconnecting...` | Timeout de keepalive excedido | Verificar estabilidad de red |

---

## 📁 Estructura del Repositorio

```
.
├── index.html              # Monolito SPA (HTML + CSS + JS ~4000 LOC)
├── ping.ogg                # SFX: Welcome notification (Opus@64k)
├── sword.ogg               # SFX: Sword hit (Opus@64k)
├── shield.ogg              # SFX: Shield block (Opus@64k)
├── wilhelmscream.ogg       # SFX: Elimination scream (Opus@64k)
├── win.ogg                 # SFX: Victory fanfare (Opus@64k)
├── fight.ogg               # SFX: Fight initiation (Opus@64k)
├── gold.ogg                # SFX: Pachinko hit (Opus@64k)
├── bad.ogg                 # SFX: Pachinko miss (Opus@64k)
└── README.md               # Este documento
```

---

## 📄 Licencia

Proyecto open source — motor de overlay genérico.

---

<sub>Documentación generada para STRM-OVR/unified v2.x — Motor de overlay en tiempo real con pipeline de renderizado Canvas2D, orquestación multi-socket, motor de física Euler/Verlet, y sistema de traducción bidireccional sobre Google Translate GTX. Arquitectura event-driven microkernel desplegada como SPA estática con autenticación OAuth2 fragment-isolated.</sub>
