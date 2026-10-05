# API Specification — Espionage Rebuild

---

## 1. Authentication Schemes

- **Public Endpoints**: Accessible without credentials; rate-limited by IP and email.
- **Participant Protected Endpoints**: Require a valid signed JWT passed in an `HttpOnly` cookie (`esp_session`) or `Authorization: Bearer <token>`.
- **Volunteer Endpoints**: Require the request header `x-attendance-secret` matching `ADMIN_PASSWORD` or `VOLUNTEER_PASSWORD`.
- **Admin Endpoints**: Require `x-admin-password` or `Authorization: Bearer <ADMIN_PASSWORD>`.

---

## 2. Public & Authentication Endpoints

### 2.1 Request Registration OTP
- **Method**: `POST`
- **Path**: `/api/send-otp`
- **Access**: Public
- **Input (JSON)**:
  - `email` (string, required)
  - `captchaToken` (string, optional/required in production)
- **Output (JSON)**:
  - `{ success: true, message: "OTP sent successfully." }`
- **Error Cases**:
  - `400 Bad Request`: Email missing or invalid format.
  - `400 Bad Request`: Turnstile CAPTCHA validation failed.
  - `429 Too Many Requests`: Rate limit exceeded for this IP or email address.
  - `500 Server Error`: SMTP transport failure.

### 2.2 Verify Registration OTP
- **Method**: `POST`
- **Path**: `/api/verify-otp`
- **Access**: Public
- **Input (JSON)**:
  - `email` (string, required)
  - `otp` (string, required, 6 digits)
- **Output (JSON)**:
  - `{ success: true, verificationToken: string }`
- **Error Cases**:
  - `400 Bad Request`: Missing email or OTP code.
  - `401 Unauthorized`: Invalid, incorrect, or expired OTP code.
  - `429 Too Many Requests`: Exceeded 5 verification attempts within 15 minutes.

### 2.3 Submit Registration
- **Method**: `POST`
- **Path**: `/api/register-manual`
- **Access**: Public
- **Input (JSON)**:
  - `participant`: `{ name, email, collegeEmail, regNo, phone }` (required)
  - `teamType`: `'solo' | 'duo'` (required)
  - `partner`: `{ name, email, collegeEmail, regNo, phone }` (required if duo)
  - `verificationToken`: string (required)
  - `captchaToken`: string (optional/required in production)
  - `honeypot`: string (must be empty)
- **Output (JSON)**:
  - `{ success: true, participantId: "ESP-1001" }`
- **Error Cases**:
  - `400 Bad Request`: Honeypot triggered, CAPTCHA failed, or missing required fields.
  - `403 Forbidden`: Public registration currently paused in `EventConfig`.
  - `409 Conflict`: Email or college registration number already registered.
  - `500 Server Error`: Atomic sequence increment or database insert error.

### 2.4 Confirm RSVP
- **Method**: `GET`
- **Path**: `/api/rsvp?token={rsvpToken}`
- **Access**: Public (Token-authenticated)
- **Output (JSON / Redirect)**:
  - Redirects to `/rsvp?status=confirmed` with confirmed participant metadata.
- **Error Cases**:
  - `400 Bad Request`: Missing token parameter.
  - `404 Not Found`: Invalid or expired RSVP token.

### 2.5 Request Participant Login OTP
- **Method**: `POST`
- **Path**: `/api/auth/send-otp`
- **Access**: Public
- **Input (JSON)**:
  - `email` (string, required)
  - `captchaToken` (string, required in production)
- **Output (JSON)**:
  - `{ success: true, message: "Login OTP dispatched." }`
- **Error Cases**:
  - `400 Bad Request`: CAPTCHA failed or missing email.
  - `403 Forbidden`: Participant login currently paused by administrator.
  - `404 Not Found`: No RSVP-confirmed registration exists for this email address.
  - `429 Too Many Requests`: Exceeded 3 OTP requests within 10 minutes.

### 2.6 Verify Login OTP & Issue Session
- **Method**: `POST`
- **Path**: `/api/auth/verify-otp`
- **Access**: Public
- **Input (JSON)**:
  - `email` (string, required)
  - `otp` (string, required)
- **Output (JSON + Cookie)**:
  - Sets `Set-Cookie: esp_session=<JWT>; HttpOnly; Secure; SameSite=Lax; Path=/`
  - Body: `{ success: true, session: { participantId, name, email, isShortlisted } }`
- **Error Cases**:
  - `401 Unauthorized`: Invalid or expired OTP code.
  - `404 Not Found`: Participant record not found or RSVP unconfirmed.

---

## 3. Participant Protected Endpoints

### 3.1 Fetch Current Participant Profile
- **Method**: `GET`
- **Path**: `/api/dashboard/me`
- **Access**: Authenticated Participant (`esp_session`)
- **Input**: None (Identity derived strictly from session token)
- **Output (JSON)**:
  - `{ participant: { participantId, name, email, round1Score, isShortlisted, round2Score, attendanceRound1, attendanceRound2 } }`
- **Error Cases**:
  - `401 Unauthorized`: Missing, invalid, or expired session cookie.
  - `404 Not Found`: Participant record missing from database.

### 3.2 Fetch Round 1 Questions
- **Method**: `GET`
- **Path**: `/api/round1/questions`
- **Access**: Authenticated Participant
- **Input**: None
- **Output (JSON)**:
  - Sets `round1StartedAt` on the server if not already set.
  - Returns: `{ questions: Array<{ _id, questionText, options, category }>, startedAt, durationMinutes }` (Omits `correctOptionIndex`).
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Round 1 is not active or participant already finalized test.

### 3.3 Submit Round 1 Answers
- **Method**: `POST`
- **Path**: `/api/round1/submit`
- **Access**: Authenticated Participant
- **Input (JSON)**:
  - `answers`: `{ [questionId: string]: number }`
- **Output (JSON)**:
  - `{ success: true, score: 24, totalQuestions: 30 }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Submission rejected because elapsed time exceeded duration + 60s grace period.
  - `409 Conflict`: Round 1 was already finalized previously.

### 3.4 Log Proctoring Event
- **Method**: `POST`
- **Path**: `/api/proctor/log-event`
- **Access**: Authenticated Participant
- **Input (JSON)**:
  - `round`: `'round1' | 'round2'`
  - `eventType`: `'TAB_SWITCH' | 'BLUR' | 'FULLSCREEN_EXIT'`
- **Output (JSON)**:
  - `{ success: true, totalViolations: number, actionRequired: 'NONE' | 'DISQUALIFY' }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `400 Bad Request`: Invalid event type.

### 3.5 Fetch Assigned Round 2 Problems
- **Method**: `GET`
- **Path**: `/api/round2/questions`
- **Access**: Authenticated Shortlisted Participant
- **Input**: None
- **Output (JSON)**:
  - Sets `round2StartedAt` if not already set.
  - Returns: `{ questions: Array<{ _id, title, description, difficulty, sampleCases, starterTemplates }> }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Participant is not shortlisted (`isShortlisted === false`) or Round 2 is inactive.

### 3.6 Execute Code Against Test Cases
- **Method**: `POST`
- **Path**: `/api/round2/execute`
- **Access**: Authenticated Shortlisted Participant
- **Input (JSON)**:
  - `questionId`: string (required)
  - `language`: `'python' | 'cpp' | 'java'` (required)
  - `code`: string (required)
- **Output (JSON)**:
  - `{ testCasesPassed: 4, totalTestCases: 5, sampleResults: Array<{ passed, input, output, expected }> }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `400 Bad Request`: Unsupported programming language or question not assigned to candidate.
  - `502 Bad Gateway`: Piston compilation or execution timed out.

### 3.7 Finalize Round 2
- **Method**: `POST`
- **Path**: `/api/round2/submit`
- **Access**: Authenticated Shortlisted Participant
- **Input**: None
- **Output (JSON)**:
  - `{ success: true, message: "Round 2 completed." }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Exam deadline elapsed or already finalized.

---

## 4. Volunteer Check-in Endpoints

### 4.1 Lookup Attendee by ID or Email
- **Method**: `GET`
- **Path**: `/api/attendance?search={query}`
- **Access**: Desk Volunteer (`x-attendance-secret`)
- **Output (JSON)**:
  - `{ participant: { participantId, name, email, regNo, teamType, partner, attendanceRound1, attendanceRound2 } }`
- **Error Cases**:
  - `401 Unauthorized`: Missing or invalid `x-attendance-secret`.
  - `404 Not Found`: No attendee matched the query.

### 4.2 Record Stage Attendance
- **Method**: `POST`
- **Path**: `/api/attendance`
- **Access**: Desk Volunteer (`x-attendance-secret`)
- **Input (JSON)**:
  - `participantId`: string (e.g. `'ESP-1001'`)
  - `round`: `'round1' | 'round2'`
- **Output (JSON)**:
  - `{ success: true, message: "Attendance confirmed.", participantId: "ESP-1001" }`
- **Error Cases**:
  - `401 Unauthorized`: Invalid header credentials.
  - `404 Not Found`: Participant ID does not exist.
  - `409 Conflict`: Attendance already marked for this round.

---

## 5. Administrator Control Endpoints

### 5.1 Event Configuration
- **Method**: `GET`, `POST`
- **Path**: `/api/admin/config`
- **Access**: Administrator (`x-admin-password`)
- **Input (POST JSON)**: Partial `EventConfig` fields to update
- **Output (JSON)**: `{ config: EventConfig }`
- **Error Cases**: `401 Unauthorized`

### 5.2 Deterministic Shortlist Cutoff
- **Method**: `POST`
- **Path**: `/api/admin/shortlist`
- **Access**: Administrator
- **Input (JSON)**:
  - `cutoffCount`: number (e.g. `30`)
- **Output (JSON)**:
  - `{ success: true, shortlistedCount: 30, lowestQualifyingScore: 22 }`
- **Error Cases**:
  - `400 Bad Request`: Invalid cutoff count.
  - `401 Unauthorized`: Invalid administrator password.

### 5.3 Per-Participant AI Evaluation
- **Method**: `POST`
- **Path**: `/api/admin/round2-evaluate`
- **Access**: Administrator
- **Input (JSON)**:
  - `participantId`: string (required)
- **Output (JSON)**:
  - `{ success: true, participantId: string, updatedScore: number }`
- **Error Cases**:
  - `401 Unauthorized`: Invalid admin credentials.
  - `404 Not Found`: Participant not found.
  - `502 Bad Gateway`: OpenRouter API failure.

### 5.4 Participant Round Reset
- **Method**: `POST`
- **Path**: `/api/admin/redo-round`
- **Access**: Administrator
- **Input (JSON)**:
  - `participantId`: string
  - `round`: `'round1' | 'round2'`
- **Output (JSON)**:
  - `{ success: true, message: "Round reset successfully." }`
- **Error Cases**:
  - `401 Unauthorized`: Invalid credentials.
  - `404 Not Found`: Participant not found.

### 5.5 Award Winner Placement
- **Method**: `POST`
- **Path**: `/api/admin/teams/[id]/winner`
- **Access**: Administrator
- **Input (JSON)**:
  - `isWinner`: boolean
  - `winnerRank`: number (e.g. 1, 2, 3)
- **Output (JSON)**:
  - `{ success: true, participant: Participant }`
- **Error Cases**:
  - `401 Unauthorized`: Invalid credentials.
  - `404 Not Found`: Participant ID not found.
