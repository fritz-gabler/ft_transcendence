# ft_transcendence

A full-stack, real-time multiplayer **Pong** platform built with Fastify, TypeScript, WebSockets, and SQLite. It supports classic 1v1 matches, multiplayer games (up to 8 players), power-ups, lobbies, matchmaking, tournaments, AI opponents, friends, chat, and a full production-grade monitoring stack.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Game Modes & Power-ups](#game-modes--power-ups)
- [Database Schema](#database-schema)
- [Prerequisites](#prerequisites)
- [Environment Variables](#environment-variables)
- [Running the Project](#running-the-project)
  - [Development](#development)
  - [Production](#production)
- [Scripts](#scripts)
- [Monitoring Stack](#monitoring-stack)
- [Vault (Secrets Management)](#vault-secrets-management)
- [API Documentation](#api-documentation)

---

## Features

- **Authentication** — Google OAuth 2.0 login with JWT sessions stored in HTTP-only cookies
- **Classic Pong (1v1)** — fast-paced head-to-head game with configurable physics
- **Multiplayer Pong** — 5-player or 8-player circular-arena pong
- **Power-ups** — 9 collectible power-ups that change gameplay in real time
- **Game Modifiers** — configurable rule layers (time limits, scoring, survival, elimination, arena shrink, etc.)
- **Matchmaking** — automatic queuing for common game modes
- **Lobbies** — create or join public/private lobbies with custom settings
- **Tournaments** — single-elimination bracket tournaments with configurable modes and player counts
- **AI Opponents** — fill empty slots with AI bots at varying skill levels
- **Local Players** — add local (keyboard-sharing) players to a lobby
- **Friends & Blocking** — friend requests, friend list, block list
- **Chat** — real-time WebSocket chat with per-room message history
- **User Profiles** — username, avatar upload, match history
- **Metrics & Monitoring** — Prometheus metrics, Grafana dashboards, Alertmanager with Discord webhooks
- **Swagger API Docs** — auto-generated docs served at `/documentation`

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 22 |
| Backend framework | [Fastify](https://fastify.dev/) v5 (TypeScript) |
| Frontend | Handlebars templates + browser TypeScript + [TailwindCSS](https://tailwindcss.com/) v4 |
| Real-time | WebSockets (`@fastify/websocket`) |
| Authentication | Google OAuth 2.0 (`@fastify/oauth2`) + JWT (`@fastify/jwt`) |
| Database | SQLite (via `sqlite` / `sqlite3`) |
| Reverse proxy | Nginx 1.28 (TLS 1.3, HTTP/2, self-signed cert auto-generated) |
| Monitoring | Prometheus · Grafana · Alertmanager · Node Exporter |
| Secrets management | HashiCorp Vault (optional) |
| Containerisation | Docker & Docker Compose |
| Code quality | Prettier · Husky · lint-staged |

---

## Project Structure

```
ft_transcendence/
├── app/                        # Fastify application (TypeScript)
│   ├── app.ts                  # Application entry point
│   ├── config.ts               # Game mode & tournament registry
│   ├── assets/                 # Source CSS (TailwindCSS input)
│   ├── public/                 # Static files served by Fastify
│   ├── client/                 # Browser-side TypeScript (SPA-style routing)
│   │   ├── router.ts           # Client-side page router
│   │   ├── pong.ts             # Canvas game renderer
│   │   ├── lobby.ts            # Lobby page logic
│   │   ├── matchmaking.ts      # Matchmaking page logic
│   │   ├── tournament.ts       # Tournament page logic
│   │   ├── chat.ts             # Chat widget
│   │   ├── friends.ts          # Friends panel
│   │   └── ...
│   ├── database/
│   │   └── migrations/         # SQL migration files (run in order)
│   ├── interfaces/             # Shared TypeScript interfaces
│   ├── middlewares/            # Fastify lifecycle hooks / middlewares
│   ├── plugins/                # Fastify plugins (auth, JWT, SQLite, metrics…)
│   ├── routes/                 # HTTP & WebSocket route handlers
│   │   ├── login/              # OAuth callback & session routes
│   │   ├── play/               # Play page (lobby list, matchmaking, tournaments)
│   │   ├── games/
│   │   │   ├── lobby/          # Lobby CRUD, WebSocket handler
│   │   │   ├── matchmaking/    # Join / leave matchmaking queue
│   │   │   ├── tournament/     # Tournament CRUD, bracket, WebSocket
│   │   │   └── gameWebsocket.ts# In-game WebSocket endpoint
│   │   ├── profile/            # Profile view & update
│   │   ├── friends/            # Friends list API
│   │   ├── chat/               # Chat WebSocket & history
│   │   ├── users/              # User search / info
│   │   └── image/              # Avatar upload
│   ├── services/
│   │   ├── auth/               # JWT helpers, Google API, new-user creation
│   │   ├── chat/               # Chat service layer
│   │   ├── config/             # Game-mode helpers
│   │   ├── database/           # Database query helpers (users, friends…)
│   │   ├── friends/            # Friend management service
│   │   ├── images/             # Image storage service
│   │   ├── routing/            # Shared routing utilities
│   │   ├── strategy/           # Pluggable strategy registry (RNG samplers…)
│   │   └── games/
│   │       ├── gameBase.ts     # Abstract base game loop
│   │       ├── physicsEngine.ts# Collision detection & resolution
│   │       ├── gameHandler/    # Active-game map, tick runner
│   │       ├── lobby/          # Lobby state machine
│   │       ├── matchMaking/    # Matchmaking queue manager
│   │       ├── tournament/     # Tournament bracket engine
│   │       ├── aiOpponent/     # AI bot implementation
│   │       └── pong/
│   │           ├── pong.ts     # Abstract Pong engine
│   │           ├── gameModes/  # classicPong, multiplayerPong
│   │           ├── gameModifiers/ # Rule modifiers (scoring, time, survival…)
│   │           └── powerUps/   # Collectible power-up implementations
│   ├── templates/              # Handlebars views & layouts
│   ├── types/                  # Shared TypeScript types
│   ├── DockerfileDev           # Dev Docker image (source mounted as volume)
│   └── DockerfileProd          # Production Docker image (compiled build)
├── compose-dev/
│   ├── docker-compose.yml      # Dev stack (app only, port 3000)
│   └── .env.example
├── compose-prod/
│   ├── docker-compose.yml      # Prod stack (app + nginx + monitoring)
│   ├── nginx.conf              # HTTPS reverse proxy config
│   ├── grafana.conf            # Nginx sub-location for Grafana
│   ├── prometheus.conf         # Nginx sub-location for Prometheus
│   └── .env.example
├── monitoring/
│   ├── prometheus/
│   │   ├── prometheus.yml      # Scrape config (app, node-exporter, self)
│   │   └── alert_rules.yml     # Alerting rules
│   ├── alertmanager/
│   │   └── alertmanager.yml    # Discord webhook receiver config
│   └── grafana/
│       └── provisioning/       # Auto-provisioned dashboards & datasources
└── vault/
    └── config/config.hcl       # HashiCorp Vault server config (file backend)
```

---

## Game Modes & Power-ups

### Game Modes

| Mode key | Description | Players |
|---|---|---|
| `classicPong` | Fast 1v1, first to 7 goals in 6 min | 2 |
| `basicPowerUpClassicPong` | Classic with all power-ups enabled | 2 |
| `multiplayerPong5` | Circular arena, 5 players, survival + scoring | 5 |
| `multiplayerPong8` | Circular arena, 8 players | 8 |
| `basicPowerUpMultiplayerPong5` | 5-player with all power-ups | 5 |
| `basicPowerUpMultiplayerPong8` | 8-player with all power-ups | 8 |
| `powerUpMayhem1v1` | 1v1 chaos: smaller paddles, more power-ups, 17-ball multi-ball | 2 |
| `powerUpMayhem5` / `powerUpMayhem8` | Full-chaos multiplayer | 5 / 8 |
| `competitiveClassicPong` | Ranked 1v1, tight physics, first to 11 | 2 |
| `competitiveClassicPongPowerUps` | Ranked 1v1 with power-ups | 2 |
| `competitiveMultiplayerPong` | Ranked 5-player | 5 |
| `competitiveMultiplayerPongPowerUps` | Ranked 5-player with power-ups | 5 |

### Power-ups

| Power-up | Effect |
|---|---|
| `speedBoost` | Temporarily increases ball speed |
| `blinkingBall` | Ball becomes periodically invisible |
| `multiBall` | Splits the ball into multiple balls |
| `bumper` | Places a bumper obstacle on the arena |
| `shooter` | Lets a player shoot a projectile |
| `portals` | Creates two linked teleport portals |
| `speedGate` | Gate that accelerates the ball when passed through |
| `protectedPowerUp` | Shields a player from one power-up hit |
| `bumperShield` | Places a shield wall near a player's goal |

### Game Modifiers

| Modifier | Effect |
|---|---|
| `paceBreaker` | Periodically resets ball speed to prevent stalling |
| `timedStart` | Countdown before the game begins |
| `timedGame` | Ends the game after a fixed duration |
| `scoredGame` | Ends when a player reaches the goal objective |
| `survivalGame` | Eliminated players leave the arena |
| `elimination` | Triggers elimination at a score threshold |
| `arenaShrink` | Arena gradually shrinks over time |
| `goalReset` | Resets ball position after each goal |
| `idleWallBounceAcceleration` | Accelerates ball when bouncing off walls without paddle contact |
| `powerUpSpawner` | Randomly spawns power-ups at configurable intervals |

### Tournament Formats

| Format | Player counts |
|---|---|
| Single elimination | 4 · 8 · 16 · 32 |

---

## Database Schema

The SQLite database is initialised via sequential SQL migrations in `app/database/migrations/`.

| Table | Description |
|---|---|
| `users` | User accounts (id, username, avatar FK) |
| `images` | Stored avatar images |
| `users_blocked` | Block relationships between users |
| `users_friends` | Friend relationships |
| `messages` | Chat messages (linked to chat rooms) |
| `chat_rooms` | Chat room definitions |
| `r_users_chat` | User ↔ chat room membership |
| `tournaments` | Tournament records (id, status, mode) |
| `matches` | Match records (game config snapshot, result, date) |
| `r_users_matches` | Player ↔ match participation (score, result) |
| `r_users_tournament` | Player ↔ tournament participation |

---

## Prerequisites

- **Docker** and **Docker Compose** (v2+)
- A **Google OAuth 2.0** application with an authorised redirect URI:
  - Dev: `http://localhost:3000/login/google/callback`
  - Prod: `https://<your-domain>/login/google/callback`

---

## Environment Variables

Copy the appropriate `.env.example` file and fill in the values.

### Required (all environments)

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth client secret |
| `JWT_SECRET` | Secret used to sign JWT tokens |
| `COOKIE_SECRET` | Secret used to sign cookies |
| `PUBLIC_URL` | Publicly reachable base URL (e.g. `https://localhost` or `http://localhost:3000`) |

### Optional

| Variable | Default | Description |
|---|---|---|
| `NODE_ENV` | `production` | Node environment (`development` enables verbose logging) |
| `DB_PATH` | `./database/db.sqlite` | Path to the SQLite database file |

### Production only

| Variable | Description |
|---|---|
| `DISCORD_WEBHOOK_URL` | Discord incoming webhook for Alertmanager notifications |
| `GRAFANA_ADMIN_USER` | Grafana admin username |
| `GRAFANA_ADMIN_PASSWORD` | Grafana admin password |

---

## Running the Project

### Development

The dev stack mounts the source directory into the container so changes are reflected live.

```bash
cd compose-dev
cp .env.example .env
# fill in .env
docker compose up --build
```

The app is available at **http://localhost:3000**.

### Production

The prod stack adds Nginx (HTTPS), Prometheus, Grafana, Alertmanager, and Node Exporter.

```bash
cd compose-prod
cp .env.example .env
# fill in .env
docker compose up --build -d
```

| Service | URL |
|---|---|
| Application | https://localhost |
| Prometheus | https://localhost:9090 |
| Grafana | https://localhost:4000 |
| Alertmanager | http://localhost:9093 |

> A self-signed TLS certificate is generated automatically on first start.

---

## Scripts

Run these inside the `app/` directory (or inside the dev container):

| Script | Description |
|---|---|
| `npm run start` | Build everything then start the production server |
| `npm run dev` | Build then start in watch mode with hot-reload |
| `npm run build:ts` | Compile server TypeScript |
| `npm run build:client` | Compile browser TypeScript |
| `npm run build:tailwind` | Compile TailwindCSS |
| `npm run lint` | Auto-format all files with Prettier |
| `npm run lint:check` | Check formatting without writing |

---

## Monitoring Stack

The production Compose stack includes a full observability pipeline:

- **Prometheus** scrapes metrics from:
  - The Fastify app at `:3000/metrics` (HTTP request counters, custom game metrics)
  - Node Exporter at `:9100` (host CPU, memory, disk)
  - Prometheus itself
- **Alertmanager** receives firing alerts from Prometheus and forwards them to a **Discord** channel via webhook.
- **Grafana** connects to Prometheus as a datasource; dashboards are auto-provisioned from `monitoring/grafana/provisioning/`.
- **Node Exporter** collects OS-level host metrics.

Data retention for Prometheus is set to **90 days**.

---

## Vault (Secrets Management)

A HashiCorp Vault configuration is provided under `vault/config/config.hcl`. It uses the file storage backend and exposes the Vault UI. This is an optional component for managing secrets in more hardened deployments; it is not wired into the default Compose stacks.

---

## API Documentation

Interactive Swagger/OpenAPI documentation is served by the running application at:

```
http(s)://<host>/documentation
```

