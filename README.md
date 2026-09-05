# RushMail

> High-throughput outbound email delivery, event-driven interaction telemetry, and security-hardened OAuth token management.

[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-blue.svg?style=flat-square)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-19-61dafb.svg?style=flat-square)](https://react.dev/)
[![Hono](https://img.shields.io/badge/Hono-4.12-E36002.svg?style=flat-square)](https://hono.dev/)
[![Drizzle ORM](https://img.shields.io/badge/Drizzle_ORM-0.45-C5F74F.svg?style=flat-square)](https://orm.drizzle.team/)
[![Turso / LibSQL](https://img.shields.io/badge/Turso-LibSQL-4FF8D2.svg?style=flat-square)](https://turso.tech/)
[![Upstash Redis](https://img.shields.io/badge/Upstash-Redis-00E599.svg?style=flat-square)](https://upstash.com/)
[![TailwindCSS](https://img.shields.io/badge/TailwindCSS-v4-38bdf8.svg?style=flat-square)](https://tailwindcss.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## Demo & Video Walkthrough

Watch a quick walkthrough of the interface, real-time telemetry dispatch, and dashboard:

<!-- DEMO VIDEO START -->
<!-- Replace the URL below with your video link (e.g. YouTube, Vimeo, Loom, or direct .mp4) -->
<p align="center">
  <a href="/public/Mercia-Launch.mp4">
    <img src="/public/Thumbnail_mercia.jpeg" alt="RushMail Demo Video" width="85%" style="border-radius: 8px; border: 1px solid #27272a;" />
  </a>
</p>

```markdown
<!-- Video Embed Example for GitHub -->
<!-- [![Watch RushMail Demo](https://img.youtube.com/vi/YOUR_VIDEO_ID/maxresdefault.jpg)](https://www.youtube.com/watch?v=YOUR_VIDEO_ID) -->
```
<!-- DEMO VIDEO END -->

---

## System Overview

RushMail is an outbound communication platform built for teams and professionals who require real-time visibility into message engagement without compromising deliverability or infrastructure security.

The platform couples direct provider API integration (OAuth 2.0 with Gmail and Microsoft Graph roadmap) with an ultra-low-latency tracking router, authenticated at-rest credential encryption, and an asynchronous telemetry ingestion pipeline.

Instead of relying on third-party tracking extensions that introduce security risks or trigger spam filters, RushMail generates RFC-compliant MIME messages directly through verified mail server endpoints. Incoming read receipts and link interactions are ingested through an edge-compatible Hono server, validated against bot traffic, rate-limited via distributed Redis counters, and persisted into a distributed SQLite database.

---

## Architectural Highlights

- **Direct Provider Dispatch**: Builds RFC 2822 compliant MIME multipart messages and dispatches them straight through official Google Workspace APIs, ensuring full SPF, DKIM, and DMARC alignment under the sender's own domain reputation.
- **Sub-Millisecond Telemetry Ingestion**: Built on the Hono web framework with zero-cold-start edge support, delivering 1x1 transparent tracking pixels and instant HTTP 302 redirects with cache-invalidation headers.
- **Zero-Trust Credential Vault**: OAuth refresh and access tokens are secured at rest using authenticated AES-256-GCM encryption with dynamic 96-bit initialization vectors and authentication tags. Plaintext credentials never persist in database records or log streams.
- **Distributed Sliding-Window Protection**: Multi-tiered rate limiters powered by Upstash Redis pipelines safeguard both ingestion endpoints and outbound quotas (per-minute, per-hour, and per-day caps).
- **Edge Reverse Proxy Verification**: Enforces custom Cloudflare header validation (`X-Mercia-CF-Secret`) and subnet checks to prevent direct origin bypass attacks.
- **Asynchronous Audit and Anomaly Pipeline**: Ingests security alerts, BOLA/IDOR attempt logs, and deliverability warnings asynchronously without blocking the user request-response cycle.

---

## Architecture and Data Flow

```
+-------------------------------------------------------------------------------+
|                                  CLIENT LAYER                                 |
|   React 19 SPA (Vite + Tailwind v4 + TanStack Query + Framer Motion + Tiptap)  |
+---------------------------------------+---------------------------------------+
                                        |
                                        | HTTPS / JSON
                                        v
+-------------------------------------------------------------------------------+
|                              EDGE / API GATEWAY                               |
|   Hono Router + Cloudflare Origin Shield + Secure Headers + CORS Controller   |
+-------------------+-----------------------------------+-----------------------+
                    |                                   |
     [ Outbound Send Route ]               [ Public Telemetry Ingest ]
                    |                                   |
                    v                                   v
+------------------------------------+  +---------------------------------------+
|        CREDENTIAL MANAGER          |  |           TRACKING ROUTER             |
| - AES-256-GCM Decrypt             |  | - 1x1 Transparent GIF Dispatch        |
| - OAuth 2.0 Auto Token Refresh    |  | - High-Speed 302 Redirect Engine      |
| - Zero Memory Buffer Wipe         |  | - User-Agent / Anomaly Filter         |
+-------------------+----------------+  +-------------------+-------------------+
                    |                                       |
                    v                                       v
+------------------------------------+  +---------------------------------------+
|      MIME MESSAGE SYNTHESIZER      |  |         RATE LIMIT & DEFENSE          |
| - RFC 2822 Multipart Encoder       |  | - Upstash Redis Sliding Window        |
| - List-Unsubscribe Header Injector |  | - In-Memory Fallback Eviction Cache   |
| - HTML / URL Rewriter Engine       |  | - Abuse Detection Logger              |
+-------------------+----------------+  +-------------------+-------------------+
                    |                                       |
                    v                                       v
+------------------------------------+  +---------------------------------------+
|        GOOGLE GMAIL API v1         |  |        STORAGE & AUDIT LOGS           |
| Dispatches message via sender      |  | - Turso (LibSQL / SQLite Database)    |
| authenticated OAuth session        |  | - Drizzle ORM Schema & Indexing       |
+------------------------------------+  | - Non-blocking Event Ingestion Queue  |
                                        +---------------------------------------+
```

---

## Technical Deep Dive

### 1. Cryptographic Token Protection (AES-256-GCM)

Third-party OAuth integrations often leave refresh tokens vulnerable in plaintext database columns. RushMail treats OAuth tokens as sensitive credentials:

- Stored tokens use authenticated AES-256-GCM encryption with 256-bit keys derived from environment secrets.
- Every encrypted record uses an isolated 96-bit random initialization vector (IV) generated via cryptographically secure pseudorandom number generators (`crypto.randomBytes`).
- A 128-bit authentication tag is validated upon decryption. If a stored token is tampered with in storage, decryption fails immediately before any outbound API call occurs.
- Memory references containing plaintext access tokens are cleared immediately after the HTTP request to Google finishes.

### 2. MIME Synthesis and Deliverability Preservation

Standard email tracking tools modify message formatting in ways that trigger spam classifiers. RushMail uses a custom message construction pipeline:

- Generates clean multipart/alternative MIME boundaries containing both an unadorned plaintext fallback and an HTML body.
- Appends RFC-compliant `List-Unsubscribe` and `List-Unsubscribe-Post` headers to protect sender score with mailbox providers.
- Transparent tracking pixels are positioned at the document base, accompanied by explicit `Cache-Control: no-store, no-cache, must-revalidate, max-age=0` and `Pragma: no-cache` response headers.
- Links are normalized through URL-safe 21-character short codes with high entropy, avoiding repetitive tracking query parameters.

### 3. Distributed Rate Limiting and Quota Enforcement

To defend against scraping, denial of service, and sender quota exhaustion, the system uses dual-mode rate limiting:

- **Redis Atomic Pipeline**: Uses Upstash Redis pipeline operations (`INCR` + `PTTL`) inside a single round trip to track sliding window counters.
- **Graceful In-Memory Fallback**: When Redis connectivity is unavailable during local development or offline testing, an internal bucket store with scheduled TTL sweepers takes over automatically.
- **Multi-Level Thresholds**:
  - Global IP Limiter: 200 requests per minute per IP.
  - Telemetry Endpoint Limiter: 20 tracking events per minute per IP.
  - Outbound Dispatch Quota: 1 send per minute, 5 sends per hour, and 10 sends per day per authenticated user.

### 4. Zero-Downtime Edge Deployment

The backend server is structured on Hono, allowing the exact same API source code to run in Node.js, Bun, or serverless Vercel Edge functions via `@hono/node-server` and `hono/vercel`. Static assets and API routes are unified under a single reverse proxy configuration, eliminating cross-origin latency penalties.

---

## Tech Stack

### Core Technologies

| Category | Technology | Purpose |
| :--- | :--- | :--- |
| **Runtime & Bundler** | Node.js / Bun, Vite 7 | Fast dev server, sub-second HMR, optimized production builds |
| **Frontend Framework** | React 19, TypeScript | Strict type safety, component architecture, state management |
| **Styling & UI** | Tailwind CSS v4, Framer Motion | Utility-first styling, design system tokens, micro-interactions |
| **Client State** | TanStack React Query v5 | Server state caching, background refetching, optimistic updates |
| **Routing** | Wouter | Minimalist client router with zero overhead |
| **Rich Text Editor** | Tiptap (ProseMirror core) | Extensible WYSIWYG email composer with clean HTML output |
| **Data Visualization** | Recharts | Interactive SVG charts for delivery and open rate trends |
| **Backend Framework** | Hono v4 | Edge-compatible, typed HTTP framework with minimal memory footprint |
| **Database & ORM** | Turso (LibSQL), Drizzle ORM | SQLite-based edge database with type-safe query generation |
| **Cache & Limiting** | Upstash Redis | Serverless distributed sliding window rate limiting |
| **Authentication** | Better Auth, Google OAuth 2.0 | Session management, OAuth credential handling |
| **Validation** | Zod v4 | Runtime schema validation for requests and environment configurations |

---

## Database Schema Design

The database schema is defined in TypeScript using Drizzle ORM with targeted indexes on foreign keys and public lookup identifiers.

```
                  +-------------------------+
                  |          users          |
                  +-------------------------+
                  | id (PK)                 |
                  | email                   |
                  | name                    |
                  +------------+------------+
                               |
            +------------------+------------------+
            | 1:N                                 | 1:N
            v                                     v
+-------------------------+           +-------------------------+
|         emails          |           |         resumes         |
+-------------------------+           +-------------------------+
| id (PK)                 |           | id (PK)                 |
| user_id (FK -> users)   |           | user_id (FK -> users)   |
| to                      |           | filename                |
| subject                 |           | url                     |
| public_tracking_id (UQ) |           | public_tracking_id (UQ) |
| total_opens             |           | total_views             |
| unique_opens            |           | total_downloads         |
| engagement_score        |           +------------+------------+
+------------+------------+                        |
             |                                     |
             | 1:N                                 | 1:N
             v                                     v
+-------------------------+           +-------------------------+
|      email_events       |           |      tracked_links      |
+-------------------------+           +-------------------------+
| id (PK)                 |           | id (PK)                 |
| email_id (FK -> emails) |           | email_id (FK -> emails) |
| user_id (FK -> users)   |           | user_id (FK -> users)   |
| type (open/click/view)  |           | original_url            |
| ip_address, user_agent  |           | short_code (UQ)         |
| is_suspicious (bool)    |           | click_count             |
+-------------------------+           +-------------------------+
```

### Key Tables

1. **`emails`**: Tracks message metadata, recipient information, delivery status, timestamp markers, and indexed public tracking identifiers.
2. **`email_events`**: Immutable event stream recording each interaction event with client telemetry (browser, operating system, device family, IP address, bot flags).
3. **`tracked_links`**: Maps 21-character short codes to destination target URLs with atomic increment counters.
4. **`resumes`**: Manages tracked document links with dedicated counters for views and downloads.
5. **`audit_logs`**: Asynchronous security trail capturing administrative actions, dispatch attempts, and status results.
6. **`security_events`**: Anomaly detection log for rate limit breaches, potential IDOR attempts, and origin bypass flags.

---

## Security Architecture

```
[ Incoming Request ]
        |
        v
[ Cloudflare Origin Protection ]  ---> Fails? -> 403 Forbidden (Audit Logged)
        |
        v
[ Security Headers Middleware ]
  - X-Frame-Options: DENY
  - X-Content-Type-Options: nosniff
  - Strict-Transport-Security: max-age=31536000
  - Content-Security-Policy (Restricted connect / script sources)
        |
        v
[ Sliding-Window Rate Limiter ]   ---> Exceeded? -> 429 Too Many Requests
        |
        v
[ Authenticated Session Guard ]  ---> Invalid? -> 401 Unauthorized
        |
        v
[ Scoped Resource Execution ]     ---> Cross-tenant access? -> 404 / BOLA Logged
```

- **Origin Cloaking**: Verifies upstream proxy headers so that the application server cannot be queried directly over the open internet.
- **Broken Object Level Authorization (BOLA) Defense**: All mutating queries strictly bind resource IDs to the active session user ID at the database level.
- **Zero Raw Error Leaks**: Production error handlers intercept unhandled exceptions, strip call stacks, log metadata asynchronously, and return sanitized JSON responses to the client.

---

## REST API Specification

### Authentication & Health
| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/health` | Health probe returning service readiness | No |
| `ALL` | `/api/auth/*` | Better Auth session and OAuth endpoints | No |

### Outbound Messaging
| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/emails` | List all dispatched emails with aggregated metrics | Yes |
| `GET` | `/api/emails/:id` | Get detailed event history and tracked links for an email | Yes |
| `POST` | `/api/emails/send` | Compile MIME, inject tracking, and dispatch via Gmail | Yes (Quota Enforced) |
| `POST` | `/api/emails/draft` | Save message composition state | Yes |

### Real-Time Ingestion
| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/track/open/:id` | Serves 1x1 transparent GIF and records open telemetry | No |
| `GET` | `/api/track/click/:code` | Records click event and issues 302 redirect | No |
| `GET` | `/api/track/resume/:id` | Telemetry capture for tracked portfolio/resume links | No |

### Analytics & Resources
| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/analytics` | Fetch engagement rates, interaction ratios, and activity | Yes |
| `GET` | `/api/notifications` | Fetch unread user engagement notifications | Yes |
| `GET` | `/api/templates` | Fetch reusable email templates with historical rates | Yes |
| `POST` | `/api/templates` | Create or update an email template | Yes |
| `GET` | `/api/resumes` | List tracked assets and view counts | Yes |

---

## Local Development & Setup

### Prerequisites

- Node.js 20+ or Bun 1.1+
- Package manager: `bun`, `pnpm`, or `npm`
- Turso database account (or local LibSQL file)
- Google Cloud Console Project (with Gmail API enabled)
- Upstash Redis instance (optional for local dev, uses in-memory store if unset)

### 1. Clone the Repository

```bash
git clone https://github.com/Rushu-Tushu/RushEmail.git
cd RushEmail/rushmail
```

### 2. Install Dependencies

```bash
# Using Bun (recommended)
bun install

# Or using pnpm
pnpm install

# Or using npm
npm install
```

### 3. Environment Configuration

Create a `.env` file in the `rushmail` directory based on the template:

```bash
cp .env.template .env
```

Populate the required environment variables:

```env
# Application Settings
NODE_ENV=development
WEBSITE_URL=http://localhost:4200

# Better Auth Secret
BETTER_AUTH_SECRET=your_32_character_random_auth_secret

# Database (Turso / LibSQL)
DATABASE_URL=libsql://your-database.turso.io
DATABASE_AUTH_TOKEN=your_turso_auth_token

# Google OAuth Credentials (Gmail API)
GOOGLE_CLIENT_ID=your_google_client_id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your_google_client_secret

# Token Vault Encryption (Must be 64-character hex string / 32 bytes)
ENCRYPTION_KEY=0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef

# Distributed Rate Limiter (Optional for local dev)
UPSTASH_REDIS_REST_URL=https://your-redis.upstash.io
UPSTASH_REDIS_REST_TOKEN=your_upstash_token

# Cloudflare Origin Guard (Optional for local dev)
CLOUDFLARE_SECRET=your_proxy_shared_secret
```

> **Tip on Generating `ENCRYPTION_KEY`**:
> You can generate a valid 64-character hex key via Node:
> ```bash
> node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
> ```

### 4. Database Schema Push

Synchronize your schema with your Turso database:

```bash
# Push schema changes directly
bun run db:push

# Or open Drizzle Studio for visual database browsing
bun run db:studio
```

### 5. Launch Development Server

```bash
bun run dev
```

The application will start on `http://localhost:4200`. The Vite development server automatically mounts the Hono backend API using the internal development plugin.

---

## Directory Layout

```
rushmail/
├── api/                    # Serverless entry point (Vercel Edge handler)
│   └── index.ts
├── src/
│   ├── api/                # Hono Backend Application
│   │   ├── database/       # Drizzle schema definitions and db connection
│   │   │   ├── schema.ts   # Core business tables (emails, events, links)
│   │   │   └── auth-schema.ts # Better Auth user/account tables
│   │   ├── middleware/     # Rate limiting, auth guard, and security headers
│   │   ├── routes/         # Modular Hono API route controllers
│   │   │   ├── emails.ts   # MIME synthesis and dispatch pipeline
│   │   │   ├── tracking.ts # Pixel and redirect telemetry handlers
│   │   │   ├── analytics.ts# Aggregated statistics and timeseries queries
│   │   │   └── oauth.ts    # Provider integration handlers
│   │   ├── utils/          # AES-256-GCM crypto and audit logger
│   │   ├── auth.ts         # Better Auth initialization
│   │   └── index.ts        # App setup, CORS, and middleware pipeline
│   └── web/                # React 19 Single Page Application
│       ├── components/     # UI primitives, dialogs, and navigation
│       ├── hooks/          # Custom React hooks
│       ├── lib/            # Auth client and client-side utilities
│       ├── pages/          # App views (Dashboard, Compose, Analytics, etc.)
│       ├── styles.css      # CSS variables and styling foundations
│       ├── app.tsx         # Route switches and TanStack Query provider
│       └── main.tsx        # React DOM entry point
├── drizzle.config.ts       # Drizzle ORM generation settings
├── vite.config.ts          # Vite build config with Hono dev plugin
├── tailwind.config.ts      # Tailwind configuration
└── package.json            # Scripts and workspace dependencies
```

---

## Engineering Decisions & Trade-Offs

1. **Hono over Express or NestJS**:
   Hono provides web standard `Request` and `Response` interfaces with zero external runtime dependencies. This allows the backend to be deployed anywhere (Node.js, Bun, Cloudflare Workers, or Vercel Serverless) without rewriting controller logic.

2. **Drizzle ORM over Prisma**:
   Drizzle operates with negligible query overhead, generates raw SQL queries without engine binary bloat, and provides first-class TypeScript inference that keeps cold start latency minimal in serverless environments.

3. **In-Memory Token Erasure**:
   OAuth access tokens retrieved during outbound email sends are kept strictly within function scope and explicitly overwritten after request execution, reducing exposure during process dumps or memory leaks.

4. **Synchronous Redirects with Asynchronous Ingestion**:
   When a recipient clicks a tracked link, the HTTP 302 redirect response is returned immediately to ensure a smooth user experience. The telemetry database write is handled asynchronously without blocking the redirect thread.

---

## Roadmap

- [ ] Multi-provider support via Microsoft Graph API for Outlook / Office 365 accounts.
- [ ] Distributed event streaming queue (Kafka / SQS) for high-volume telemetry ingestion spikes.
- [ ] Webhook notification system for CRM sync (HubSpot, Salesforce).
- [ ] DKIM and custom tracking domain CNAME self-service verification wizard.
- [ ] Multi-tenant workspace teams with role-based access control (RBAC).

---

## License

This project is licensed under the [MIT License](LICENSE).
