# Gaia-Ouranos

**GAIA** (Greenhouse Automation Intuitive App) + **Ouranos** — a home automation
system for controlling and monitoring plant growth environments.
Built over 5 years, running in production on a Raspberry Pi at home since 2020.

[![gaia CI](https://img.shields.io/github/actions/workflow/status/vaamb/gaia/test.yml?label=Gaia)](https://github.com/vaamb/gaia/actions/workflows/test.yml)
[![ouranos-core CI](https://img.shields.io/github/actions/workflow/status/vaamb/ouranos-core/test.yml?label=Ouranos-core)](https://github.com/vaamb/ouranos-core/actions/workflows/test.yml)
[![event-dispatcher CI](https://img.shields.io/github/actions/workflow/status/vaamb/event-dispatcher/test.yml?label=event-dispatcher)](https://github.com/vaamb/event-dispatcher/actions/workflows/test.yml)
[![gaia-validators CI](https://img.shields.io/github/actions/workflow/status/vaamb/gaia-validators/test.yml?label=Gaia-validators)](https://github.com/vaamb/gaia-validators/actions/workflows/test.yml)

---

## What it does

Gaia-Ouranos is a project started during a PhD in plant biology to replicate 
laboratory phytotrons on a budget. It monitors and controls enclosed plant 
growth environments — greenhouses, terrariums, aquariums, or any chamber where 
temperature, humidity, light, and CO₂ matter. One or more Raspberry Pis run 
[Gaia](https://github.com/vaamb/gaia), the edge automation node. They
report to [Ouranos](https://github.com/vaamb/ouranos-core), a backend server 
that aggregates and archives sensor data, exposes a REST + WebSocket API, and 
serves a [web UI](https://github.com/vaamb/ouranos-frontend). Everything 
communicates through a custom 
[event dispatcher](https://github.com/vaamb/event-dispatcher) that works 
in-memory, over RabbitMQ, or over Redis.

Beyond raw sensor data, Ouranos provides a set of companion services: weather
forecasts (including sunrise/sunset scheduling), a calendar for tracking
grow-cycle events, a wiki for plant care notes, and a system monitor. Gaia
instances are designed to be resilient — each one logs data locally to a SQLite
database and syncs with Ouranos whenever a connection is available, so a flaky
network never causes data loss.

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

---

## Repositories

### Core

| Repo                                                          | Description                                                                                            |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| [gaia](https://github.com/vaamb/gaia)                         | Automation edge node — runs on a Raspberry Pi, manages sensors, actuators, lights, and climate control |
| [ouranos-core](https://github.com/vaamb/ouranos-core)         | Backend server — aggregates data from Gaia instances, archives it, and exposes a REST + Socket.IO API  |
| [ouranos-frontend](https://github.com/vaamb/ouranos-frontend) | SvelteKit web UI — visualises sensor data, controls ecosystems, displays plant health images           |
| [ouranos-chatbot](https://gitlab.com/eupla/ouranos-chatbot)   | Telegram bot plugin — interact with Ouranos from a phone without opening the web UI                    |

### Libraries

| Repo                                                              | Description                                                                            |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| [event-dispatcher](https://github.com/vaamb/event-dispatcher)     | Broker-agnostic pub/sub event dispatcher — extracted from Gaia, used by all components |
| [gaia-validators](https://github.com/vaamb/gaia-validators)       | Shared Pydantic models for the Gaia ↔ Ouranos protocol                                 |
| [sqlalchemy-wrapper](https://github.com/vaamb/sqlalchemy-wrapper) | Async SQLAlchemy helpers used by Ouranos                                               |

## How it grew

Gaia-Ouranos started as a single Python script on a Raspberry Pi Zero during my PhD
in plant biology — a few lines to toggle a grow light and log DHT22 readings to a
text file. Over the following years it grew into what it is now, one decision at a time.

A few of the forks in the road:

* **Text file → SQLite → SQLAlchemy.** As the volume of sensor data grew, the 
  logging format had to grow with it. Switching to an ORM also meant less time on
  raw SQL and more time on the domain problem.
* **Monolith → Gaia + Ouranos.** The automation logic and the web server were initially
  one program. Running both on a Pi Zero was impractical. Splitting them forced a clean
  protocol boundary — which later became `gaia-validators`.
* **Socket.IO → event-dispatcher.** The first inter-process communication used
  Socket.IO (a working but clearly suboptimal choice). When the server needed to
  support multiple concurrent processes without duplicating data, a proper broker-
  agnostic pub/sub library was the right tool. Building it rather than adopting an
  existing one was a deliberate choice to control the API surface and keep it close to
  what the rest of the system expected.
* **Flask + Jinja2 → Flask RestX + Vue → FastAPI + Pydantic + SvelteKit.** The 
  Flask frontend accumulated spaghetti JavaScript. Vue.js solved the frontend 
  problem but made the lack of API validation visible. FastAPI + Pydantic addressed
  both; Svelte replaced Vue once the boilerplate became a burden.
* **GPIO only → WebSocket devices.** To support hardware that cannot connect directly
  to a Raspberry Pi's GPIO pins, Gaia gained a WebSocket-based hardware protocol.
  An ESP32 firmware written in Rust (early beta) is the first implementation of
  this protocol, allowing microcontrollers to act as remote actuators and sensors.

The result is a system that has been continuously evolving for 5 years while
remaining in daily production use throughout.

---

## Tech stack

**Gaia (edge node)**

* Python 3.11+ · asyncio · APScheduler
* RPi.GPIO · Adafruit CircuitPython drivers (DHT, VEML7700, STEMMA ...) · picamera2
* numpy · OpenCV (plant health image analysis)
* uv · ruff · ty · pytest · pytest-asyncio · coverage

**Ouranos-core (server)**

* Python 3.11+ · FastAPI · Uvicorn · asyncio
* SQLAlchemy 2.0 (async) · Alembic · SQLite / PostgreSQL
* python-socketio · aio-pika · aioredis
* PyJWT · argon2-cffi
* uv · ruff · pytest · pytest-asyncio · coverage

**Ouranos-frontend**

* SvelteKit 2 · Svelte 5 · Vite
* Chart.js · Socket.IO client · axios · Font Awesome
* ESLint · Prettier
* Ouranos plugin SDK

**Ouranos-chatbot**

* Python 3.11+ · python-telegram-bot
* Ouranos plugin SDK

**Shared / infra**

* GitHub Actions CI: lint (ruff) + type check (ty) + tests across Python 3.11 / 3.12 / 3.13
* NGINX · RabbitMQ / Redis (optional message brokers)

---

## Status

Active. Running in production on a Raspberry Pi 3B+ at home since 2020, with
~1100 sensor readings logged per day in 1 ecosystem and 70 months of
archived data.

Each component is independently versioned and tested. The system as a whole is
considered beta: the core data flow is stable, but APIs may still change.
