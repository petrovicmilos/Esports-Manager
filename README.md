# 🎮 Esports Manager Web Platform

> A Football Manager-inspired web game for the esports world. Manage your team, sign players, compete in matches, and climb the global rankings.

---

## 📌 Project Overview

Esports Manager is a browser-based management simulation game where users build and manage their own esports team. Inspired by Football Manager, the game replaces traditional sports with competitive esports titles, starting with **CS2**.

Users register as **Managers**, build a roster of imaginary NPC players, compete in simulated matches through a matchmaking system, and grow their team through transfers and player development.

---

## 🧑‍💼 User Roles

| Role | Description |
|------|-------------|
| **Manager** | Standard user. Manages a team, plays matches, buys/sells players |
| **Admin** | Platform administrator. Access to admin dashboard, user management, platform statistics |

> Admin role is assigned manually via database. A user's role is stored as a single `role` field on the `User` table — no separate Admin table needed.

---

## 🎮 Supported Games

| Game | Status |
|------|--------|
| CS2 | ✅ Active |
| Valorant | 🔜 Planned |
| League of Legends | 🔜 Planned |
| Dota 2 | 🔜 Planned |

> New games are added via **Data-Driven Design** — adding a new game record to the database, not changing application code. Each game has its own simulation logic module on the backend.

---

## 🏗️ Architecture

### High-Level System Diagram

```
[ Angular Frontend ]
        ↕ HTTP / REST API
[ Express.js Backend ] ←→ [ Redis ]
        ↕
[ PostgreSQL + Prisma ]
        ↕
[ Socket.io ] ←→ [ Angular Frontend ]
```

### Architecture Pattern

- **Modular Monolith** on the backend — one container, internally organized by domain modules with clear boundaries
- **Monorepo** via Nx Workspace — shared TypeScript types between frontend and backend
- **Containerized** via Docker Compose — 4 containers

### Docker Containers

| Container | Technology | Purpose |
|-----------|------------|---------|
| `frontend` | Angular | Client application |
| `backend` | Express.js + TypeScript | API, game logic, Socket.io |
| `db` | PostgreSQL | Persistent data storage |
| `cache` | Redis | Matchmaking queue, session cache, online status |

---

## 🧱 Backend Domain Modules

```
backend/
└── modules/
    ├── auth/          → Registration, login, JWT, role guard
    ├── game/          → Game list, simulation engine per game
    ├── matchmaking/   → Queue management, ELO matching, match creation
    ├── team/          → Team management, roster, bench
    ├── player/        → NPC player CRUD, attribute management, progression
    ├── transfer/      → Transfer market, offers, dynamic pricing
    ├── stats/         → Rankings, MMR history, performance stats
    ├── socket/        → Socket.io events, rooms, real-time communication
    └── admin/         → User management, platform statistics, ban system
```

Each module follows a strict **Layer Architecture**:

```
Controller → Service → Repository → Prisma (Database)
```

---

## 🗄️ Database Schema (Domain Overview)

### Auth Domain
- **User** — email, password hash, role (MANAGER | ADMIN), avatar, createdAt

### Game Domain
- **Game** — name, slug, isActive (data-driven, extensible)
- **Team** — belongs to one Manager, per Game (one team per game per manager)
- **Player** — imaginary NPC player, belongs to Team, has status (ACTIVE | BENCH | MARKET)
- **PlayerStats** — CS2-specific attributes per Player (aim, utility, positioning, clutch, gameIQ)

### Match Domain
- **Match** — gameId, homeTeamId, awayTeamId, status, map, winnerId, timestamps
- **MatchParticipant** — links User/Team to Match, tracks side (CT | T)
- **Round** — per-round data: winnerId, economy state, round number, events log

### Economy Domain
- **ManagerWallet** — balance per Manager
- **Transfer** — player on market, floor price (algorithm-calculated), status
- **TransferOffer** — Manager's offer on a Transfer (one Transfer → many Offers)

### Stats Domain
- **GameProfile** — MMR, wins, losses, winRate per User per Game
- **MmrHistory** — full MMR change log per match (for charts and history)
- **PlayerPerformance** — per-player per-match stats (rating, kills, assists, etc.)

---

## ⚙️ Core Systems

### 1. ELO / MMR Rating System

Every Manager has an MMR rating per game. Matchmaking uses the ELO algorithm:

**Expected win probability:**
```
E(A) = 1 / (1 + 10^((RB - RA) / 400))
```

**MMR update after match:**
```
New MMR = Old MMR + K × (Result - Expected)

K  = change factor (32 for new players, 16 for experienced)
Result = 1 (win) or 0 (loss)
```

### 2. Matchmaking System

```
Player clicks Find Match
      ↓
Added to Redis Queue
      ↓
Backend checks queue every second
      ↓
MMR Window matching with timeout expansion:
  0–10s  → ±50 MMR
  10–30s → ±100 MMR
  30–60s → ±200 MMR
  60s+   → any opponent
      ↓
Match found → Match record created in PostgreSQL
      ↓
Socket.io room created → both players notified
```

### 3. CS2 Match Simulation Engine

Inspired by Football Manager's tick-based simulation engine:

- Each **round** = one simulation tick
- Backend calculates round outcome based on team/player attributes
- **Probabilistic resolution** using weighted attributes + RNG:

```
Round win probability = f(
  avg(Aim, Utility, Positioning, Clutch, GameIQ),
  side bonus (CT slight advantage),
  economic state (Full Buy vs Eco),
  tactical modifiers
)
```

- Economy system per round: Full Buy / Force Buy / Eco / Semi-Eco
- Side switch at round 12 (CT ↔ T)
- Match ends at 13 round wins (standard) or draw at 12-12
- All round events streamed via **Socket.io** to frontend in real-time

### 4. Player Progression System

Imaginary NPC players grow through matches:

```
Win  → +higher attribute XP
Loss → +lower attribute XP (still learn, just less)

Attributes cap at max value
Growth rate influenced by:
  → Player Potential (hidden stat, like PA in FM)
  → Current match performance
  → Player age
```

### 5. Transfer Market & Dynamic Pricing

**Floor Price Algorithm:**
```
Base Price = f(
  total attribute score,
  player potential,
  player age,
  recent form (last N matches),
  market demand
)
```

- Manager cannot list a player **below** the floor price
- Manager can set price **above** floor price
- Multiple managers can place **TransferOffers** on a listed player
- Seller accepts/rejects offers manually

**Safety Rule:** Manager must keep a minimum of **5 players** at all times (cannot sell below 5-player roster). Players above 5 can be on bench or transfer market.

### 6. Game State Persistence & Recovery

Match state is managed in two layers:

```
Redis (fast, temporary)    → current round state, economy, active events
PostgreSQL (slow, durable) → checkpoint every 3 rounds + final result
```

**Rejoin Flow** (server crash recovery):
```
1. Player returns → sends matchId + JWT token
2. Backend validates: does this player belong to this match?
3. Backend checks match status: "in_progress"?
4. State loaded from last PostgreSQL checkpoint
5. Socket.io room rejoined
6. Other player notified → match resumes
7. If other player doesn't rejoin within timeout → match resolved
```

### 7. Onboarding — New Manager

When a new Manager registers:

```
→ Starting budget: $50,000
→ Starter team: 5 randomly generated NPC players
  ├── Random attributes within beginner range
  ├── Different hidden potential values
  └── Pre-calculated floor prices
→ Manager can immediately start playing or go to transfer market
```

---

## 📡 Real-Time Events (Socket.io)

| Event | Direction | Description |
|-------|-----------|-------------|
| `match:found` | Server → Client | Notifies both players a match was found |
| `match:round_result` | Server → Client | Sends round outcome and updated economy |
| `match:state` | Server → Client | Full match state (used on rejoin) |
| `match:finished` | Server → Client | Final result, MMR changes |
| `player:status` | Server → Client | Online/offline status updates |
| `match:rejoin` | Client → Server | Player requesting to rejoin active match |

---

## 🛡️ Admin Dashboard

- View all registered users (search, filter, ban, IP ban)
- View all active matches in real-time
- View platform statistics (total matches, active users, top managers)
- Charts: daily active users, matches played, top-ranked managers per game
- Manage game list (activate/deactivate games)
- Override/cancel matches if needed

---

## 🧰 Technology Stack

| Layer | Technology |
|-------|------------|
| Frontend | Angular 17+ (Signals, Standalone Components, RxJS) |
| Backend | Express.js + TypeScript |
| Database | PostgreSQL |
| ORM | Prisma |
| Cache / Queue | Redis |
| Real-time | Socket.io |
| Monorepo | Nx Workspace |
| Containerization | Docker + Docker Compose |
| Auth | JWT (Access Token + Refresh Token) |

---

## 📁 Monorepo Structure (Nx)

```
esports-manager/
├── apps/
│   ├── frontend/          → Angular application
│   └── backend/           → Express.js API
├── libs/
│   └── shared/
│       └── types/         → Shared TypeScript interfaces (User, Match, Player...)
├── docker-compose.yml
├── nx.json
└── package.json
```

---

## 🚀 Implementation Roadmap

```
Phase 1 → Nx Monorepo + Docker Compose setup
Phase 2 → Prisma schema + PostgreSQL
Phase 3 → Backend module structure + Layer architecture
Phase 4 → Auth system (register, login, JWT, role guard)
Phase 5 → Team & Player management
Phase 6 → Transfer market
Phase 7 → Matchmaking system (Redis queue + ELO)
Phase 8 → CS2 Simulation engine
Phase 9 → Socket.io real-time integration
Phase 10 → Angular frontend (dashboard, game view, transfer market)
Phase 11 → Admin dashboard
Phase 12 → Docker production setup + final polish
```

---

## 👤 Author

> Portfolio project — built to demonstrate senior full-stack engineering skills across Angular, Express.js, PostgreSQL, Prisma, Redis, and Socket.io.
