# API Specification — Accountill Rebuild

---

## 1. Authentication & Security Schemes

- **Public Routes**: Accessible without credentials. Rate-limited by IP.
- **Protected Routes**: Require a valid JWT passed in the HTTP `Authorization: Bearer <token>` header. The token payload must contain `{ id: string, email: string }`. All database operations are strictly scoped to `req.userId`.

---

## 2. Authentication & User Account Endpoints

### 2.1 User Sign In
- **Method**: `POST`
- **Path**: `/users/signin`
- **Access**: Public
- **Input (JSON)**:
  - `email`: string (required)
  - `password`: string (required)
- **Output (JSON)**:
  - `{ result: { _id, name, email }, token: string }`
- **Error Cases**:
  - `400 Bad Request`: Email or password omitted.
  - `401 Unauthorized`: Invalid credentials.
  - `404 Not Found`: Account does not exist.

### 2.2 User Sign Up
- **Method**: `POST`
- **Path**: `/users/signup`
- **Access**: Public
- **Input (JSON)**:
  - `firstName`: string (required)
  - `lastName`: string (required)
  - `email`: string (required)
  - `password`: string (required)
  - `confirmPassword`: string (required)
  - `bio`: string (optional)
- **Output (JSON)**:
  - `{ result: { _id, name, email }, token: string }`
- **Error Cases**:
  - `400 Bad Request`: Passwords do not match or fields missing.
  - `409 Conflict`: An account with this email already exists.

### 2.3 Request Password Reset Token
- **Method**: `POST`
- **Path**: `/users/forgot`
- **Access**: Public
- **Input (JSON)**:
  - `email`: string (required)
- **Output (JSON)**:
  - `{ message: "If that email exists, a password reset link has been dispatched." }`
- **Error Cases**:
  - `400 Bad Request`: Email parameter missing.
  - `500 Server Error`: SMTP transport failure.

### 2.4 Complete Password Reset
- **Method**: `POST`
- **Path**: `/users/reset`
- **Access**: Public
- **Input (JSON)**:
  - `token`: string (required)
  - `password`: string (required)
- **Output (JSON)**:
  - `{ message: "Password updated successfully." }`
- **Error Cases**:
  - `400 Bad Request`: Token or new password missing.
  - `422 Unprocessable Entity`: Token is invalid or expired.

---

## 3. Business Profile Endpoints

### 3.1 Fetch Business Profile by User ID
- **Method**: `GET`
- **Path**: `/profiles?searchQuery={userId}`
- **Access**: Authenticated User (must match `req.userId`)
- **Input**: Query parameter `searchQuery`
- **Output (JSON)**:
  - `{ data: Profile }`
- **Error Cases**:
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Querying another user's business profile.
  - `404 Not Found`: Profile not yet configured.

### 3.2 Create Business Profile
- **Method**: `POST`
- **Path**: `/profiles`
- **Access**: Authenticated User
- **Input (JSON)**:
  - `name`, `email`, `phoneNumber`, `businessName`, `contactAddress`, `paymentDetails`, `logo`, `website`
- **Output (JSON)**:
  - Newly created `Profile` document
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `409 Conflict`: Profile already exists for this user.

### 3.3 Update Business Profile
- **Method**: `PATCH`
- **Path**: `/profiles/:id`
- **Access**: Profile Owner (`req.userId === profile.userId`)
- **Input (JSON)**: Partial profile fields to update
- **Output (JSON)**: Updated `Profile` document
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Attempting to edit a profile owned by another user.
  - `404 Not Found`: Profile ID does not exist.

---

## 4. Customer / Client Directory Endpoints

### 4.1 Fetch Scoped Clients List
- **Method**: `GET`
- **Path**: `/clients/user?searchQuery={userId}`
- **Access**: Authenticated User (`req.userId === searchQuery`)
- **Output (JSON)**:
  - `{ data: Array<Client> }`
- **Error Cases**:
  - `401 Unauthorized`: Missing Bearer token.
  - `403 Forbidden`: Attempting to read another user's client list.

### 4.2 Create Client Contact
- **Method**: `POST`
- **Path**: `/clients`
- **Access**: Authenticated User
- **Input (JSON)**:
  - `name`: string (required)
  - `email`: string (required)
  - `phone`: string (optional)
  - `address`: string (optional)
- **Output (JSON)**:
  - Newly created `Client` document with `userId` set from authenticated session
- **Error Cases**:
  - `400 Bad Request`: Name or email omitted.
  - `401 Unauthorized`: Unauthenticated.

### 4.3 Update Client Contact
- **Method**: `PATCH`
- **Path**: `/clients/:id`
- **Access**: Client Owner (`req.userId === client.userId`)
- **Input (JSON)**: Partial client fields
- **Output (JSON)**: Updated `Client` document
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Caller does not own this client record.
  - `404 Not Found`: Client ID not found.

### 4.4 Delete Client Contact
- **Method**: `DELETE`
- **Path**: `/clients/:id`
- **Access**: Client Owner (`req.userId === client.userId`)
- **Output (JSON)**:
  - `{ message: "Client deleted successfully." }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Unauthorized attempt to delete another user's client.
  - `404 Not Found`: Client ID not found.

---

## 5. Invoice & Billing Endpoints

### 5.1 Fetch Invoices by User
- **Method**: `GET`
- **Path**: `/invoices?searchQuery={userId}`
- **Access**: Authenticated User (`req.userId === searchQuery`)
- **Output (JSON)**:
  - `{ data: Array<Invoice> }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Unauthorized cross-tenant query.

### 5.2 Fetch Single Invoice by ID
- **Method**: `GET`
- **Path**: `/invoices/:id`
- **Access**: Invoice Owner or Recipient
- **Output (JSON)**:
  - Single `Invoice` document
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Not authorized to view this invoice.
  - `404 Not Found`: Invoice ID not found.

### 5.3 Fetch User Invoice Count (Serial Numbering)
- **Method**: `GET`
- **Path**: `/invoices/count?searchQuery={userId}`
- **Access**: Authenticated User
- **Output (JSON)**:
  - Integer count of invoices created by user
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.

### 5.4 Create Invoice
- **Method**: `POST`
- **Path**: `/invoices`
- **Access**: Authenticated User
- **Input (JSON)**:
  - `dueDate`, `currency`, `items`, `rates`, `client`, `notes`, `type`
- **Output (JSON)**:
  - Newly created `Invoice` document with server-verified totals
- **Error Cases**:
  - `400 Bad Request`: Invalid line item structure or missing required fields.
  - `401 Unauthorized`: Unauthenticated.

### 5.5 Update Invoice or Record Payment
- **Method**: `PATCH`
- **Path**: `/invoices/:id`
- **Access**: Invoice Owner (`req.userId === invoice.creator`)
- **Input (JSON)**: Updated line items or appended `paymentRecords`
- **Output (JSON)**: Updated `Invoice` document with recalculated balance and status
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Caller does not own this invoice.
  - `404 Not Found`: Invoice ID not found.

### 5.6 Delete Invoice
- **Method**: `DELETE`
- **Path**: `/invoices/:id`
- **Access**: Invoice Owner (`req.userId === invoice.creator`)
- **Output (JSON)**:
  - `{ message: "Invoice deleted successfully." }`
- **Error Cases**:
  - `401 Unauthorized`: Unauthenticated.
  - `403 Forbidden`: Unauthorized attempt to delete another user's invoice.
  - `404 Not Found`: Invoice ID not found.

---

## 6. PDF Rendering & Email Dispatch Endpoints

### 6.1 Stream PDF Invoice (In-Memory)
- **Method**: `POST`
- **Path**: `/create-pdf`
- **Access**: Authenticated User
- **Input (JSON)**: Complete invoice data payload
- **Output (Binary Stream)**:
  - `Content-Type: application/pdf` streamed directly to client; zero disk persistence
- **Error Cases**:
  - `400 Bad Request`: Incomplete invoice payload.
  - `500 Server Error`: PDF compilation engine error.

### 6.2 Send PDF Invoice via Email
- **Method**: `POST`
- **Path**: `/send-pdf`
- **Access**: Authenticated User
- **Input (JSON)**:
  - `email`: string (recipient email)
  - `company`: object (business profile details)
  - Complete invoice data payload
- **Output (JSON)**:
  - `{ success: true, message: "Invoice dispatched to recipient." }`
- **Error Cases**:
  - `400 Bad Request`: Recipient email missing.
  - `500 Server Error`: SMTP transport failure or PDF rendering failure.

---

## 7. Forensic Audit: Original Codebase Entry Points & Route Inventory

### 7.1 Table 1: Original Backend API Routes

| Method | Path | What It Does | Input | Output | Who May Call It | Auth Check (Path:Line) |
|---|---|---|---|---|---|---|
| `POST` | `/create-pdf` | Compiles HTML invoice to PDF file on disk | Invoice JSON payload | Binary status promise | Anyone | **none found** (`server/index.js:87-95`) |
| `GET` | `/fetch-pdf` | Streams static `invoice.pdf` from server disk | None | Binary PDF stream | Anyone | **none found** (`server/index.js:97-100`) |
| `POST` | `/send-pdf` | Compiles PDF on disk and sends via Nodemailer | Recipient email, company, invoice JSON | Status message promise | Anyone | **none found** (`server/index.js:53-85`) |
| `GET` | `/` | Root server status check | None | HTML welcome string | Anyone | **none found** (`server/index.js:102-104`) |
| `GET` | `/invoices/count` | Returns total invoice count for sequential numbering | Query param `searchQuery` (userId) | Integer count | Anyone | **none found** (`server/routes/invoices.js:6`) |
| `GET` | `/invoices/:id` | Fetches single invoice document | URL param `id` | Invoice JSON | Anyone | **none found** (`server/routes/invoices.js:7`) |
| `GET` | `/invoices` | Fetches invoices filtered by creator | Query param `searchQuery` (userId) | Array of invoices | Anyone | **none found** (`server/routes/invoices.js:8`) |
| `POST` | `/invoices` | Persists a new invoice | Invoice JSON payload | Newly created invoice | Anyone | **none found** (`server/routes/invoices.js:9`) |
| `PATCH` | `/invoices/:id` | Updates invoice line items or adds payment | URL param `id`, update JSON | Updated invoice JSON | Anyone | **none found** (`server/routes/invoices.js:10`) |
| `DELETE` | `/invoices/:id` | Permanently deletes invoice | URL param `id` | Deletion success message | Anyone | **none found** (`server/routes/invoices.js:11`) |
| `GET` | `/clients/user` | Fetches clients belonging to a user | Query param `searchQuery` (userId) | Array of clients | Anyone | **none found** (`server/routes/clients.js:10`) |
| `GET` | `/clients` | Fetches paginated clients across all tenants | Query param `page` | Object with data array & pages | Anyone | **none found** (`server/routes/clients.js:6`) |
| `POST` | `/clients` | Creates a new client contact | Client JSON payload | Created client JSON | Anyone | **none found** (`server/routes/clients.js:7`) |
| `PATCH` | `/clients/:id` | Updates client contact information | URL param `id`, update JSON | Updated client JSON | Anyone | **none found** (`server/routes/clients.js:8`) |
| `DELETE` | `/clients/:id` | Deletes a client contact | URL param `id` | Deletion success message | Anyone | **none found** (`server/routes/clients.js:9`) |
| `POST` | `/users/signin` | Validates credentials and returns JWT token | `email`, `password` | `{ result, token }` | Anyone | **none found** (`server/routes/userRoutes.js:6`) |
| `POST` | `/users/signup` | Registers new user and returns JWT token | Registration JSON | `{ result, token }` | Anyone | **none found** (`server/routes/userRoutes.js:7`) |
| `POST` | `/users/forgot` | Creates reset token and emails recovery link | `email` | Status message JSON | Anyone | **none found** (`server/routes/userRoutes.js:8`) |
| `POST` | `/users/reset` | Resets password using valid token | `token`, `password` | Status message JSON | Anyone | **none found** (`server/routes/userRoutes.js:9`) |
| `GET` | `/profiles/:id` | Fetches single business profile | URL param `id` | Profile JSON | Anyone | **none found** (`server/routes/profile.js:6`) |
| `GET` | `/profiles` | Fetches profile matching user ID | Query param `searchQuery` (userId) | Profile JSON | Anyone | **none found** (`server/routes/profile.js:8`) |
| `POST` | `/profiles` | Creates business profile | Profile JSON payload | Created profile JSON | Anyone | **none found** (`server/routes/profile.js:9`) |
| `PATCH` | `/profiles/:id` | Updates business profile | URL param `id`, update JSON | Updated profile JSON | Anyone | **none found** (`server/routes/profile.js:10`) |
| `DELETE` | `/profiles/:id` | Deletes business profile | URL param `id` | Deletion success message | Anyone | **none found** (`server/routes/profile.js:11`) |

### 7.2 Table 2: Frontend Screens & Pages

| URL / Screen Name | What the User Does There | API Calls It Makes | File |
|---|---|---|---|
| `/` | Creates a new invoice, selects client, inputs items | `GET /invoices/count`, `GET /clients/user`, `GET /profiles`, `POST /invoices` | `client/src/components/Invoice/Invoice.js` |
| `/edit/invoice/:id` | Edits an existing invoice draft or rates | `GET /invoices/:id`, `PATCH /invoices/:id` | `client/src/components/Invoice/Invoice.js` |
| `/invoice/:id` | Views invoice details, adds payment installment, prints or emails PDF | `GET /invoices/:id`, `PATCH /invoices/:id`, `POST /create-pdf`, `GET /fetch-pdf`, `POST /send-pdf` | `client/src/components/InvoiceDetails/InvoiceDetails.js` |
| `/invoices` | Lists all invoices for logged-in user with pagination and status chips | `GET /invoices?searchQuery={userId}` | `client/src/components/Invoices/Invoices.js` |
| `/login` | Signs in, signs up, or authenticates via Google OAuth | `POST /users/signin`, `POST /users/signup`, `POST /profiles` | `client/src/components/Login/Login.js` |
| `/settings` | Configures company business name, logo, address, and bank instructions | `GET /profiles?searchQuery={userId}`, `POST /profiles`, `PATCH /profiles/:id` | `client/src/components/Settings/Settings.js` |
| `/dashboard` | Reviews financial analytics: income received, pending dues, recent payments | `GET /invoices?searchQuery={userId}` | `client/src/components/Dashboard/Dashboard.js` |
| `/customers` | Views customer address book, opens modal to add or edit clients | `GET /clients/user?searchQuery={userId}`, `POST /clients`, `PATCH /clients/:id`, `DELETE /clients/:id` | `client/src/components/Clients/Clients.js` |
| `/forgot` | Enters registered email address to receive password reset link | `POST /users/forgot` | `client/src/components/Password/Forgot.js` |
| `/reset/:token` | Enters new password with reset token from email | `POST /users/reset` | `client/src/components/Password/Reset.js` |
| `*` (Fallback) | Redirects undefined routes to `/invoices` or `/` | None | `client/src/App.js` |

### 7.3 Route Counting Methodology

- **Total Route Files**: **5 files**
  1. `server/index.js` (4 root handlers)
  2. `server/routes/invoices.js` (6 handlers)
  3. `server/routes/clients.js` (5 handlers)
  4. `server/routes/userRoutes.js` (4 handlers)
  5. `server/routes/profile.js` (5 handlers)
- **Total Mounted Route Handlers**: **24 handlers** (4 + 6 + 5 + 4 + 5 = 24).
- **Unmounted Handlers in Controllers**: **3 functions**
  1. `getInvoices` in `server/controllers/invoices.js:36` (not imported or mounted in router).
  2. `getProfiles` in `server/controllers/profile.js:8` (imported in router, but route commented out at `server/routes/profile.js:7`).
  3. `getProfilesBySearch` in `server/controllers/profile.js:85` (exported but never mounted in router).

