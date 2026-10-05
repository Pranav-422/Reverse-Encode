# OBSERVATIONS.md — Verified Claims About Accountill

> Forensic observations extracted from the reverse-engineering analysis of `Panshak/accountill`. Every claim is verified directly against repository source files.

---

## 1. Tech Stack & Infrastructure

- Accountill is built on the MERN stack (MongoDB, Express, React, Node.js).
  Evidence: accountill/README.md:2-3 [Confirmed]
- The backend server uses Node.js ES Modules via `"type": "module"` in package configuration.
  Evidence: accountill/server/package.json:7 [Confirmed]
- The backend web framework is Express version `^4.17.1`.
  Evidence: accountill/server/package.json:18 [Confirmed]
- The server connects to MongoDB via Mongoose version `^5.12.10`.
  Evidence: accountill/server/package.json:22, accountill/server/index.js:109 [Confirmed]
- Password encryption on the server uses `bcryptjs` version `^2.4.3`.
  Evidence: accountill/server/package.json:15, accountill/server/controllers/user.js:52, 145 [Confirmed]
- JSON Web Token signing and decoding uses `jsonwebtoken` version `^8.5.1`.
  Evidence: accountill/server/package.json:20, accountill/server/controllers/user.js:35, 60 [Confirmed]
- PDF invoice generation uses `html-pdf` version `^3.0.1`.
  Evidence: accountill/server/package.json:19, accountill/server/index.js:9, 57, 88 [Confirmed]
- Outbound emails are sent through an SMTP transport using Nodemailer version `^6.6.3`.
  Evidence: accountill/server/package.json:23, accountill/server/index.js:8, 38 [Confirmed]
- Cross-Origin Resource Sharing is enabled on the server using `cors` version `^2.8.5`.
  Evidence: accountill/server/package.json:16, accountill/server/index.js:5, 30 [Confirmed]
- The frontend is built with React version `^17.0.2` and `react-dom` version `^17.0.2`.
  Evidence: accountill/client/package.json:25, 27 [Confirmed]
- Frontend build tooling is managed using `react-scripts` version `4.0.3` (Create React App).
  Evidence: accountill/client/package.json:34 [Confirmed]
- The frontend user interface utilizes Material-UI v4 (`@material-ui/core` `^4.11.4`, `@material-ui/icons` `^4.11.2`, `@material-ui/lab` `^4.0.0-alpha.58`, and `@material-ui/pickers` `^3.3.10`).
  Evidence: accountill/client/package.json:10-13 [Confirmed]
- Client-side application state is managed using `redux` `^4.1.0`, `react-redux` `^7.2.4`, and `redux-thunk` `^2.3.0`.
  Evidence: accountill/client/package.json:32, 38, 39 [Confirmed]
- Client-side routing is handled using `react-router-dom` version `^5.2.0`.
  Evidence: accountill/client/package.json:33, accountill/client/src/App.js:5 [Confirmed]
- HTTP requests from the frontend client are dispatched using `axios` version `^0.21.1`.
  Evidence: accountill/client/package.json:19, accountill/client/src/api/index.js:1, 4 [Confirmed]
- The client decodes JWT authentication tokens using `jwt-decode` version `^3.1.2`.
  Evidence: accountill/client/package.json:22, accountill/client/src/components/NavBar/NavBar.js:13 [Confirmed]
- Google OAuth is integrated on the client via `@react-oauth/google` version `^0.9.0`.
  Evidence: accountill/client/package.json:14, accountill/client/src/components/Login/Login.js:11 [Confirmed]
- Data visualization and analytics charts utilize `apexcharts` version `^3.28.1` and `react-apexcharts` version `^1.3.9`.
  Evidence: accountill/client/package.json:18, 26 [Confirmed]

---

## 2. Authentication & Authorization

- An authentication middleware function exists in `server/middleware/auth.js` that checks for a Bearer authorization token.
  Evidence: accountill/server/middleware/auth.js:7-10 [Confirmed]
- Authentication middleware `server/middleware/auth.js` is never imported, mounted, or applied to any route in `server/routes/` or `server/index.js`.
  Evidence: accountill/server/routes/invoices.js:1-14, accountill/server/routes/clients.js:1-12, accountill/server/routes/profile.js:1-14, accountill/server/routes/userRoutes.js:1-11, accountill/server/index.js:1-36 [Confirmed]
- All invoice CRUD operations (`GET`, `POST`, `PATCH`, `DELETE`) operate with zero server-side authentication checks.
  Evidence: accountill/server/routes/invoices.js:6-11, accountill/server/controllers/invoices.js:9, 53, 68, 81, 93 [Confirmed]
- The client list route `/clients` returns customer records across all users with no ownership filter.
  Evidence: accountill/server/controllers/clients.js:44-45 [Confirmed]
- The auth middleware inspects token length (`token.length < 500`) and calls `jwt.decode(token)` for tokens $\ge 500$ without verifying signatures against Google public keys.
  Evidence: accountill/server/middleware/auth.js:10, 22-25 [Confirmed]
- The forgot-password endpoint returns an explicit error message `"User does not exist in our database"` if an email is not registered.
  Evidence: accountill/server/controllers/user.js:110-112 [Confirmed]
- Password reset links emailed to users hardcode the remote production domain `https://accountill.com/reset/${token}`.
  Evidence: accountill/server/controllers/user.js:122-124 [Confirmed]

---

## 3. Data Models & Relationships

- The application defines 4 primary Mongoose models: `User`, `Profile`, `ClientModel`, and `InvoiceModel`.
  Evidence: accountill/server/models/userModel.js:11, accountill/server/models/ProfileModel.js:15, accountill/server/models/ClientModel.js:16, accountill/server/models/InvoiceModel.js:25 [Confirmed]
- Cross-collection relationships between User and Invoices/Clients/Profiles are stored as arrays of plain ID strings (`[String]`) rather than native Mongoose `ObjectId` references with `ref`.
  Evidence: accountill/server/models/InvoiceModel.js:15, accountill/server/models/ClientModel.js:9, accountill/server/models/ProfileModel.js:12 [Confirmed]
- `InvoiceModel` does not reference `ClientModel` by ID; it embeds a denormalized snapshot object `{ name, email, phone, address }`.
  Evidence: accountill/server/models/InvoiceModel.js:17 [Confirmed]
- Line items are stored as an embedded array of subdocuments `items: [ { itemName, unitPrice, quantity, discount } ]` inside `InvoiceModel`.
  Evidence: accountill/server/models/InvoiceModel.js:6 [Confirmed]
- Payment records are stored as an embedded array of subdocuments `paymentRecords: [ { amountPaid, datePaid, paymentMethod, note, paidBy } ]` inside `InvoiceModel`.
  Evidence: accountill/server/models/InvoiceModel.js:18 [Confirmed]
- Line item numeric attributes (`unitPrice`, `quantity`, `discount`) and tax `rates` are defined as `String` in the schema rather than `Number`.
  Evidence: accountill/server/models/InvoiceModel.js:6-7 [Confirmed]
- Default document creation timestamps use `default: new Date()` instead of passing the function reference `default: Date.now`.
  Evidence: accountill/server/models/ClientModel.js:12, accountill/server/models/InvoiceModel.js:21 [Confirmed]

---

## 4. Endpoints & Concurrency

- The backend provides 5 route files exposing 24 route handlers in total.
  Evidence: accountill/server/index.js:53, 87, 97, 102, accountill/server/routes/invoices.js:6-11, accountill/server/routes/clients.js:6-10, accountill/server/routes/userRoutes.js:6-9, accountill/server/routes/profile.js:6, 8-11 [Confirmed]
- The frontend exposes 11 client route paths across 10 page components.
  Evidence: accountill/client/src/App.js:32-43 [Confirmed]
- Both `/create-pdf` and `/send-pdf` write generated PDFs to a single hardcoded filename on disk (`invoice.pdf`) in the server root.
  Evidence: accountill/server/index.js:57, 69, 88, 98 [Confirmed]
- PDF generation error callbacks invoke `res.send(Promise.reject())` without a return statement, causing unhandled promise rejections and subsequent double response sending.
  Evidence: accountill/server/index.js:73-76, 89-92 [Confirmed]
- Invoice serial numbers are derived by querying `/invoices/count?searchQuery=...` on the client and incrementing the count in client state.
  Evidence: accountill/client/src/components/Invoice/Invoice.js:93, 241-245, accountill/server/controllers/invoices.js:22-33 [Confirmed]
- Redux action creators dispatch `START_LOADING` but fail to dispatch `END_LOADING` inside `catch` blocks, causing UI spinners to hang on failure.
  Evidence: accountill/client/src/actions/invoiceActions.js:16, 20-23, accountill/client/src/actions/invoiceActions.js:44, 48-51 [Confirmed]

---

## 5. Dead Code & Odd Files

- A compiled PDF artifact `server/invoice.pdf` exists directly in the repository.
  Evidence: accountill/server/invoice.pdf [Confirmed]
- The compiled production build directory `client/build/` is committed to git because `build` is omitted from `client/.gitignore`.
  Evidence: accountill/client/build/index.html:1, accountill/client/.gitignore:1-3 [Confirmed]
- `client/src/clients.json` contains 50 rows of mock client data whose import is commented out in `Clients.js`.
  Evidence: accountill/client/src/components/Clients/Clients.js:28, accountill/client/src/clients.json:1-50 [Confirmed]
- `server/documents/invoice.js` is an abandoned HTML template whose import is commented out in `server/index.js`.
  Evidence: accountill/server/index.js:22, accountill/server/documents/invoice.js:1-90 [Confirmed]
- `server/models/ClientModel.js` contains an unreferenced `import express from 'express'`.
  Evidence: accountill/server/models/ClientModel.js:1 [Confirmed]
- Password reset email bodies still refer to the project by its old name `"Arc Invoicing application"`.
  Evidence: accountill/server/controllers/user.js:121 [Confirmed]
- The Docker Compose configuration points to a database named `arch`.
  Evidence: accountill/README.md:141 [Likely]
- There are no `.env.example` templates in either `client/` or `server/`.
  Evidence: accountill/server/.gitignore:2, accountill/client/.gitignore:2 [Confirmed]

---

## 6. Execution Commands & Environment Variables

- The server is started in production using `node index.js` and in development using `nodemon index.js`.
  Evidence: accountill/server/package.json:8-9 [Confirmed]
- The client is started using `react-scripts start` (Create React App).
  Evidence: accountill/client/package.json:44 [Confirmed]
- Multi-container production deployment uses Docker Compose file `docker-compose.prod.yml`.
  Evidence: accountill/README.md:131, accountill/docker-compose.prod.yml:1-35 [Confirmed]
- The backend server requires `DB_URL` for MongoDB connection string.
  Evidence: accountill/server/index.js:109 [Confirmed]
- The backend server defaults its listening port to `process.env.PORT` or fallback port `5000`.
  Evidence: accountill/server/index.js:110 [Confirmed]
- The backend server requires `SECRET` for signing and verifying JWT tokens.
  Evidence: accountill/server/controllers/user.js:35, 60, accountill/server/middleware/auth.js:13 [Confirmed]
- Outbound emailing requires 4 SMTP environment variables: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, and `SMTP_PASS`.
  Evidence: accountill/server/index.js:38-42, accountill/server/controllers/user.js:91-95 [Confirmed]
- The frontend client requires `REACT_APP_API` for setting the backend baseURL.
  Evidence: accountill/client/src/api/index.js:1 [Confirmed]
- Google OAuth on the frontend requires `REACT_APP_GOOGLE_CLIENT_ID`.
  Evidence: accountill/client/src/components/Login/Login.js:110 [Confirmed]

---

## 7. Unmounted & Dead Controller Functions

- Controller function `getInvoices` (`InvoiceModel.find({}).sort({_id:-1})`) is exported in `server/controllers/invoices.js:36` but never imported or mounted in `server/routes/invoices.js:1-14`.
  Evidence: accountill/server/controllers/invoices.js:36-48, accountill/server/routes/invoices.js:1-14 [Confirmed]
- Controller function `getProfiles` is imported in `server/routes/profile.js:2`, but its route definition `// router.get('/', getProfiles)` is commented out.
  Evidence: accountill/server/routes/profile.js:2, 7 [Confirmed]
- Controller function `getProfilesBySearch` is exported in `server/controllers/profile.js:85` but never imported or mounted in `server/routes/profile.js:1-14`.
  Evidence: accountill/server/controllers/profile.js:85-98, accountill/server/routes/profile.js:1-14 [Confirmed]

