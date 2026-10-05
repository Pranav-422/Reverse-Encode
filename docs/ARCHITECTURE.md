# System Architecture — Espionage Rebuild

---

## 1. Architectural Overview & Component Topology

The rebuilt architecture modernizes the single-tiered Next.js platform into an audited, secure, and distributed-ready system. It establishes strict boundary enforcement between public web interfaces, secure backend route handlers, centralized datastores, and sandboxed third-party services.

```mermaid
graph LR
    subgraph Clients["Presentation Layer (Client Browsers)"]
        P_UI["Participant Web App<br/>(React 19 / Monaco IDE)"]
        V_UI["Volunteer QR Scanner<br/>(html5-qrcode)"]
        A_UI["Admin Operations Console<br/>(Event Controls)"]
    end

    subgraph Edge["Security & Gateway Layer"]
        AUTH_MW["Session & Auth Guard<br/>(Signed JWT / HttpOnly Cookie)"]
        RL_GUARD["Distributed Rate Limiter<br/>(MongoDB TTL Store)"]
    end

    subgraph Core["Application Core (Next.js App Router)"]
        REG_SVC["Registration & ID Service"]
        EXAM_SVC["Exam & Proctoring Engine"]
        CODE_SVC["Sandboxed Execution Service"]
        EVAL_SVC["AI Grading Orchestrator"]
        MAIL_SVC["Guaranteed Email Dispatcher"]
    end

    subgraph Data["Persistence & Storage Layer"]
        DB[("MongoDB Replica Set<br/>(Strict Schemas & Unique Indexes)")]
        DISK["Static Asset Repository<br/>(Certificate PDF Template)"]
    end

    subgraph External["External Cloud Integrations"]
        CF_TURN["Cloudflare Turnstile API"]
        SMTP_RELAY["SMTP Mail Transport"]
        PISTON_API["Piston Execution Sandbox"]
        AI_GATEWAY["OpenRouter LLM API"]
    end

    %% Client traffic into Security Layer
    P_UI -->|"HTTPS + Cookie"| AUTH_MW
    V_UI -->|"HTTPS + Header Secret"| AUTH_MW
    A_UI -->|"HTTPS + Bearer Secret"| AUTH_MW

    %% Security into Application Core
    AUTH_MW --> RL_GUARD
    RL_GUARD --> REG_SVC
    RL_GUARD --> EXAM_SVC
    RL_GUARD --> CODE_SVC
    RL_GUARD --> EVAL_SVC

    %% Core Services to Persistence
    REG_SVC -->|"Atomic Sequence Increments"| DB
    EXAM_SVC -->|"Strict CRUD & Timestamp Verification"| DB
    CODE_SVC -->|"Log Test Results"| DB
    EVAL_SVC -->|"Write JSON Grading Rationales"| DB
    EVAL_SVC -->|"Read Template Buffer"| DISK

    %% Core Services to External Integrations
    REG_SVC -->|"Verify CAPTCHA"| CF_TURN
    REG_SVC -->|"Async Awaited Dispatch"| SMTP_RELAY
    MAIL_SVC -->|"SMTP TLS"| SMTP_RELAY
    CODE_SVC -->|"POST Wrapped Code"| PISTON_API
    EVAL_SVC -->|"POST Sandboxed System Prompt"| AI_GATEWAY
```

---

## 2. Component Interactions & Communication Protocols

### 2.1 Client-to-Server Protocol
- All communications run over HTTPS using JSON payloads.
- **Participant Sessions**: Identified via an `HttpOnly`, `Secure`, `SameSite=Lax` cookie containing a signed HMAC-SHA256 JWT. Bearer authentication is accepted as a fallback for headless clients.
- **Admin & Volunteer Operations**: Authenticated via dedicated request headers (`x-admin-password`, `x-attendance-secret`) verified against server environment configurations.

### 2.2 Server-to-Database Protocol
- The application connects to MongoDB using Mongoose over the native MongoDB Wire Protocol (BSON over TLS).
- Connections are pooled globally within the serverless environment to prevent connection starvation during request surges.

### 2.3 External Integrations
- **Piston Sandbox**: Dispatches user-submitted code inside language-specific wrappers over HTTPS POST to `/api/v2/execute`. Receives structured execution outputs (`stdout`, `stderr`, `exitCode`).
- **OpenRouter LLM**: Dispatches multi-turn evaluation prompts (`system` instruction + isolated `user` candidate code) over HTTPS POST to `/chat/completions`. Receives strict JSON-structured evaluation schemas.
- **Cloudflare Turnstile**: Verifies client anti-bot tokens via server-to-server POST requests to `/siteverify`.
- **Nodemailer SMTP Relay**: Delivers system notices, OTPs, QR codes, and shortlist invitations via encrypted SMTP/TLS (port 587/465).

---

## 3. External Services Inventory

| Service | Protocol / Format | Purpose | Failover / Degraded Behavior |
|---|---|---|---|
| **Cloudflare Turnstile** | HTTPS POST (URL-encoded) | Human verification on registration and OTP forms | If secret is omitted in development, bypass checks gracefully. |
| **SMTP Mail Server** | SMTP over TLS | Dispatching OTPs, attendance QR codes, and shortlist emails | Wrap dispatch in safe error boundaries so failed emails do not abort database transactions. |
| **Piston Code Runner** | HTTPS POST (JSON) | Remote code compilation and execution for C++, Python, Java | Returns structured execution errors (`COMPILE_ERROR`, `TIMEOUT`) to the candidate. |
| **OpenRouter LLM API** | HTTPS POST (JSON) | Semantic grading and partial credit evaluation | Fallback circuit-breaker awards 0 score if code failed all sample test cases or service times out. |

---

## 4. State Management Matrix

| Tier | Storage Mechanism | Data Kept | Lifecycle & Persistence |
|---|---|---|---|
| **Database** | MongoDB Collections | Participants, questions, OTPs, event configuration, proctor events | Durable, persistent across all serverless lifecycles. Indexed by unique constraints. |
| **Distributed Cache** | MongoDB TTL Collections | Rate-limit counters (`RateLimitEntry`) and active session invalidations | Ephemeral with automated TTL index expiration (5–15 minutes). |
| **Server Memory** | Global Node.js variables | Cached database connection handle (`global.mongooseCache`) | Persists across warm serverless invocations; re-instantiated on cold start. |
| **Browser Storage** | HTTP-Only Cookies | Signed participant session JWT (`esp_session`) | Cleared on explicit logout or expires after 12 hours. |
| **Browser RAM** | Client React State | Editor content, live test countdown display, UI view states | Resets on tab refresh; synced against server timestamps on reload. |
| **Disk Assets** | Static Filesystem | `public/Certificate.pdf` | Read-only static template; no dynamic writes to disk occur during runtime. |

---

## 5. Key Architectural Decisions & Rationale

### Decision 1: Signed Stateless JWTs in HttpOnly Cookies vs. In-Memory Tokens
- **Context**: The original code generated random hex tokens in `verify-otp`, returned them in JSON, and saved them to client `localStorage` while backend endpoints checked nothing.
- **Decision**: Issue a signed HMAC-SHA256 JWT inside an `HttpOnly`, `SameSite=Lax` cookie upon OTP verification. Verify the token on every protected route.
- **Rationale**: Eliminates Broken Object-Level Authorization (BOLA), prevents XSS token theft, and avoids server-side session lookup bottlenecks on serverless infrastructure.

### Decision 2: Atomic MongoDB Sequence Counter vs. Random 3-Digit IDs
- **Context**: The original used `Math.floor(100 + Math.random() * 900)` in a `while` loop, hitting an infinite loop when approaching 900 registrations.
- **Decision**: Use a dedicated `counters` collection with atomic `findOneAndUpdate({ _id: 'participantId' }, { $inc: { seq: 1 } })` to issue formatted sequential IDs (`ESP-1001`, `ESP-1002`).
- **Rationale**: Guarantees zero collisions, scales infinitely, and avoids thread contention or database deadlocks.

### Decision 3: Server-Side Start Timestamps vs. Pure Client Countdown
- **Context**: The original kept `timeLeft` in React state. Refreshing the browser reset the 45-minute or 90-minute countdown clock.
- **Decision**: Persist `round1StartedAt` and `round2StartedAt` in the database. Verify elapsed time on submission ($elapsed \le duration + grace$).
- **Rationale**: Prevents client-side timer manipulation and ensures equal testing time for all candidates regardless of browser restarts.

### Decision 4: Chunked Per-Participant AI Grading vs. Bulk Sequential Loop
- **Context**: The original evaluated 30 participants $\times$ 3 questions sequentially in one endpoint, guaranteed to trigger 504 Gateway Timeouts.
- **Decision**: Restructure the admin AI evaluation route to accept `{ participantId }`, evaluating a single participant's problems concurrently using `Promise.allSettled()`. Orchestrate the batch loop from the admin UI with a progress bar.
- **Rationale**: Keeps individual API response times under 5 seconds, easily conforming to serverless execution limits.
