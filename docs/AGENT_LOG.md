# AGENT_LOG.md — Conversation Prompts & Engineering Corrections

> Detailed chronological record of primary user inquiries, analytical trajectories, forensic findings, and corrections established during the reverse-engineering and architecture audit of the `espionage-event` platform.

---

## 1. Chronological Prompt Invocations & Discoveries

### Prompt 1: Initial Repository Orientation
- **User Prompt**: Provided GitHub repository context (`https://github.com/Pranav-422/Reverse-Encode`).
- **Objective**: Establish baseline repository context and identify the operational target within the workspace (`d:\hackback\espionage-event`).

---

### Prompt 2: Static Reverse-Engineering & Codebase Audit
- **User Prompt**: Reverse-engineer the codebase without modifying files or running install/build commands. Catalog tech stack, local run instructions, top-level folder map, 10 most important files, and odd files with exact file and line citations.
- **Key Discoveries & Evidence**:
  - Identified Next.js 16.2.1 App Router, React 19.0.0, TypeScript 5, Tailwind CSS 4, and Mongoose 9.2.4 ([package.json:12-29](file:///d:/hackback/espionage-event/package.json#L12-L29)).
  - Identified external dependencies: Piston code execution API ([src/lib/piston.ts:1](file:///d:/hackback/espionage-event/src/lib/piston.ts#L1)), OpenRouter LLM ([src/lib/openrouter.ts:6](file:///d:/hackback/espionage-event/src/lib/openrouter.ts#L6)), Cloudflare Turnstile ([src/lib/captcha.ts:28](file:///d:/hackback/espionage-event/src/lib/captcha.ts#L28)), and SMTP Nodemailer ([src/lib/mailer.ts:8](file:///d:/hackback/espionage-event/src/lib/mailer.ts#L8)).
- **What We Corrected / Clarified**:
  - Found that `save_and_clean.js` is an emergency script containing raw database credentials and destructive `.deleteMany()` operations run during a spam incident ([save_and_clean.js:1-15](file:///d:/hackback/espionage-event/save_and_clean.js#L1-L15)).
  - Found `test-rp.js` is an obsolete Playwright test referencing Razorpay payments that was committed before being added to `.gitignore` ([test-rp.js:30](file:///d:/hackback/espionage-event/test-rp.js#L30), [.gitignore:51](file:///d:/hackback/espionage-event/.gitignore#L51)).
  - Flagged `/src/app/run-code/route.ts` as a misplaced, unauthenticated public proxy to Piston outside the `/api` directory ([src/app/run-code/route.ts:4](file:///d:/hackback/espionage-event/src/app/run-code/route.ts#L4)).

---

### Prompt 3: Round 2 Malpractice & Retake Analysis
- **User Prompt**: Inquire whether a participant can take the Round 2 test multiple times through repeated malpractice.
- **Key Discoveries & Evidence**:
  - Examined the client anti-cheat logic in [src/app/dashboard/round2/page.tsx:140-142](file:///d:/hackback/espionage-event/src/app/dashboard/round2/page.tsx#L140-L142) and [src/app/dashboard/round2/page.tsx:552-583](file:///d:/hackback/espionage-event/src/app/dashboard/round2/page.tsx#L552-L583).
- **What We Corrected / Clarified**:
  - **Correction**: Proved that anti-cheat disqualification is enforced **strictly in client-side React state** (`disqualified: true`). Exceeding the warning threshold never calls a server endpoint or writes to MongoDB.
  - Demonstrated that simply refreshing the browser tab resets the React state (`warnings = 0`, `disqualified = false`) and restores access to the coding editor.
  - Revealed that the admin reset route ([src/app/api/admin/redo-round/route.ts:24-28](file:///d:/hackback/espionage-event/src/app/api/admin/redo-round/route.ts#L24-L28)) clears scores but retains `round2QuestionIds`, meaning retakes receive the identical problem set.

---

### Prompt 4: First-Year Student Explanation & README Divergence
- **User Prompt**: Explain the product in 3 sentences, catalog user roles, list 3–5 core features with file paths, and identify claims in the documentation that do not match the code.
- **What We Corrected / Clarified**:
  - **Documentation vs. Reality Disconnect 1 (Winner Marking)**: README and admin UI claim to record contest winners, but the backend handler ([src/app/api/admin/teams/[id]/winner/route.ts:24](file:///d:/hackback/espionage-event/src/app/api/admin/teams/%5Bid%5D/winner/route.ts#L24)) queries the deprecated `Team` collection, causing 100% of winner-marking calls to fail with 404.
  - **Documentation vs. Reality Disconnect 2 (Volunteer Authentication)**: README claims volunteers authenticate to check in attendees, but while the frontend UI has an optional password input ([src/app/attendance/page.tsx:27](file:///d:/hackback/espionage-event/src/app/attendance/page.tsx#L27)), the backend check-in endpoint ([src/app/api/attendance/route.ts:70-140](file:///d:/hackback/espionage-event/src/app/api/attendance/route.ts#L70-L140)) has zero authentication.
  - **Documentation vs. Reality Disconnect 3 (Certificates)**: Certificate generation hardcodes an old event name `'Housie of Fame'` and queries the empty `Team` collection, resulting in 404 errors ([src/app/api/admin/certificates/route.ts:32-45](file:///d:/hackback/espionage-event/src/app/api/admin/certificates/route.ts#L32-L45)).

---

### Prompt 5: Critical Vulnerability & Reliability Audit
- **User Prompt**: Analyze 13 specific vulnerabilities spanning BOLA, unauthenticated attendance, prompt injection, client timers, broken features, ID collisions, rate limiting, and serverless timeouts, and recommend best fixes.
- **What We Corrected / Clarified**:
  - Formulated a production-grade stateless JWT authentication scheme using HttpOnly cookies to replace unauthenticated email parameter lookups.
  - Designed an atomic MongoDB sequence counter (`Counter.findByIdAndUpdate`) to replace the `Math.random()` 3-digit loop that locks at 900 registrations.
  - Designed an XML boundary tag and multi-turn prompt defense to neutralize prompt injection attacks in OpenRouter AI grading.
  - Restructured the AI evaluation pipeline to run per-participant with `Promise.allSettled()` to prevent 504 serverless timeouts.

---

### Prompt 6: Architecture & Workflow Audit Request
- **User Prompt**: Create an architecture and workflow audit markdown document.
- **What We Corrected / Clarified**:
  - Mapped out the end-to-end architecture flowchart (`graph LR`) with explicit data flow and protocol labels.
  - Audited all 6 operational workflows (Registration, RSVP & Check-in, Round 1 Quiz, Shortlisting, Round 2 Coding, AI Evaluation & Export) with Mermaid sequence diagrams, identifying failure modes at each step.
  - Conducted a forensic state inventory proving that no background job queues (BullMQ/Redis) or active payment gateways exist.

---

### Prompt 7: Production Rebuild Documentation Generation
- **User Prompt**: Generate our team's documentation as Markdown files in `[path to our own repo]/docs/` covering `OBSERVATIONS.md`, `PRD.md`, `ARCHITECTURE.md`, `DATA_MODEL.md`, `API.md`, `GAPS.md`, and `AGENT_LOG.md`.
- **What We Corrected / Clarified**:
  - **Clarified Unspecified Path**: The user prompt included the un-filled template placeholder `[path to our own repo]/docs/`. Interactively confirmed with the user to target `d:\hackback\docs\`.
  - **Clarified Missing Card Brief**: The prompt included the placeholder `[paste the Brief and the Killer Tests from the card]`. Confirmed with the user to mark unstated brief items and killer tests as `"Unknown"` rather than fabricating assumptions, preserving forensic integrity.
  - **No Code Copying Constraint**: Described all schemas, routes, and architectural components purely in words, schema definitions, and conceptual diffs without copying proprietary implementation code from the source repository.
