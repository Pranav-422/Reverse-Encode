# OBSERVATIONS.md — Verified Claims About the Original Codebase

> Forensic observations extracted from the reverse-engineering analysis of `dBug-Labs/espionage-event`. Every claim is verified directly against repository source files.

---

## 1. Tech Stack & Infrastructure

- The application is built using Next.js 16.2.1 App Router and React 19.0.0 with TypeScript 5.
  Evidence: package.json:22, package.json:26, package.json:37 [Confirmed]
- Styling is implemented using Tailwind CSS 4 with `@tailwindcss/postcss`.
  Evidence: package.json:14, package.json:28, postcss.config.mjs:3 [Confirmed]
- MongoDB is the primary database, accessed via Mongoose version 9.2.4.
  Evidence: package.json:21, src/lib/mongodb.ts:6-25 [Confirmed]
- Database connections use a global connection cache (`global.mongooseCache`) to avoid reconnecting across hot-reloads.
  Evidence: src/lib/mongodb.ts:14-17 [Confirmed]
- Outbound emails are sent through an SMTP transport using Nodemailer 8.0.2.
  Evidence: package.json:23, src/lib/mailer.ts:8-16 [Confirmed]
- Code editing is provided in-browser using Monaco Editor (`@monaco-editor/react` 4.7.0 and `monaco-editor` 0.55.1).
  Evidence: package.json:13, package.json:20, src/app/dashboard/round2/page.tsx:48 [Confirmed]
- Code compilation and execution is proxied to an external Piston API instance defaulting to `https://emkc.org/api/v2/piston`.
  Evidence: src/lib/piston.ts:1-25, .env.example:14 [Confirmed]
- Automated code evaluation is proxied to an external OpenRouter LLM API instance.
  Evidence: src/lib/openrouter.ts:6-20, src/lib/round2Evaluation.ts:25 [Confirmed]
- Anti-bot human verification is handled via Cloudflare Turnstile server-side siteverify.
  Evidence: src/lib/captcha.ts:28-40, package.json:16 [Confirmed]
- Attendance QR code generation utilizes the `qrcode` library (version 1.5.4).
  Evidence: package.json:25, src/lib/mailer.ts:179 [Confirmed]
- Camera-based venue QR code scanning utilizes `html5-qrcode` (version 2.3.8).
  Evidence: package.json:18, src/app/attendance/page.tsx:20-30 [Confirmed]
- Certificate generation uses `pdf-lib` (version 1.17.1) and `jszip` (version 3.10.1).
  Evidence: package.json:19, package.json:24, src/app/api/admin/certificates/route.ts:4-5 [Confirmed]
- The project is configured for deployment on Vercel serverless infrastructure.
  Evidence: README.md:8 [Confirmed]

---

## 2. Authentication & Authorization

- Administrator authentication relies on a single shared secret string (`ADMIN_PASSWORD`) passed via headers or request bodies.
  Evidence: src/app/api/admin/config/route.ts:10-15, README.md:81 [Confirmed]
- Participant authentication uses an email-based One-Time Password (OTP) verification route.
  Evidence: src/app/api/auth/send-otp/route.ts:18, src/app/api/auth/verify-otp/route.ts:8 [Confirmed]
- In participant OTP verification, the server generates a 32-byte hex token that is returned in JSON but never stored in the database or set as an HTTP cookie.
  Evidence: src/app/api/auth/verify-otp/route.ts:50-61 [Confirmed]
- Participant dashboard sessions are stored in browser `localStorage` under the key `'espionage_session'`.
  Evidence: src/lib/auth.ts:15-25, src/app/dashboard/page.tsx:73 [Confirmed]
- Protected participant routes accept user identity via plain query parameters or body fields without validating any bearer token or cookie.
  Evidence: src/app/api/dashboard/me/route.ts:7-15, src/app/api/round1/submit/route.ts:9-15, src/app/api/round2/submit/route.ts:8-15, src/app/api/round2/execute/route.ts:18 [Confirmed]
- The attendance scanning and check-in endpoints perform zero authentication checks on incoming requests.
  Evidence: src/app/api/attendance/route.ts:70-189 [Confirmed]
- An exposed code execution endpoint exists at `/run-code` with no authentication check.
  Evidence: src/app/run-code/route.ts:4-35 [Confirmed]

---

## 3. Data Models & Domain Logic

- The primary entity representing attendees is `Participant`, storing both solo and duo registration details.
  Evidence: src/models/Participant.ts:12-145 [Confirmed]
- Participant IDs are generated as random 3-digit strings matching the format `ESP-[100-999]`.
  Evidence: src/app/api/register-manual/route.ts:234-242 [Confirmed]
- The schema contains an unused legacy `Team` model with fields for Razorpay payment IDs and team sizes of 2–3.
  Evidence: src/models/Team.ts:17-44, README.md:67 [Confirmed]
- Global stage switches are stored in a singleton `EventConfig` model controlling registration, rounds, and durations.
  Evidence: src/models/EventConfig.ts:6-25 [Confirmed]
- OTP records are stored in an `OTP` model with a 10-minute expiry timestamp.
  Evidence: src/models/OTP.ts:6-18, src/app/api/send-otp/route.ts:55 [Confirmed]
- Question banks are represented by `MCQQuestion` (for Round 1) and `CodingQuestion` (for Round 2).
  Evidence: src/models/MCQQuestion.ts:1-25, src/models/CodingQuestion.ts:1-35 [Confirmed]
- Announcements to contestants are stored in a `Notification` model with broadcast scopes.
  Evidence: src/models/Notification.ts:5-18 [Confirmed]
- Organizing Committee member attendance is recorded in an `Organizer` model.
  Evidence: src/models/Organizer.ts:5-20 [Confirmed]

---

## 4. Assessment Engines & Anti-Cheat

- Round 1 presents 30 multiple-choice questions randomized from the database.
  Evidence: src/app/api/round1/questions/route.ts:15-35 [Confirmed]
- Round 1 exam timers are maintained strictly within client React state (`timeLeft = 45 * 60`).
  Evidence: src/app/dashboard/round1/page.tsx:40-42 [Confirmed]
- Page reloads during Round 1 or Round 2 reset the client countdown timer to the maximum duration.
  Evidence: src/app/dashboard/round1/page.tsx:40, src/app/dashboard/round2/page.tsx:77 [Confirmed]
- Anti-cheat violations (tab switching, window blur, fullscreen exit) increment local React counters and are submitted in client JSON payloads.
  Evidence: src/app/dashboard/round1/page.tsx:43, src/app/api/round1/submit/route.ts:49-50, src/app/dashboard/round2/page.tsx:81, src/app/api/round2/submit/route.ts:30-31 [Confirmed]
- Exceeding the violation threshold in Round 2 sets a client state flag (`disqualified: true`) without notifying the backend or updating the database.
  Evidence: src/app/dashboard/round2/page.tsx:140-142, src/app/dashboard/round2/page.tsx:552-583 [Confirmed]
- Round 2 assigns 3 coding problems (Easy, Medium, Hard) stored in `round2QuestionIds`.
  Evidence: src/lib/round2.ts:45-65, src/app/api/round2/questions/route.ts:25-45 [Confirmed]
- Code execution injects language-specific stdin/stdout wrappers before calling Piston.
  Evidence: src/lib/round2.ts:120-160, src/app/api/round2/execute/route.ts:40-55 [Confirmed]

---

## 5. Administration & Pipelines

- Round 1 shortlisting evaluates participants by descending score (`round1Score: -1`) with no secondary sort keys.
  Evidence: src/app/api/admin/shortlist/route.ts:21-25 [Confirmed]
- The shortlist calculation unconditionally unsets all existing shortlist flags across all participants before updating.
  Evidence: src/app/api/admin/shortlist/route.ts:18 [Confirmed]
- Admin round reset clears scores and timestamps but retains assigned question IDs (`round1QuestionIds`).
  Evidence: src/app/api/admin/redo-round/route.ts:24-28 [Confirmed]
- Admin Round 2 AI evaluation processes every shortlisted participant and problem sequentially via synchronous OpenRouter HTTP calls.
  Evidence: src/app/api/admin/round2-evaluate/route.ts:33-80 [Confirmed]
- Candidate source code is interpolated directly into the OpenRouter prompt string without boundary encapsulation or system prompt role separation.
  Evidence: src/lib/round2Evaluation.ts:100-106, src/lib/openrouter.ts:16-18 [Confirmed]
- Admin winner marking calls `Team.findByIdAndUpdate(id)` while the UI displays records from the `Participant` collection.
  Evidence: src/app/api/admin/teams/[id]/winner/route.ts:24-31, src/app/api/admin/teams/route.ts:17 [Confirmed]
- Admin certificate generation queries the `Team` collection and hardcodes the event title `'Housie of Fame'`.
  Evidence: src/app/api/admin/certificates/route.ts:32-45 [Confirmed]
- Registration confirmation emails are invoked inside `src/app/api/register-manual/route.ts` without `await`.
  Evidence: src/app/api/register-manual/route.ts:276 [Confirmed]
- In-memory rate limiting stores request counts in a local JavaScript `Map` inside the Node process.
  Evidence: src/lib/rate-limit.ts:11-21 [Confirmed]
- Client IP resolution trusts the first IP listed in the `x-forwarded-for` HTTP header.
  Evidence: src/lib/rate-limit.ts:76 [Confirmed]

---

## 6. Odd & Dead Code Files

- `save_and_clean.js` is a standalone CommonJS emergency script that deletes all documents in the `participants` collection except the first five.
  Evidence: save_and_clean.js:1-15 [Confirmed]
- `test-rp.js` is a legacy Playwright test targeting Razorpay payment flows that no longer exist in the application.
  Evidence: test-rp.js:30-34 [Confirmed]
- `export_q_bank.ts` is a utility script that exports the static question bank to JSON files in the workspace root.
  Evidence: export_q_bank.ts:1-25 [Confirmed]
- `stitch_screens/` contains 3 static HTML mockup prototypes using CDN Tailwind CSS that are not integrated into the Next.js runtime.
  Evidence: stitch_screens/landing.html:1, stitch_screens/enrollment.html:1, stitch_screens/dashboard.html:1 [Confirmed]
- `.gitignore` references missing scripts (`check_json.js`, `do_seed.js`, `generate_questions.ts`, `merge_questions.js`) while tracked files like `test-rp.js` remain committed.
  Evidence: .gitignore:47-52 [Confirmed]
