# GAPS.md — Original Flaws & Strategic Rebuild Improvements

---

## 1. Forensic Audit: What the Original Codebase Gets Wrong or Misses

### 1.1 Broken Object-Level Authorization (BOLA) & Missing Participant Sessions
- **The Defect**: The server issues a random 32-byte hex token upon OTP verification, but never stores it or validates it on subsequent calls. All protected endpoints accept user identity via plain query parameters or JSON body fields.
- **Evidence**:
  - `src/app/api/auth/verify-otp/route.ts:50`: Token generated without server persistence.
  - `src/app/api/dashboard/me/route.ts:7-15`: Accepts `?email=...` with zero authentication.
  - `src/app/api/round1/submit/route.ts:9-51`: Submits Round 1 scores on behalf of any unverified email.
  - `src/app/api/round2/submit/route.ts:8-32`: Submits Round 2 completions without verifying caller identity.
  - `src/app/api/round2/execute/route.ts:18`: Executes code and modifies submissions for arbitrary emails.

### 1.2 Unauthenticated Attendance Check-in
- **The Defect**: The attendance scanner frontend asks for a password, but only sends it to `/api/attendance/stats`. The primary attendance endpoints (`GET` lookup and `POST` check-in) have no password checks.
- **Evidence**:
  - `src/app/api/attendance/route.ts:70-189`: GET and POST routes execute without checking `ADMIN_PASSWORD`.
  - `src/app/attendance/page.tsx:27, 115, 172`: Passwords sent only to stats; attendance calls made without auth headers.

### 1.3 Direct Prompt Injection in AI Evaluation
- **The Defect**: Candidate source code is concatenated directly into a single prompt string sent to OpenRouter.
- **Evidence**:
  - `src/lib/round2Evaluation.ts:100-106`: Candidate code `${params.code}` interpolated into prompt string.
  - `src/lib/openrouter.ts:16-18`: Prompt dispatched as a single un-sandboxed user prompt.

### 1.4 Client-Controlled Anti-Cheat & Timers
- **The Defect**: Timers exist only in client React state (`useState`), resetting to maximum duration on page reload. Anti-cheat warning tallies are self-reported by client JSON payloads during submission.
- **Evidence**:
  - `src/app/dashboard/round1/page.tsx:40`: `useState(45 * 60)` resets on refresh.
  - `src/app/dashboard/round2/page.tsx:77`: `useState(90 * 60)` resets on refresh.
  - `src/app/api/round1/submit/route.ts:49-50`: Accepts `warnings` directly from request body.
  - `src/app/api/round2/submit/route.ts:30-31`: Accepts `warnings` directly from request body.
  - `src/app/dashboard/round2/page.tsx:140-142`: Disqualification flag is purely client-side.

### 1.5 Broken Winner Marking (Always 404s)
- **The Defect**: The admin teams dashboard lists attendees from `Participant`, but the winner route attempts to update `Team`.
- **Evidence**:
  - `src/app/api/admin/teams/route.ts:17`: Queries `Participant`.
  - `src/app/api/admin/teams/[id]/winner/route.ts:24-31`: Calls `Team.findByIdAndUpdate(id)` and returns 404.

### 1.6 Broken Certificate Generation
- **The Defect**: Queries the empty `Team` model and hardcodes the title of an older unrelated event (`'Housie of Fame'`).
- **Evidence**:
  - `src/app/api/admin/certificates/route.ts:32-45`: Queries `Team.find(...)` and hardcodes `'Housie of Fame'`.

### 1.7 ID Collision & Infinite Loop on Scale
- **The Defect**: Generates participant IDs using `Math.floor(100 + Math.random() * 900)` in a `while` loop (capped at 900 total combinations). At 900 participants, it enters an infinite loop. Concurrent writes trigger unhandled `E11000` duplicate key errors.
- **Evidence**:
  - `src/app/api/register-manual/route.ts:234-242`: Random generation while loop.
  - `src/app/api/register-manual/route.ts:244`: `Participant.create(...)` fails under race conditions.

### 1.8 Non-Deterministic Shortlisting
- **The Defect**: Sorts strictly by `{ round1Score: -1 }` without tie-breakers, selecting candidates based on arbitrary database storage order. Unconditionally wipes all shortlist flags before running.
- **Evidence**:
  - `src/app/api/admin/shortlist/route.ts:18`: Global blind wipe `updateMany({}, { isShortlisted: false })`.
  - `src/app/api/admin/shortlist/route.ts:21-25`: Single-key sort without secondary tie-breaker.

### 1.9 Redo Round Retains Prior Questions
- **The Defect**: Admin round reset clears scores and timestamps but leaves `round1QuestionIds` intact, giving the retaking student the exact same questions.
- **Evidence**:
  - `src/app/api/admin/redo-round/route.ts:24-28`: Unsets scores but retains question IDs.

### 1.10 In-Memory Rate Limiting & Spoofable IP
- **The Defect**: In-memory `Map` counters are wiped across serverless cold starts. IP resolution blindly trusts the first entry in `x-forwarded-for`.
- **Evidence**:
  - `src/lib/rate-limit.ts:11-21`: Local process Map used for rate limiting.
  - `src/lib/rate-limit.ts:76`: Untrusted header split `headers.get('x-forwarded-for')?.split(',')[0]`.

### 1.11 AI Grading Gateway Timeout
- **The Defect**: Iterates sequentially through all participants and questions in a single synchronous route, taking 90–270 seconds and exceeding serverless execution limits.
- **Evidence**:
  - `src/app/api/admin/round2-evaluate/route.ts:33-80`: Synchronous sequential LLM loop.

### 1.12 Dropped Confirmation Emails
- **The Defect**: `sendRegistrationConfirmationEmail` is invoked without `await`. Serverless freeze tears down pending SMTP sockets.
- **Evidence**:
  - `src/app/api/register-manual/route.ts:276`: Unawaited email dispatch.

### 1.13 Missing Captcha & Rate Limiting on Login OTP
- **The Defect**: `/api/auth/send-otp` lacks the rate limits and Turnstile CAPTCHA present on registration routes, enabling OTP email spamming.
- **Evidence**:
  - `src/app/api/auth/send-otp/route.ts:18-60`: No rate limiting or Turnstile verification.

---

## 2. The 2 Strategic Improvements We Will Build

---

### Improvement 1: Cryptographically Sealed, Server-Enforced Contest Proctoring

#### What We Will Build
1. **Stateless JWT Session Guard**: We will implement signed, HMAC-SHA256 JWT tokens delivered via `HttpOnly`, `SameSite=Lax` cookies upon email OTP verification. Every protected endpoint (`/api/dashboard/me`, `/api/round1/submit`, `/api/round2/execute`, `/api/round2/submit`) derives contestant identity strictly from the verified token payload.
2. **Server-Side Lifecycle & Deadline Verification**: The server records `round1StartedAt` and `round2StartedAt` in MongoDB upon initial question retrieval. On submission, the server computes $\Delta t = \text{currentTime} - \text{startedAt}$ and rejects any submission that exceeds the allocated time limit plus a 60-second network grace window.
3. **Incremental Event-Driven Proctor Logging**: Instead of allowing the client to self-report aggregate violation counts during final submission, all blur, tab-switching, and shortcut violations will be logged incrementally to `/api/proctor/log-event`. Once the server-side violation count breaches the threshold, the participant's session is locked server-side.

#### Why It Matters to the User
- **Eliminates Malicious Impersonation**: Competitors can no longer submit empty assessments or execute failing code on behalf of rival contestants.
- **Enforces Equal Exam Duration**: Prevents candidates from gaining extra time by reloading the browser or manipulating local React state.
- **Ensures Fair Contest Ranking**: Guarantees that honest contestants compete on a level playing field where anti-cheat violations cannot be edited out using browser developer tools.

---

### Improvement 2: Fault-Tolerant, Distributed Scalability & Evaluation Pipeline

#### What We Will Build
1. **Atomic Collision-Free ID Allocation**: Replace the random 3-digit generation loop with an atomic MongoDB sequence counter (`Counter.findByIdAndUpdate('participantId', { $inc: { seq: 1 } })`), issuing clean, zero-collision sequential IDs (`ESP-1001`, `ESP-1002`).
2. **Distributed TTL-Based Rate Limiting**: Migrate rate limiting from in-process Node.js memory to a centralized MongoDB collection with native TTL indexes (`expireAfterSeconds: 0`), preventing rate limit bypass across multi-instance serverless deployments while extracting trusted client IPs via `x-real-ip` / `cf-connecting-ip`.
3. **Chunked, Non-Blocking AI Grading Orchestration**: Redesign the AI evaluation endpoint to evaluate a single candidate per request, running that candidate's 3 problems in parallel with `Promise.allSettled()`. Orchestrate the batch from the admin UI with a progress bar, preventing serverless gateway timeouts.
4. **Guaranteed Email Delivery**: Explicitly `await` outbound SMTP dispatches inside structured error boundaries to ensure serverless containers do not freeze before network packets leave the host.

#### Why It Matters to the User
- **Prevents High-Traffic Registration Outages**: Allows hundreds of simultaneous students to register during event announcements without encountering infinite loops or duplicate key crashes.
- **Guarantees Immediate Attendee Confirmation**: Ensures every student receives their registration details and QR codes reliably.
- **Eliminates Scoring Bottlenecks**: Organizers can execute AI code grading across all contestants without experiencing 504 Gateway Timeouts, allowing winners to be finalized promptly at the venue.
