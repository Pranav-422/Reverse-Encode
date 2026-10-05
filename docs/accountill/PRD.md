# Product Requirements Document (PRD) — Accountill Rebuild

---

## 1. Problem Definition

Freelancers, contractors, and small business owners require an intuitive, professional invoicing system to manage billing, issue PDF estimates and receipts, and track partial payment histories. Existing open-source invoicing applications frequently suffer from severe multi-tenant security vulnerabilities—such as missing authorization middlewares that allow any user to view, edit, or delete another business's invoices and client lists. Additionally, hardcoded file paths during PDF generation create critical concurrency race conditions where simultaneous users overwrite and download each other's financial documents.

---

## 2. Target Users

- **Target User**: Freelancers, sole proprietors, independent contractors, and small business operators needing simple, self-hosted or SaaS invoicing.
  - *Card Rebuild Brief Reference*: Unknown (Specific card brief was not provided; derived strictly from verified repository functionality).

---

## 3. One-Line Problem Statement

> **For** freelancers and small business owners **who** struggle with complicated enterprise billing software or insecure spreadsheet templates, **Accountill** provides a streamlined, secure MERN invoicing platform that isolates tenant data, generates professional PDF documents concurrently, tracks partial payment installments, and visualizes cash flow analytics, **unlike** manual invoice generators or unprotected open-source billing tools.

---

## 4. Core Flow (End-to-End)

1. **User Registration & Business Profile Setup**:
   The user signs up using email/password or Google OAuth. Upon login, they configure their business profile (company name, branding logo, contact phone, physical address, and bank payment instructions).
2. **Customer Contact Management**:
   The user adds client profiles (client name, billing email, phone number, address) to their private address book.
3. **Invoice Creation & Auto-Calculations**:
   The user initiates an invoice, selecting a client from their address book. They select a document type (Invoice, Receipt, Estimate, Quotation, Bill), currency, and due date. They add line items with unit prices, quantities, and discounts. The system dynamically computes subtotal, tax/VAT percentages, and total balances.
4. **Sequential Numbering & Finalization**:
   The server generates a unique sequential invoice number scoped to the user, records the creation timestamp, and persists the document with an initial status of `'UNPAID'`.
5. **PDF Generation & Email Dispatch**:
   The user previews the invoice, downloads an isolated PDF generated on the fly, or dispatches the invoice PDF directly to the client's email via Nodemailer.
6. **Payment Recording & Status Lifecycle**:
   As payments arrive, the business owner logs payment installments (amount paid, payment method, date, transaction note). The system updates `totalAmountReceived` and automatically transitions status from `'UNPAID'` to `'PARTIAL'` or `'PAID'`.
7. **Analytics & Financial Dashboard**:
   The user monitors payment health on the dashboard via charts displaying total revenue received, pending balances, recent payment timelines, and paid vs. unpaid document distributions.

---

## 5. MoSCoW Feature Categorization

### Must Have (P0)
- **Tenant Data Isolation & Authentication Middleware**: Universal JWT authorization middleware mounted on every invoice, client, and profile endpoint, strictly enforcing user ownership (`req.userId === record.userId`).
- **Concurrent-Safe In-Memory PDF Streaming**: PDF generation streamed directly via HTTP response buffers or isolated UUID temporary files, eliminating hardcoded file overwrite collisions.
- **Server-Side Financial Recalculation**: Server-side validation of line item totals, rates, and VAT calculations to prevent client tampering.
- **Cryptographic Google OAuth Signature Verification**: Validating Google OAuth identity tokens using Google's public keys via official libraries.
- **Atomic Invoice Number Allocation**: Server-enforced monotonic sequential numbering per user to prevent duplicate numbers.

### Should Have (P1)
- **Partial Payment Tracking**: Support for incremental payment records that recalculate balances and auto-advance invoice statuses.
- **Email Delivery via SMTP**: Automated dispatch of PDF invoices and receipts to client email addresses.
- **Customer Address Book**: Scoped CRUD management for billing clients.
- **Branded Business Profiles**: Custom logo, contact info, and bank payment terms embedded dynamically into PDF invoices.

### Could Have (P2)
- **Multi-Currency Formatting**: Support for dynamic currency symbols and localized date formats.
- **Cash Flow Dashboard Visualizations**: Chart-based breakdown of monthly income and pending collections.
- **Password Recovery Pipeline**: Email-based password reset using cryptographically hashed tokens with dynamic client origin links.

### Won't Have (This Version)
- Live payment gateway integrations (Stripe, PayPal, Razorpay) for direct client credit card processing.
- Automated recurring subscription billing or scheduled cron invoices.
- Multi-currency live forex exchange rate conversions.

---

## 6. Out of Scope

- Credit card processing, merchant accounts, and bank feeds.
- Multi-user team collaboration within a single business account (single-user tenancy per business).
- Double-entry accounting ledgers, payroll, and corporate tax return filings.

---

## 7. Acceptance Criteria (Given / When / Then)

### Feature 1: Multi-Tenant Route Authorization
- **Given** User A is authenticated and User B owns invoice `INV-123`,
- **When** User A attempts to view, edit, or delete `INV-123` via `GET /invoices/INV-123`, `PATCH /invoices/INV-123`, or `DELETE /invoices/INV-123`,
- **Then** the server must reject the request with HTTP status `403 Forbidden` and prevent data disclosure or modification.

### Feature 2: Concurrent PDF Generation Isolation
- **Given** two users generate invoices simultaneously,
- **When** the server handles both PDF rendering requests in parallel,
- **Then** each user must receive their exact invoice without file collision, data leakage, or unhandled promise rejection errors.

### Feature 3: Server-Side Calculation Verification
- **Given** an invoice creation payload with line item 2 units @ $100 and a client-tampered `total: 50`,
- **When** the client submits the invoice to `POST /invoices`,
- **Then** the server must recalculate the total based on unit prices, quantities, and taxes, saving the correct total ($200 + tax) or returning `400 Bad Request`.

### Feature 4: Scoped Client Listing
- **Given** User A has 5 clients and User B has 10 clients in the database,
- **When** User A queries `GET /clients`,
- **Then** the server must return only User A's 5 clients and zero clients belonging to User B.

### Killer Tests (Card-Specific)
- **Killer Test 1**: Unknown (Card Rebuild Brief was not supplied; replace with specific card criteria if provided).
  - *Fallback Criteria*: System must sustain 20 concurrent PDF invoice generation requests without file overwrites or process crashes.
- **Killer Test 2**: Unknown (Card Rebuild Brief was not supplied; replace with specific card criteria if provided).
  - *Fallback Criteria*: System must block unauthorized cross-tenant invoice modifications when an authenticated user sends an update request containing another user's invoice ID.
