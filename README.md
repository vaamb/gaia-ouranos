# Gaia-Ouranos

**Home-built plant growth automation, running in production since 2020.**

**GAIA** (Greenhouse Automation Intuitive App) is the edge node that lives on a
Raspberry Pi next to the plants. **Ouranos** is the server that gathers, archives
and serves what Gaia measures. Together they monitor and control enclosed plant
growth environments, be it greenhouses, terrariums, aquariums, or any chamber where
temperature, humidity, light and CO₂ matter.

🌱 **See it running: [gaia-py.vaamb.dev](https://gaia-py.vaamb.dev/)**, the
instance that has been monitoring my plants for six years. The ecosystem pages
(sensors, actuators, camera), the weather and the calendar are public; nothing
can be changed without logging in. The
[about page](https://gaia-py.vaamb.dev/about) tells the full story.

[![gaia CI](https://img.shields.io/github/actions/workflow/status/vaamb/gaia/test.yml?label=Gaia)](https://github.com/vaamb/gaia/actions/workflows/test.yml)
[![ouranos-core CI](https://img.shields.io/github/actions/workflow/status/vaamb/ouranos-core/test.yml?label=Ouranos-core)](https://github.com/vaamb/ouranos-core/actions/workflows/test.yml)
[![event-dispatcher CI](https://img.shields.io/github/actions/workflow/status/vaamb/event-dispatcher/test.yml?label=event-dispatcher)](https://github.com/vaamb/event-dispatcher/actions/workflows/test.yml)
[![gaia-validators CI](https://img.shields.io/github/actions/workflow/status/vaamb/gaia-validators/test.yml?label=Gaia-validators)](https://github.com/vaamb/gaia-validators/actions/workflows/test.yml)
![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue)
![SvelteKit 2](https://img.shields.io/badge/Svelte-5-orange)

---

## What it does

Gaia-Ouranos started during a PhD in plant biology, to replicate laboratory
phytotrons (growth chambers) at home on a budget. It has grown into a small
distributed system:

* **[Gaia](https://github.com/vaamb/gaia)** runs on one or more Raspberry Pis.
  It reads sensors, drives actuators, follows a light schedule that can track
  the real sunrise and sunset, and keeps temperature and humidity on target with
  a hysteresis PID controller. It works standalone, logs to a local SQLite
  database, and syncs with Ouranos whenever a connection is available. A flaky
  network never causes data loss.
* **[Ouranos-core](https://github.com/vaamb/ouranos-core)** aggregates the data from every Gaia instances, archives 
  it for the long term, and exposes it through a REST and Socket.IO API with JWT 
  authentication and role-based access. It also runs companion services: 
  weather forecasts and sun times, a calendar for grow-cycle events, a wiki for 
  plant care notes, sensor alarms and a system monitor. A plugin SDK lets extra 
  functionality hook into its lifecycle.
* **[Ouranos-frontend](https://github.com/vaamb/ouranos-frontend)** is the SvelteKit dashboard: live-updating sensor 
  graphs, actuator switches with countdown timers, plant health pictures, and a 
  day/night theme that follows the site's own sun rather than the operating system.
* **[Ouranos-chatbot](https://github.com/vaamb/ouranos-chatbot)** is a Telegram
  bot plugin, to check on the plants from a phone without opening the web UI.
* Everything talks through a custom
  **[event dispatcher](https://github.com/vaamb/event-dispatcher)** that runs
  in memory, over RabbitMQ or over Redis, and agrees on a protocol defined by
  the shared Pydantic models of
  **[gaia-validators](https://github.com/vaamb/gaia-validators)**.

---

## Architecture

```
    ┌───────────────────────────────────────────────────────┐
    │                 Raspberry Pi(s)                       │
    │  ┌──────────┐     ┌──────────┐     ┌──────────┐       │
    │  │   Gaia   │     │   Gaia   │     │   Gaia   │  ...  │
    │  └──────────┘     └──────────┘     └──────────┘       │
    │        ▲                ▲                ▲            │
    └────────┼────────────────┼────────────────┼────────────┘
             │                │                │
             └────────────────┼────────────────┘
                              │
                       event-dispatcher
                        (AMQP / Redis)
                              │
                              ▼
         ┌──────────────────────────────────────────┐
         │               Ouranos-core               │
         │  ┌───────────────┐    ┌───────────────┐  │
         │  │  Aggregator   │◄──►│   Webserver   │  │
         │  │  (ingest,     │    │  (FastAPI +   │  │
         │  │   archive,    │    │   Socket.IO)  │  │
         │  │   weather)    │    │      API      │  │
         │  └───────┬───────┘    └───────┬───────┘  │
         │          └──────────┬─────────┘          │
         │             ┌───────▼────────┐           │
         │             │   Database     │           │
         │             │ (SQLite/PgSQL) │           │
         │             └────────────────┘           │
         └─────────────────────┬────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
   ┌──────────▼─────────┐         ┌─────────────▼──────┐
   │  Ouranos-frontend  │         │   Ouranos-chatbot  │
   │  (SvelteKit)       │         │   (Telegram bot)   │
   └────────────────────┘         └────────────────────┘
```

Gaia itself can also reach beyond the Pi's GPIO pins: it speaks a WebSocket
protocol to remote hardware, so microcontrollers (an ESP32 firmware in Rust is
in early beta) can act as sensors and actuators anywhere in the greenhouse.

---

## Repositories

### Applications

| Repo                                                          | Role                                                                                                  | License  |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | -------- |
| [gaia](https://github.com/vaamb/gaia)                         | Automation edge node: runs on a Raspberry Pi, manages sensors, actuators, lights and climate control  | GPL-3.0  |
| [ouranos-core](https://github.com/vaamb/ouranos-core)         | Backend server: aggregates data from Gaia instances, archives it, exposes a REST + Socket.IO API      | LGPL-3.0 |
| [ouranos-frontend](https://github.com/vaamb/ouranos-frontend) | SvelteKit web UI: visualises sensor data, controls ecosystems, displays plant health images           | LGPL-3.0 |
| [ouranos-chatbot](https://github.com/vaamb/ouranos-chatbot)   | Telegram bot plugin: interact with Ouranos from a phone without opening the web UI                    | LGPL-3.0 |

### Libraries

| Repo                                                              | Role                                                                                       | License |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------- |
| [event-dispatcher](https://github.com/vaamb/event-dispatcher)     | Broker-agnostic, Socket.IO-inspired pub/sub, extracted from Gaia, used by every component  | MIT     |
| [gaia-validators](https://github.com/vaamb/gaia-validators)       | Shared Pydantic models: the Gaia ↔ Ouranos data transfer protocol                          | GPL-3.0 |
| [sqlalchemy-wrapper](https://github.com/vaamb/sqlalchemy-wrapper) | Async SQLAlchemy helpers, with multi-database binds, used by Gaia and Ouranos              | MIT     |

Each repository is versioned, tested and released on its own; what has to stay
compatible is the event contract they share, not the version numbers.

---

## Tech stack

| Layer                  | Choices                                                                                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Language & runtime** | Python 3.11+ · asyncio (uvloop on the edge) · Node.js 22+                                                                                                              |
| **Gaia (edge)**        | APScheduler · Pydantic · ruamel.yaml config · websockets · Adafruit Blinka / CircuitPython drivers · OpenCV + numpy OpenCV for plant health images                     |
| **Ouranos (server)**   | FastAPI · Uvicorn · python-socketio · Pydantic · SQLAlchemy 2.0 async + Alembic · SQLite / PostgreSQL / MariaDB · PyJWT + argon2 · aiosmtplib · Jinja2 · a plugin SDK  |
| **Frontend**           | SvelteKit 2 · Svelte 5 (runes) · TypeScript · Socket.IO client · axios                                                                                                 |
| **Messaging**          | event-dispatcher over memory, RabbitMQ (AMQP) or Redis                                                                                                                 |
| **Persistence**        | SQLite on the Pi · SQLite, MariaDB or PostgreSQL on the server · sqlalchemy-wrapper for multi-bind async sessions                                                      |
| **Tooling & CI**       | uv · ruff · ty · pytest + coverage · GitHub Actions (lint, type check and tests on Python 3.11 / 3.12 / 3.13)                                                          |
| **Deployment**         | Raspberry Pi (Zero, 4B) · install/update shell scripts · systemd · NGINX reverse proxy · RabbitMQ                                                                      |

---

## How it grew

Gaia-Ouranos started in 2020 as a single Python script on a Raspberry Pi Zero:
a loop that checked whether the lights were on, read a DHT22 and wrote the
readings to a text file. Everything since has been one decision at a time, each
made when the previous shape stopped fitting.

* **Text file → SQLite → SQLAlchemy.** As the volume of sensor data grew, the
  storage had to grow with it. Moving to an ORM meant less time on raw SQL and
  more on the domain problem.
* **Monolith → Gaia + Ouranos + plugins.** The automation logic and the web server 
  were one program until running both on a Pi Zero became impractical. Splitting 
  them was harder than expected as many objects were shared, and the ones the two
  sides still had to agree on later became `gaia-validators`.
* **Socket.IO → event-dispatcher.** The first inter-process link was Socket.IO,
  which worked but was not made for the job. When the server had to run several
  web workers while running some jobs (archiving, weather) exactly once, a
  proper broker-agnostic pub/sub library became the right tool. Writing it rather
  than adopting one was deliberate: it keeps the API surface small and close to
  what the rest of the system already expected.
* **Flask + Jinja2 → Flask RestX + Vue → FastAPI + Pydantic + SvelteKit.** The
  Flask templates accumulated spaghetti JavaScript to add some interactivity. 
  Vue fixed that but later turned Ouranos into an API, which made the lack of 
  request validation visible. FastAPI + Pydantic addressed it (and taught me 
  asyncio); Svelte replaced Vue once the boilerplate became a burden.
* **GPIO only → WebSocket devices.** To support hardware that cannot sit on a
  Pi's pins, Gaia gained a WebSocket hardware protocol. An ESP32 firmware
  written in Rust is its first implementation.

The result is a system that has been continuously evolving for six years while
staying in daily production use throughout. It has taught me Python, project 
architecture, databases, JavaScript, NGINX, concurrency and RabbitMQ along the way.

---

## Status

Active, and in production at home since 2020, today on a Pi Zero and a Pi 4B,
with several years of archived readings behind the graphs at
[gaia-py.vaamb.dev](https://gaia-py.vaamb.dev/).

Each component is independently versioned and tested. The system as a whole is
considered beta: the core data flow is stable, but APIs may still change.
