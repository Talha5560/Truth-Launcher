# TruthSoft

<div align="center">

**Multi-tenant SaaS for Minecraft server operators — white-label launcher, signed auto-update, license-gated access, customer admin panel, on-prem wrapper.**

[![Rust](https://img.shields.io/badge/Rust-000000?style=for-the-badge&logo=rust&logoColor=white)](https://www.rust-lang.org/)
[![Tauri](https://img.shields.io/badge/Tauri%20v2-FFC131?style=for-the-badge&logo=tauri&logoColor=black)](https://tauri.app/)
[![Axum](https://img.shields.io/badge/Axum-005571?style=for-the-badge&logo=rust&logoColor=white)](https://github.com/tokio-rs/axum)
[![React](https://img.shields.io/badge/React%2019-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=for-the-badge&logo=stripe&logoColor=white)](https://stripe.com/)

</div>

---

## What is TruthSoft

TruthSoft is a SaaS platform that lets independent Minecraft server operators run a **branded launcher + admin console + license enforcement** stack without writing any of it themselves. Operators sign up at `truthsoft.net`, configure their server (CMS preset, branding, plan), and the platform builds, signs, and ships a per-customer launcher that:

- authenticates players against the operator's own CMS (NamelessMC / Azuriom / LeaderOS / Minexon),
- only runs while the operator's TruthSoft license is paid + valid,
- auto-updates from the operator's customer-facing admin panel (`customer-admin.<their-domain>`),
- pipes session/HWID telemetry through a per-customer **wrapper** running on the operator's own VPS — so player data never leaves their infrastructure.

A typical customer onboarding takes ~6 wizard steps and ~10 minutes; from then on the operator manages everything from their admin SPA.

---

## Architecture

```
                                                  Stripe / Cloudflare
                                                          │
                                                          ▼
                              ┌───────────────────────────────────────────────┐
                              │              truthsoft.net                    │
                              │   (Express + React signup / dashboard /       │
                              │    onboarding wizard / Stripe webhooks)       │
                              └─────────────┬─────────────────────────────────┘
                                            │  bridge (HMAC-signed)
                                            ▼
┌────────────────────┐   wss/https   ┌─────────────────────┐   sql   ┌──────────────────┐
│  customer-admin    │◄────────────►│  backend.truthsoft  │◄────────►│  control DB      │
│  .<customer>.net   │              │   .net  (Axum/Rust) │          │  (multi-tenant)  │
│  (admin SPA, SaaS) │              └────────┬───┬────────┘          └──────────────────┘
└────────────────────┘                       │   │
                                  per-tenant │   │ wrapper tunnel
                                       JWT   │   │ (mTLS-ish + Ed25519)
                                             ▼   ▼
                              ┌──────────────────────────────────────┐
                              │   Customer VPS                       │
                              │  ┌───────────────┐  ┌─────────────┐  │
                              │  │ truthsoft     │◄─┤  customer's │  │
                              │  │ wrapper       │  │  CMS DB     │  │
                              │  │ (Rust agent)  │  │ (NamelessMC │  │
                              │  └─────────┬─────┘  │  /Azuriom)  │  │
                              │            │        └─────────────┘  │
                              │            ▼                         │
                              │    [Customer's MC server]            │
                              └──────────────────────────────────────┘
                                             ▲
                                             │ join via signed launcher
                                             │
                              ┌──────────────┴───────────────┐
                              │     Player's machine         │
                              │  Per-customer Tauri launcher │
                              │  (signed, auto-updated)      │
                              └──────────────────────────────┘
```

---

## Components

### 1. `minetruth-backend` — multi-tenant control plane

Rust + Axum HTTP/WebSocket service running on `backend.truthsoft.net`. Resolves tenants by Host header (custom domain or `<slug>.truthsoft.net`), authenticates operator admins + players, enforces license state, and proxies launcher CMS calls through the customer's wrapper tunnel.

| Category | Details |
|---|---|
| Stack | Rust 2021, Axum 0.7, sqlx + MariaDB |
| Auth | JWT (per-tenant signing key) in httpOnly cookie + token-version revocation |
| Surfaces | `/api/v2/auth/*`, `/api/v2/admin/*`, `/api/launcher/*`, `/api/wrapper/{tunnel,bootstrap}`, `/api/bridge/*` |
| Per-tenant | Ed25519 signing key, AES-GCM encrypted secrets, JWT secret, license state cache (72h TTL) |

Key features:
- License enforcement gates *every* admin / player surface — expiry + revocation propagate within ~60 s via the bridge.
- Wrapper tunnel: long-lived WebSocket the customer's wrapper opens to backend; backend forwards CMS queries (auth, perms, mods) over it instead of opening a port to the customer's DB.
- Tunnel bootstrap rate limited per source IP (10 / min) to block tenant-slug enumeration.
- At-rest encryption: license keys, JWT secrets, SMTP passwords stored as AES-256-GCM blobs (master key in env, encrypted-blob-present implies decrypt-must-succeed).

### 2. `truthsoft-web` — operator signup + dashboard

Express + React app at `truthsoft.net` (static SPA in Plesk httpdocs, Express server at `/opt/truthsoft-server`). Handles signup, Stripe checkout, the 6-step onboarding wizard (CnameSetupCard included), and dispatches a per-tenant build + provisioning to the bridge once payment clears.

- Stripe webhook idempotency (`webhook_events.event_id UNIQUE`).
- Email verification + forgot-password (token tables + hashed reset codes).
- SSRF guard on logo/banner URL fields, magic-byte upload validation.
- JWT_SECRET fail-fast on boot.
- Subscription cancel JOIN through `tenants` so the bridge actually revokes on Stripe `customer.subscription.deleted`.

### 3. `minetruth-admin` — customer admin SPA

React 19 + TypeScript + Tailwind v4, served at `customer-admin.truthsoft.net` (single deploy, brand fetched at runtime via CSS variables). The operator's admin team logs in here and gets a panel that's branded for their server but talks to the central backend via the per-tenant resolver.

| Page | Purpose |
|---|---|
| Dashboard | live player count, build status, license state |
| Manage Admins | grant / revoke admin (writes to operator's CMS via wrapper) |
| Schema Settings | CMS / password / TOTP three-axis composer |
| SMTP & Notifications | per-tenant SMTP config + new-device login mail |
| Order Settings | branding, server address, MC version |
| Sessions | live HWIDs / IPs / heartbeat (wrapper-piped) |
| Builds | request rebuild, download history, publish version |
| Subscriptions | plan, billing, license keys |

LicenseGate component blanket-overlays the SPA + force-logouts when backend returns 402.

### 4. `minetruth-launcher` — per-tenant Tauri launcher

Tauri 2 desktop binary, source-rebuilt by the build runner per order. Each binary embeds:

- the operator's brand (logo, banner, colors),
- backend URL pin,
- license key,
- per-tenant Ed25519 public key (server-issued artifacts verified against this).

Player flow: launch → HWID + license probe → if OK, fetch mods + assets (parallel, hash-verified, https-only on tenant content) → launch JVM → minimize. Auto-update plugin polls a signed manifest, downloads the new MSI, verifies signature, replaces the binary.

Hardening this release: zip-extract budget (2 GB / entry-cap 1 GB), mod fetch failure aborts launch, mandatory mod with empty hash hard-errors, JDK download SHA-256 pinned.

### 5. `minetruth-wrapper` — customer-VPS agent

Rust binary the operator installs on their MC VPS. Opens a persistent WebSocket to `tunnel.truthsoft.net` (DNS-only for DDoS surface separation), authenticates with license_key + tenant_slug, then services backend's CMS lookups locally against the operator's MariaDB.

- Wizard-driven install: detects CMS, reads schema, writes wrapper config.
- HWID-bound admin grant on first run (no manual SQL for the operator).
- LicenseRevoked frame → process exit + supervisor will-not-restart.
- (In progress) SPKI pin against `tunnel.truthsoft.net` cert; per-tenant rate limiting on tunnel ops.

### 6. `axolotlclient-mod` (legacy) — single-tenant Fabric mod

Forked AxolotlClient. Pre-SaaS legacy; not part of the multi-tenant pipeline. Kept in-repo for historical builds and as a reference for the upcoming per-tenant MC plugin (force-launcher mode + game-ticket verify, Bant A6).

---

## Security model

Layered, with most layers shipping in named "sprints":

| Layer | Mechanism |
|---|---|
| Tenant isolation | every row has `tenant_id NOT NULL` + 9 FKs; v2 endpoints route through tenant resolver |
| License enforcement | 4/4 entry points blocked: backend login, tunnel handshake, admin SPA gate, launcher startup probe |
| At-rest encryption | AES-256-GCM master key (env), per-secret nonce; encrypted-blob present requires successful decrypt — no silent plaintext fallback |
| Per-tenant signing | Ed25519 keypair per tenant; one tenant's compromise can't forge another tenant's manifests / tickets |
| Auth | httpOnly cookie + token_version bump on password change / forced logout; HWID + IP admin gate; constant-time license key compare |
| Transport | TLS at origin (Plesk LE), Cloudflare in front, SPKI pin in wrapper, https-only mod URLs |
| Anti-abuse | per-IP rate limits (login 10/min, heartbeat 30/min, crash 5/h, wrapper bootstrap 10/min); zip-bomb cap; magic-byte upload check |
| Build runner | dedicated `truthsoft-build` user, `NoNewPrivileges`, `ProtectSystem=strict`, scoped `ReadWritePaths` |
| Webhooks | Stripe signature verify + event-id idempotency; cancel path JOINs through tenants table for revoke fan-out |

Known accepted risks (documented, not yet shipped):
- Cracked/patched launcher could connect to the operator's MC server directly. The fix is per-tenant MC plugin + game-ticket (Bant A6, customer-driven).
- Authenticode signing not applied — Windows SmartScreen warning persists. Operator decision.
- Iyzico / PayTR (Türkiye PSP) checkout returns 501; Stripe is the only enabled provider.

---

## Repository layout

```
Truth Launcher/
├── minetruth-backend/      # Rust/Axum control plane (api.truthsoft.net surface)
├── minetruth-launcher/     # Tauri 2 player launcher (per-tenant binary)
├── minetruth-admin/        # React SPA → customer-admin.truthsoft.net
├── minetruth-wrapper/      # Customer-VPS Rust agent (CMS bridge + HWID grant)
├── truthsoft-web/          # Express + React signup / dashboard / Stripe webhooks
│   ├── server/             # Express + sqlx-style MySQL via mysql2 promise pool
│   └── client/             # Vite + React 19 SPA
├── axolotlclient-mod/      # Legacy single-tenant Fabric mod (pre-SaaS)
├── deploy/                 # Versioned deploy zips, nginx + systemd templates, runbook scripts
└── docs/                   # CUSTOMER-SETUP-{TR,EN}.md, ADMIN-GRANTS-MANUAL-SETUP-*.md
```

---

## Local development

Production assumes the Plesk + Cloudflare stack above; for local hacking each component runs standalone.

```bash
# backend (Rust + sqlx, requires MariaDB)
cd minetruth-backend
cp .env.example .env   # edit DATABASE_URL, BRIDGE_TOKEN, TRUTHSOFT_MASTER_KEY
cargo run

# truthsoft-web (Express + React)
cd truthsoft-web/server && npm i && npm run dev    # :3001
cd truthsoft-web/client && npm i && npm run dev    # :5173

# customer-admin SPA
cd minetruth-admin && npm i && npm run dev         # :5174

# launcher (Tauri 2 — needs webkit2gtk on Linux, or use Windows / cargo-xwin)
cd minetruth-launcher && npm i && npm run tauri dev

# wrapper (customer VPS agent — point at local backend)
cd minetruth-wrapper
cargo run -- --config wrapper-config.toml
```

The first `cargo run` of `minetruth-backend` runs the migration set in `src/migrations.rs` (numbered, idempotent). `dev_schema.sql` is no longer the source of truth; migrations are.

---

## License

Proprietary. © Truth Oyun Hizmetleri. Third-party components retain their own licenses (`axolotlclient-mod/LICENSE` for the Fabric fork, OSS deps via Cargo / npm).
