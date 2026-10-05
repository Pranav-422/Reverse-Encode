# Reverse-Encode — Systems Reverse-Engineering & Architecture Audit Suite

> **Author**: Team Encode  
> **Repository**: [Reverse-Encode](https://github.com/Pranav-422/Reverse-Encode)

---

## Overview

Welcome to the **Reverse-Encode** documentation repository by **Team Encode**. This repository hosts comprehensive reverse-engineering audits, forensic security reviews, data models, workflow sequence diagrams, and strategic improvement roadmaps for analyzed systems.

---

## Featured Audit: Accountill (Invoicing & Billing Platform)

The primary forensic reverse-engineering audit in this repository focuses on **Accountill** (MERN-stack billing and invoicing system).

### Master Synthesis Document
👉 **[docs/accountill/PROMPTS_GAPS_AND_IMPROVEMENTS.md](docs/accountill/PROMPTS_GAPS_AND_IMPROVEMENTS.md)**  
*Complete journey from database schema extraction to documentation, highlighting all 10 verified architectural and security gaps with exact code citations, engineered solutions, and 9 strategic innovations to make the platform 10x better.*

---

## Accountill Documentation Suite Directory

All documentation files for Accountill are organized in [`docs/accountill/`](docs/accountill/):

| File | Description |
|---|---|
| **[PROMPTS_GAPS_AND_IMPROVEMENTS.md](docs/accountill/PROMPTS_GAPS_AND_IMPROVEMENTS.md)** | **Master Guide**: Prompt history, forensic gaps table, solutions, and 9 innovations. |
| **[system_architecture_and_workflow_audit.md](docs/accountill/system_architecture_and_workflow_audit.md)** | **Visual Blueprint**: 12 Mermaid diagrams, component inventories, and workflow sequence audits. |
| **[OBSERVATIONS.md](docs/accountill/OBSERVATIONS.md)** | **Verified Claims**: Tech stack, environment configuration, unmounted controller functions, and line citations. |
| **[PRD.md](docs/accountill/PRD.md)** | **Product Requirements**: Problem statement, target personas, 7-step core flow, MoSCoW priorities, and acceptance criteria. |
| **[ARCHITECTURE.md](docs/accountill/ARCHITECTURE.md)** | **Rebuild Blueprint**: Component topology, external services matrix, state residency, and key design decisions. |
| **[DATA_MODEL.md](docs/accountill/DATA_MODEL.md)** | **Data Models & Schemas**: Entity definitions, relationships, indexes, consumer files table, and Mermaid ERD. |
| **[API.md](docs/accountill/API.md)** | **API Specification**: Rebuild endpoints plus complete forensic audit of all 24 original endpoints and 11 UI screens. |
| **[GAPS.md](docs/accountill/GAPS.md)** | **Code Review**: 10 ranked vulnerabilities with code citations, impacts, fixes, and 2 strategic rebuild improvements. |
| **[AGENT_LOG.md](docs/accountill/AGENT_LOG.md)** | **Audit Trajectory**: Detailed chronological log of Prompts 1 through 11 and forensic corrections. |

---

## Core Gaps Identified in Original Architecture

1. **Unmounted Auth Middleware (BOLA)**: Endpoints lack route-level authorization guards, exposing CRUD operations to unauthenticated callers.
2. **Global Customer Directory Leakage**: `getClients` returns customer contacts and PII across all tenants without scoping.
3. **Hardcoded PDF Disk Overwrite Collision**: Both `/create-pdf` and `/send-pdf` write to a static file `server/invoice.pdf`, creating critical concurrency race conditions.
4. **Unverified Google OAuth Token Decoding**: `jwt.decode()` is called without verifying cryptographic signatures against Google public keys.
5. **Blind Server Trust of Client Calculations**: Backend accepts line item prices, taxes, and totals without server-side recalculation.
6. **Sequential Numbering Race Condition**: Invoice serial numbers are derived from client-side counts, risking duplicates under concurrency.
7. **Unhandled Promise Rejections & Double Responses**: Error callbacks invoke `res.send(Promise.reject())` without return statements.
8. **Redux Infinite Loading Spinners**: Missing `END_LOADING` dispatches in catch blocks trap the UI in loading skeletons on failure.
9. **Hardcoded Remote Domain in Password Reset Links**: Password recovery links hardcode `https://accountill.com/reset/${token}`.
10. **Dead & Unmounted Code**: Unmounted controller functions (`getInvoices`, `getProfilesBySearch`) and legacy naming references.

---

## Strategic Innovations & Rebuild Roadmap

1. **Secure Client Shareable Portal**: Tokenized public link (`/view/invoice/:shareToken`) for friction-free viewing, PDF downloading, and payment tracking.
2. **Integrated Online Payments (Stripe / Razorpay)**: Embedded checkout with webhook listeners auto-advancing invoice status to `PAID`.
3. **Automated Recurring Billing & Smart Dunning**: Cron workers auto-generating monthly retainer invoices with automated reminder schedules.
4. **Modern Full-Stack Stack Upgrade**: Modernization with **Next.js 14 / Vite + React 18 + TypeScript + Tailwind CSS / Shadcn UI**.
5. **Multi-Currency & Real-Time Forex**: Dynamic exchange rate API integration to invoice in foreign currencies while tracking base-currency revenue.
6. **Customizable Drag-and-Drop Invoice Designer**: Selectable invoice themes (Modern, Minimalist, Corporate) with custom fonts and branding.
7. **Organization Accounts & Granular RBAC**: Multi-user permissions supporting Owner, Accountant (read-only), and Project Manager roles.
8. **Financial Ledger & Accounting Exports**: One-click export to CSV/Excel formatted for QuickBooks, Xero, and quarterly tax reporting.
9. **Real-Time Activity Webhooks & Audit Trail**: Event logging (`invoice.opened`, `invoice.paid`) with outbound webhooks to team communication tools.

---

## Authors & Credits

Crafted with precision by **Team Encode**.
