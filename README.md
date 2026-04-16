# Truth Launcher

<div align="center">

**Secure, managed Minecraft ecosystem with a custom launcher, backend API, admin panel, and client-side mod.**

[![Rust](https://img.shields.io/badge/Rust-000000?style=for-the-badge&logo=rust&logoColor=white)](https://www.rust-lang.org/)
[![Tauri](https://img.shields.io/badge/Tauri-FFC131?style=for-the-badge&logo=tauri&logoColor=black)](https://tauri.app/)
[![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Fabric](https://img.shields.io/badge/Fabric_Mod_Loader-DBD0B4?style=for-the-badge&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAOCAYAAAAfSC3RAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsMAAA7DAcdvqGQAAABhSURBVDhPY2AYBYMBMDIw/GdgYHBE4jMiC4ALQhQBARY+I7oEVBwKkPkMWMUhHIJ+QQHTD7Y4uh+gzsKqA5cAhQ8OgKYFY3VkEI4DqyNQwEAcBVYJdhTsGqAA2x8GAQAMABeIFrnlDgcAAAAASUVORK5CYII=&logoColor=white)](https://fabricmc.net/)

</div>

---

## Overview

MineTruth is a full-stack Minecraft management platform designed to provide a controlled and secure gaming experience. The project follows a **hybrid security model** that combines strict enforcement mechanisms (HWID bans, session heartbeats, IP tracking) with a polished, user-friendly interface.

The ecosystem consists of four integrated components that work together to deliver a complete solution -- from game launching and mod management to real-time player monitoring and administration.

---

## Architecture

```
                          +----------------------+
                          |     Truth Admin      |
                          |  (React + Tailwind)  |
                          +---------+-----------+
                                    |
                                    | REST API
                                    v
+---------------------+   +---------------------+   +---------------------+
|    Truth Launcher   |-->|    Truth Backend    |<--|    MySQL Database   |
|   (Tauri + React)   |   |    (Rust / Axum)    |   |        (SQLx)       |
+---------------------+   +---------------------+   +---------------------+
         |
         | Launches
         v
+---------------------+
|     Truth Client     |
| + AxolotlClient Mod  |
|       (Fabric)       |
+---------------------+
```

---

## Components

### 1. Backend 

The core REST API powering the entire ecosystem, built with **Rust** and **Axum** for maximum performance and safety.

| Category | Details |
|---|---|
| **Language** | Rust (Edition 2021) |
| **Framework** | Axum 0.7 |
| **Database** | MySQL via SQLx 0.7 |
| **Auth** | JWT + Bcrypt password hashing |

**API Routes:**

| Route | Description |
|---|---|
| `/api/auth/*` | Login, register, heartbeat, game lifecycle |
| `/api/mods/*` | Mod listing, CRUD, hash verification |
| `/api/admin/users/*` | User management, analytics, alt detection |
| `/api/admin/bans/*` | HWID ban creation and management |
| `/api/admin/stats/*` | Dashboard stats, active players, peak hours |
| `/api/admin/config/*` | Remote config, polls, announcements |
| `/api/crash/*` | Crash report collection and viewing |

**Key Features:**
- JWT authentication with role-based access control (RBAC)
- HWID-based ban enforcement for anti-cheat
- Session heartbeat tracking for real-time player monitoring
- System info collection (OS, GPU, CPU, RAM, resolution, location)
- Mod dependency resolution and hash verification
- Community polling system
- Crash report analytics and event logging

**Database Tables:** `accounts`, `launcher_sessions`, `hwid_bans`, `approved_mods`, `user_system_info`, `polls`, `poll_votes`, `permissions`, `account_permissions`

---

### 2. Truth Launcher

A cross-platform desktop application built with **Tauri v2** that handles authentication, mod management, and game launching.

| Category | Details |
|---|---|
| **Backend** | Rust (Tauri v2) |
| **Frontend** | React 19 + TypeScript + Vite 7 |
| **Styling** | Vanilla CSS (Dark theme with red/purple accents) |
| **Features** | Discord Rich Presence, System Tray |

**Frontend Components:**
- `Login` -- Authentication screen with HWID extraction
- `ModManager` -- Automatic mod downloading and hash verification
- `LaunchOverlay` -- Game launch progress and status
- `Settings` -- User preferences and configuration
- `PollModal` -- In-launcher community polls
- `WindowControls` -- Custom frameless window controls

**Rust Backend (Tauri):**
- `lib.rs` -- HWID extraction, Tauri commands, system integration
- `downloader.rs` -- Parallel asset/library downloading with hash verification
- `file_manager.rs` -- Game directory management, mod file operations

**Key Features:**
- Automatic Minecraft asset and library downloading
- Mod auto-sync with hash verification against the backend
- HWID extraction and session heartbeat reporting
- Discord Rich Presence integration
- System tray with minimize-to-tray support
- Custom frameless window with native controls
- Auto-close when game starts, auto-reopen when game exits

---

### 3. Admin Panel

A web-based administration dashboard for managing the entire ecosystem.

| Category | Details |
|---|---|
| **Framework** | React 19 + TypeScript + Vite 7 |
| **Styling** | Tailwind CSS v4 (violet/dark theme) |
| **Charts** | Recharts 3.7 |
| **Drag & Drop** | dnd-kit |

**Dashboard Pages:**

| Page | Description |
|---|---|
| **Dashboard** | Overview stats, hourly player chart, quick actions |
| **User Management** | Search, paginate, view user profiles with system info |
| **Ban Management** | HWID-based banning with reason tracking |
| **Mod Management** | Drag-and-drop sortable mod list with versioning |
| **Session Monitor** | Active player sessions with HWID/IP/heartbeat |
| **Crash Logs** | View and filter crash reports from players |
| **Event Logs** | Filterable system event history |
| **Analytics** | System distribution charts (OS, GPU, CPU, RAM, country) |
| **Remote Config** | Announcements, maintenance mode, polls |

**Key Features:**
- Alt account detection via shared HWID/IP matching
- Real-time session monitoring with heartbeat tracking
- System analytics with distribution charts (OS, hardware, location)
- Drag-and-drop mod ordering with dependency management
- Announcement system with image/message templates
- Maintenance mode toggle
- Community poll management with vote tracking

---

### 4. PvP Mod

A fork of [AxolotlClient](https://github.com/AxolotlClient/AxolotlClient-mod) -- a Fabric-based client mod providing HUD modules, PvP enhancements, and quality-of-life features for Minecraft.

| Category | Details |
|---|---|
| **Language** | Java |
| **Mod Loader** | Fabric |
| **Build System** | Gradle (multi-version) |
| **Supported Versions** | 1.8.9, 1.16 (Combat), 1.20.1, 1.21.1, 1.21.4 |

**HUD Modules:**

| Module | Module | Module | Module |
|---|---|---|---|
| FPS | Ping | CPS | Armor |
| Potions | Keystrokes | Speed | Scoreboard |
| Crosshair | Coordinates | BossBar | Arrow Count |
| Hotbar | Memory | Combo | TPS |
| Player Count | Compass | Real Time | Reach Display |

**PvP & Gameplay Features:**
- **Visual** -- Custom Skies, Beacon Beams, Hit Color, Low Fire/Shield, Motion Blur, Custom Block Outlines
- **Utility** -- Freelook, Zoom, Screenshot Utils, Fullbright, Time Changer, TNT Timer, Scrollable Tooltips
- **Social** -- Custom Badges, Nametag customization

---

## Tech Stack Summary

| Component | Backend | Frontend | Database |
|---|---|---|---|
| **Backend API** | Rust + Axum | -- | MySQL (SQLx) |
| **Launcher** | Rust (Tauri v2) | React + TS + Vite | -- |
| **Admin Panel** | -- | React + TS + Tailwind | -- |
| **PvP Mod** | Java (Fabric) | -- | -- |

---

## Getting Started

### Prerequisites

- **Rust** (latest stable)
- **Node.js** (v18+)
- **MySQL** (or XAMPP)
- **Java 17+** and **Gradle** (for the mod)

### Running the Stack

```bash
# 1. Start the backend API
cd minetruth-backend
cargo run

# 2. Start the launcher in dev mode
cd minetruth-launcher
npm install
npm run tauri dev

# 3. Start the admin panel
cd minetruth-admin
npm install
npm run dev

# 4. Build the client mod
cd axolotlclient-mod
./gradlew build
```

### Database Setup

Set up a MySQL database and run the schema:

```bash
mysql -u root < minetruth-backend/dev_schema.sql
```

Default connection string: `mysql://root:@localhost/minetruth`

---

## Project Structure

```
MineTruth/
├── minetruth-backend/          # Rust/Axum REST API
│   ├── src/
│   │   ├── routes/             # API route handlers
│   │   │   ├── auth.rs         # Authentication & sessions
│   │   │   ├── bans.rs         # HWID ban management
│   │   │   ├── mods.rs         # Mod distribution
│   │   │   ├── users.rs        # User management
│   │   │   ├── stats.rs        # Analytics & statistics
│   │   │   ├── crash.rs        # Crash report handling
│   │   │   ├── config.rs       # Remote configuration
│   │   │   └── events.rs       # Event logging
│   │   ├── models/             # Database structs
│   │   └── main.rs             # Entry point & router
│   └── dev_schema.sql          # Database schema
│
├── minetruth-launcher/         # Tauri v2 Desktop App
│   ├── src-tauri/
│   │   └── src/
│   │       ├── lib.rs          # HWID, Tauri commands
│   │       ├── downloader.rs   # Asset downloading
│   │       └── file_manager.rs # File operations
│   └── src/
│       ├── components/         # React UI components
│       ├── services/           # API client
│       ├── contexts/           # Auth context
│       └── App.tsx             # Main application
│
├── minetruth-admin/            # Admin Web Dashboard
│   └── src/
│       ├── components/         # Dashboard, Users, Bans, Mods...
│       ├── services/           # API client (Axios)
│       └── App.tsx             # Router & layout
│
└── axolotlclient-mod/          # Fabric Client Mod (Fork)
    ├── common/                 # Shared code across versions
    ├── 1.8.9/                  # MC 1.8.9 version
    ├── 1.20/                   # MC 1.20.1 version
    ├── 1.21/                   # MC 1.21.1 version
    ├── 1.21.4/                 # MC 1.21.4 version
    └── 1.16_combat-6/          # Combat snapshot version
```

---

## Security Model

Project implements a layered security approach:

- **HWID Binding** -- Each account is tied to a hardware identifier, preventing unauthorized sharing
- **HWID Bans** -- Banned hardware IDs are blocked at login, with optional expiration dates
- **Session Heartbeats** -- Active sessions send periodic heartbeats; stale sessions are automatically cleaned
- **IP Tracking** -- Login IPs are recorded for alt account detection and abuse prevention
- **Alt Detection** -- Admin panel can identify related accounts via shared HWID or IP patterns
- **Mod Integrity** -- SHA-256 hash verification ensures players run approved, unmodified mods
- **RBAC** -- Role-based access control with permission system for admin operations

---

## License

- **Truth Oyun Hizmetleri** (Backend, Launcher, Admin) -- Proprietary
- **AxolotlClient Mod** -- See [axolotlclient-mod/LICENSE](axolotlclient-mod/LICENSE)
