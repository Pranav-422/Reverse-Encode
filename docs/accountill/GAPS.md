# GAPS.md — Original Flaws & Strategic Rebuild Improvements

---

## 1. Forensic Audit: What the Original Codebase Gets Wrong or Misses

### 1.1 Unprotected Routes & Missing Authorization Middleware (BOLA)
- **The Defect**: While an authentication middleware file exists (`server/middleware/auth.js`), it is **never imported or mounted** on any route in `server/routes/invoices.js`, `server/routes/clients.js`, or `server/routes/profile.js`. Any unauthenticated caller can query, modify, or permanently delete any invoice or customer record in the database by passing an `_id`.
- **Evidence**:
  - `accountill/server/routes/invoices.js:6-11`: Router declares routes without `auth` middleware.
  - `accountill/server/routes/clients.js:6-10`: Client routes mounted with no middleware.
  - `accountill/server/routes/profile.js:6-11`: Profile routes mounted with no middleware.
  - `accountill/server/controllers/invoices.js:81-98`: `updateInvoice` and `deleteInvoice` modify or delete documents without verifying ownership.

### 1.2 Global Customer Directory Leakage
- **The Defect**: The `getClients` controller queries `ClientModel.find()` without a user filter, returning paginated customer contacts (names, emails, physical addresses, phone numbers) across all users on the platform.
- **Evidence**:
  - `accountill/server/controllers/clients.js:44-45`: Unfiltered query `ClientModel.find().sort({ _id: -1 })`.
  - `accountill/client/src/api/index.js:22`: Client API endpoint calls `/clients?page=${page}` without scoping.

### 1.3 Shared Hardcoded Disk File Collision in PDF Generation
- **The Defect**: Both `/create-pdf` and `/send-pdf` write generated invoice documents to a single hardcoded filename on disk (`invoice.pdf`) in the server root. When two users download or email an invoice simultaneously, their files overwrite each other, causing User A to download or email User B's invoice.
- **Evidence**:
  - `accountill/server/index.js:57, 69`: Hardcoded `invoice.pdf` written and attached to email.
  - `accountill/server/index.js:88`: Hardcoded `toFile('invoice.pdf')` during PDF creation.
  - `accountill/server/index.js:98`: `res.sendFile(`${__dirname}/invoice.pdf`)` sends the static shared file.

### 1.4 Unverified Google OAuth Token Decoding
- **The Defect**: The auth middleware inspects token length (`token.length < 500`). For tokens $\ge 500$ characters, it assumes the token is from Google and calls `jwt.decode(token)` instead of verifying the cryptographic signature with Google's public keys. Anyone can supply an unverified token payload with an arbitrary `sub` claim.
- **Evidence**:
  - `accountill/server/middleware/auth.js:10`: Arbitrary length check `isCustomAuth = token.length < 500`.
  - `accountill/server/middleware/auth.js:22-25`: Unverified decoding `decodeData = jwt.decode(token)`.

### 1.5 Blind Server Trust of Client Calculations & Missing Validation
- **The Defect**: The backend performs zero calculation verification or schema validation on invoice creation. Fields like `total`, `subTotal`, `vat`, `rates`, and `items` are written directly to MongoDB from `req.body` without verifying whether unit prices $\times$ quantities minus discounts actually sum to `total`.
- **Evidence**:
  - `accountill/server/controllers/invoices.js:55-60`: `new InvoiceModel(invoice)` saved directly without validation.
  - `accountill/server/models/InvoiceModel.js:6-10`: Schema accepts unvalidated numeric and string inputs.

### 1.6 Invoice Serial Number Collision Race Condition
- **The Defect**: Invoice numbering relies on the client querying `GET /invoices/count?searchQuery=...` on form mount, parsing the integer count, and padding it to 3 digits. If two invoices are created concurrently, or an invoice is deleted, duplicate invoice numbers are produced.
- **Evidence**:
  - `accountill/client/src/components/Invoice/Invoice.js:93`: Querying count from client component.
  - `accountill/client/src/components/Invoice/Invoice.js:241-245`: Client-side padding and number generation.
  - `accountill/server/controllers/invoices.js:22-33`: Non-atomic count endpoint.

### 1.7 Unhandled Promise Rejections & Double Response Sending
- **The Defect**: In `/create-pdf` and `/send-pdf`, `res.send(Promise.reject())` is called inside the callback upon error. Express `res.send()` does not accept rejected promises, causing unhandled rejections. Additionally, there is no `return` after error sending, causing Express to attempt to execute `res.send(Promise.resolve())` right after, triggering `ERR_HTTP_HEADERS_SENT`.
- **Evidence**:
  - `accountill/server/index.js:73-76`: Callback invokes `res.send(Promise.reject())` followed immediately by `res.send(Promise.resolve())`.
  - `accountill/server/index.js:89-92`: Same defect repeated in `/create-pdf`.

### 1.8 Infinite Loading Spinners on Client Network Failures
- **The Defect**: Across multiple Redux action creators (`getInvoicesByUser`, `createInvoice`), the action dispatches `START_LOADING`, but the `catch` block only logs errors to `console.log` without dispatching `END_LOADING`. Any network drop or 4xx/5xx response causes the client UI to remain trapped in an unrecoverable loading skeleton.
- **Evidence**:
  - `accountill/client/src/actions/invoiceActions.js:16, 20-23`: `START_LOADING` dispatched without corresponding `END_LOADING` in `catch`.
  - `accountill/client/src/actions/invoiceActions.js:44, 48-51`: Same defect in `createInvoice`.

### 1.9 Hardcoded Production Domain in Password Reset Links
- **The Defect**: The forgot-password controller creates recovery links with a hardcoded URL string `https://accountill.com/reset/${token}`. When running locally or on custom domains, password reset emails send recipients to the remote production domain rather than the active application origin.
- **Evidence**:
  - `accountill/server/controllers/user.js:122-124`: Hardcoded URL in reset email HTML.

### 1.10 Branding and Documentation Drift
- **The Defect**: `README.md:8` announces the repository rename from "Arc Invoice" to "Accountill", yet outbound password emails still read *"from Arc Invoicing application"*, Docker compose points to `mongodb://mongo:27017/arch`, `client/src/clients.json` contains unused mock data, and `server/documents/invoice.js` is unreferenced dead code.
- **Evidence**:
  - `accountill/server/controllers/user.js:121`: Outdated project name in email body.
  - `accountill/README.md:141`: Legacy database name in Docker compose instructions.
  - `accountill/client/src/components/Clients/Clients.js:28`: Commented-out mock import.
  - `accountill/server/index.js:22`: Commented-out invoice template import.

---

## 2. The 2 Strategic Improvements We Will Build

---

### Improvement 1: Strict Multi-Tenant Isolation & Cryptographic Auth Pipeline

#### What We Will Build
1. **Universal Route-Level JWT Authorization**: Mount a hardened authentication middleware on every route in `invoices`, `clients`, and `profiles`. The middleware validates bearer tokens against `process.env.SECRET` and binds the verified user identifier directly to `req.userId`.
2. **Cryptographic Google Identity Verification**: Integrate the official `google-auth-library` to cryptographically verify Google OAuth ID tokens against Google's public certificates, extracting the validated `sub` claim and replacing the vulnerable length check (`token.length < 500`).
3. **Strict Database Ownership Scoping**: Enforce that every query, patch, and delete operation on invoices, customer clients, and profiles filters by `req.userId`. Attempted cross-tenant access to another user's invoice or client will unconditionally fail with `403 Forbidden`.

#### Why It Matters to the User
- **Protects Confidential Financial Records**: Guarantees that small business owners and freelancers can store proprietary client lists, invoice rates, and payment records without risk of unauthorized access or deletion by other users.
- **Prevents Identity Forgery**: Ensures user accounts cannot be hijacked through forged or tampered OAuth tokens.
- **Builds Client Trust**: Businesses can invoice clients knowing their data and customer contacts remain strictly private and isolated.

---

### Improvement 2: Concurrency-Safe In-Memory Document Streaming & Server-Verified Calculations

#### What We Will Build
1. **Direct In-Memory PDF Streaming**: Redesign the PDF generation engine to stream binary buffers directly to the HTTP response using `pdf.create(html).toStream((err, stream) => stream.pipe(res))` or in-memory buffers. Completely eliminate the shared static `server/invoice.pdf` file on disk.
2. **Atomic Monotonic Invoice Number Allocation**: Implement server-side sequential invoice numbering per business using an atomic counter or transactional sequence generator, preventing duplicate invoice numbers when multiple invoices are created concurrently.
3. **Authoritative Server-Side Financial Recalculation**: Implement rigorous server-side recalculation of subtotal, tax/VAT percentages, line item discounts, and total payable amounts from individual items before persisting invoice documents to MongoDB.
4. **Resilient Redux State & Error Boundaries**: Ensure every client action creator dispatches `END_LOADING` inside `finally` or `catch` blocks and surfaces user-friendly toast notifications, preventing hanging loading screens.

#### Why It Matters to the User
- **Eliminates High-Concurrency Document Collisions**: Multiple business owners or clients downloading invoices simultaneously will always receive their own distinct documents, eliminating data leakage and confusion.
- **Guarantees Accounting & Tax Compliance**: Precludes calculation errors, negative totals, or tampered invoices, ensuring invoices are auditable and mathematically accurate.
- **Provides a Flawless User Experience**: Eliminates frozen loading screens on transient network failures, giving business owners instant, actionable feedback.
