# Vendor Portal — Project Roadmap v4
> Specs & Approach Document — no implementation code

---

## Project Summary

An independent bilingual (Vietnamese + English) web portal serving two types of users: **vendors** and **portal admins**. Vendors log in using their **email address**, confirm or reject Sent RFQs, edit Delivery Orders (quantities and delivery date), digitally sign and print DOs, and track delivery status through to store receipt confirmation. The portal also supports returns via Return Purchase Orders (RPO) and Goods Return Notes. Vendors can export data as PDF or CSV for invoicing and reconciliation. Portal admins share the same interface but have additional access to view all vendors, all POs and DOs across the system, trigger the Odoo sync manually, unlock signed DOs, and download any vendor's signed PDF. The portal runs on a separate VM from Odoo and integrates via Odoo's XML-RPC API using a dedicated service account. **Vendors only access the portal; stores only access Odoo.** All email is delivered via AWS SES.

---

## Confirmed Decisions

| Concern | Decision |
|---|---|
| Odoo version | 16 Community Edition |
| Odoo API protocol | XML-RPC (Python `xmlrpc.client`) |
| Vendor login identifier | Email address (managed on portal, NOT synced from Odoo) |
| Vendor password | Portal-owned, stored in portal PostgreSQL only |
| Profile data source | One-way sync from Odoo `res.partner` for profile fields (name, company, phone, tax ID). Email is managed on portal only — not synced from Odoo |
| Account provisioning | Auto from Odoo partners where `supplier_rank > 0` |
| Portal language | Vietnamese + English (bilingual, user-switchable) |
| HTTP client (frontend) | Native `fetch` API — no axios or third-party HTTP lib |
| Portal PO statuses | Waiting (`sent`), Confirmed (`purchase`), Cancelled (`cancel`) — Draft (`draft`) is not shown. Auto-cancel after 7 days past Expected Arrival if vendor has not confirmed or rejected |
| Portal DO statuses | Draft, Signed, Done, Cancelled |
| Odoo PO states (unchanged) | RFQ / RFQ Sent / Purchase Order / Cancelled — portal does not modify Odoo's base behaviour |
| DO per PO | Exactly 1 DO auto-created per confirmed PO — vendor cannot create additional DOs |
| DO delivery date | Single date for the entire DO (not per product line) |
| DO editing | Vendor edits delivery date + quantities (qty <= ordered qty from PO) |
| DO locking trigger | Vendor digital signature only — no 23:00 cutoff |
| DO signing | Digital signature on portal locks the DO, pushes delivery date + set quantities to Odoo Receipt |
| DO printing | After signing, vendor can print DO PDF (includes signature) multiple times |
| DO PDF language | Vietnamese only — all printed DO PDFs use Vietnamese labels |
| DO PDF content | PO number encoded as Code128 barcode (scannable by handheld), vendor info (ID, Tax ID, mobile, email), store ID (from `stock.warehouse.code`), PO confirmation date, delivery date, product table with single UoM column |
| UoM handling | Single UoM per product line, inherited from PO (e.g., Thùng 12 Chai, Kg). Vendor can only adjust quantity, not UoM. If UoM is wrong, PO must be re-created |
| Receipt confirmation | Store confirms Receipt in Odoo — sets final qty_done. DO status becomes Done, showing final received amounts |
| Returns | RPO (Return Purchase Order) + RN (Return Note / Biên Bản Trả Hàng). Vendor can only set pickup date and confirm — cannot change quantities. Must sign and print RN like DO |
| PO auto-cancel | If vendor does not confirm or reject within 7 days past Expected Arrival date, PO is auto-cancelled |
| Vendor PO confirmation | Vendor confirms Sent RFQ via portal → portal calls `button_confirm` on `purchase.order` in Odoo (no email notification) |
| Vendor RFQ rejection | Vendor rejects Sent RFQ via portal → portal calls `button_cancel` on `purchase.order` in Odoo + email to PO creator |
| Post-signature locking | DO locked after vendor signs — portal admin (buyer) can unlock directly in the portal (vendor notified by email, no reason required) |
| Data retention | 24 months — POs older than 24 months are permanently deleted from portal DB. Applies to all statuses |
| Data export | Vendors can export as PDF (individual or summary) or CSV. Single or bulk export. Date range filter available |
| Backorder handling | Vendor submits qty only — store validates and decides in Odoo |
| Admin role | Separate `admin_users` table, password set via env variable initially |
| Admin capabilities | View all vendors, trigger sync, view all POs/DOs, unlock signed DOs, download any PDF |
| Admin UI | Same layout as vendor portal with additional admin menu items |
| Signature capture | `signature_pad.js` → PNG sent to backend |
| PO list search | Filter by PO number and date range |
| Vendor dashboard summary | PO counts by status (Waiting, Confirmed, Cancelled) shown above PO list |
| Vendor comment on DO | Free-text note added when vendor signs the DO |
| PDF retention | 24 months — matching PO data retention |
| Responsive design | Works on both desktop and mobile equally |
| Vendor accounts | 1 Odoo partner = 1 portal account. Email login managed on portal (not synced from Odoo). No multi-user per vendor |
| Profile changes | All profile changes must go through Odoo — admin cannot edit in portal |
| Admin language | Bilingual (Vietnamese + English) — same as vendor portal |
| SSL / TLS | Handled at infrastructure level (load balancer / reverse proxy) |
| Audit logging | Key actions logged in DB: login, po_confirm, po_reject, do_update, do_sign, do_unlock, receipt_validated |
| Email notifications | Invite, password reset, PO rejection (to PO creator), receipt confirmed (to vendor with discrepancy alert if any), DO unlocked (to vendor) |
| Store email recipient | Email sent to the person who created the PO in Odoo (not a generic inbox) |
| Email service | AWS SES |
| Frontend stack | React + Vite |
| Backend stack | FastAPI (Python) |
| Database | PostgreSQL (portal-owned) |
| Cache / rate limiting | Redis |
| Deployment | Docker Compose on a separate VM, Nginx reverse proxy |

---

## Architecture Overview

```
Vendor Browser
     │  HTTPS
     ▼
Nginx (reverse proxy + TLS termination)
     ├── /        → React SPA (frontend container)
     └── /api/    → FastAPI (backend container)
                        │
                        ├── PostgreSQL  (vendor accounts, tokens, signatures)
                        ├── Redis       (rate limiting, token blacklist)
                        ├── AWS SES     (all outbound email)
                        │
                        └── Odoo 16 CE (separate VM)
                              XML-RPC  : portal → Odoo (read/write)
                              Odoo → portal : sync mechanism TBD (webhook/polling)
                              HTTP session : PDF download
```

**Key principle:** The React frontend never contacts Odoo. All Odoo communication is proxied through the FastAPI backend using a single service account. Vendor credentials never leave the portal's own database.

---

## Data Flow Summary

### Authentication flow
1. Vendor enters their **email address** and password on the login page
2. Backend looks up `vendor_users` by `email`
3. Password verified with bcrypt — same error message for all failure cases
4. On success, issues a JWT access token (30 min) and refresh token (7 days)
5. All subsequent requests carry the access token in the `Authorization` header
6. Expired access tokens are silently refreshed using the refresh token
7. On refresh failure, vendor is redirected to the login page

### Profile sync flow
1. Scheduled job runs every 6 hours, reads `res.partner` where `supplier_rank > 0`
2. **New partner:** creates `vendor_users` row (no password, inactive) with email initially copied from Odoo `res.partner.email`. Generates 24h invite token, sends welcome email via AWS SES containing the email (login) and set-password link
3. **Existing partner:** syncs profile fields only (name, company_name, phone, tax_id) — `email` and `hashed_password` are **never overwritten** by sync. Email is managed independently on the portal
4. **Partner deactivated in Odoo:** sets `is_active = FALSE` — vendor can no longer log in
5. Partners with no email in Odoo are skipped and logged for manual follow-up (email is needed for initial account creation only)

### Delivery Order (DO) flow
1. When a vendor confirms a PO, the portal auto-creates exactly 1 DO linked to the PO
2. Vendor edits the DO: sets delivery date and quantities per product line (qty must be <= ordered qty from PO)
3. Saves are incremental — vendor can save and come back multiple times while the DO is in Draft state
4. When ready, vendor clicks "Sign DO", draws their digital signature, and confirms
5. Backend verifies ownership, stores the signature PNG, generates the signed DO PDF (WeasyPrint + pypdf), and marks the DO as **locked** in the portal database
6. Backend pushes delivery date + set quantities to the corresponding Odoo Receipt via XML-RPC (these become pre-filled quantities in the Receipt, not yet `qty_done`)
7. Vendor can print the signed DO PDF multiple times — this is the document they bring to the store
8. The DO is now read-only in the portal — only a portal admin can unlock it (vendor is notified by email on unlock)

### Receipt confirmation flow (store-side, reflected on portal)
1. Vendor delivers goods to the store with the printed DO (2 paper copies, both parties sign, each keeps 1)
2. Store reviews the Receipt in Odoo — can adjust the pre-filled set quantities before confirming
3. Store confirms the Receipt in Odoo — `qty_done` is finalized
4. Portal receives notification that the Receipt is validated (sync mechanism TBD — webhook vs polling, pending team confirmation)
5. DO status on portal becomes **Done** — the final received amounts are shown on the DO
6. Portal sends an email to the vendor confirming receipt, alerting if any quantities differ between DO and receipt
7. Vendor can see both their DO quantities and the store's final received quantities on the DO detail page
8. Vendor can export the data as PDF or CSV for invoicing and reconciliation

### Sync mechanism (Odoo → Portal)
- **TBD — pending team confirmation.** Options under consideration:
  - (a) Webhook: Odoo sends notification to portal on receipt validation (requires lightweight Odoo module)
  - (b) Polling: portal periodically checks Odoo for state changes
  - (c) Nightly batch sync at 23:00
- Portal → Odoo communication (PO confirm/reject, DO push) is always real-time via XML-RPC

---

## Business Logic and Flow

This section describes the portal's behaviour in plain business terms, intended for stakeholder communication. It covers the four main scenarios: onboarding a new vendor, day-to-day portal usage, the delivery confirmation workflow, and exception handling.

---

### 1. Vendor Onboarding

**Trigger:** A new vendor is registered in Odoo with `supplier_rank > 0` and has a valid email address on their partner record.

**What happens automatically:**
1. The portal sync job runs every 6 hours and detects the new vendor in Odoo
2. A portal account is created, linked to the vendor's Odoo ID
3. The vendor receives a **Welcome Email** (in Vietnamese by default) containing:
   - Their **email address** — which they will use to log in
   - A **set-password link** valid for 24 hours
4. The vendor clicks the link, sets their own password, and their account becomes active
5. From this point, the vendor can log in at any time using their email and password

**If the vendor misses the 24h window:** they use the "Forgot Password" option on the login page, enter their email, and receive a new reset link.

**If the vendor has no email in Odoo:** the sync job skips them and logs the case. The internal team must add an email to the Odoo partner record — the account will be created on the next sync cycle.

**Profile updates:** If the vendor's name, phone, tax ID, or company name changes in Odoo, the portal reflects those changes automatically on the next sync. The vendor's **email** (login) and **password** are managed on the portal only and are never overwritten by sync.

---

### 2. Viewing Purchase Orders

**Who can see what:** Each vendor sees only their own Purchase Orders. It is technically impossible for a vendor to view another vendor's data.

**Which POs are visible (portal statuses):**
- **Waiting** — RFQ has been sent to the vendor and is awaiting confirmation. Vendor can confirm or reject it.
- **Confirmed** — PO has been approved, DO has been created. Vendor can edit and sign the DO.
- **Cancelled** — PO has been cancelled (vendor rejected, or store cancelled). Read-only.

Draft RFQs are not shown. Vendors can view PO data for **24 months** from creation date. Older POs are permanently deleted.

**Confirming a Sent RFQ:**
- A "Confirm PO" button is shown on Waiting PO detail pages
- Vendor clicks "Confirm PO" → portal calls `button_confirm` on `purchase.order` via XML-RPC using the service account
- Odoo updates the PO state from `sent` to `purchase` and generates the linked Receipt
- The portal auto-creates a DO for this PO and reflects the new Confirmed status immediately
- Once confirmed, the button is no longer shown — the PO-level record is now read-only

**Rejecting a Sent RFQ:**
- A "Reject" button is shown alongside the "Confirm PO" button on Waiting PO detail pages
- Vendor clicks "Reject" → portal calls `button_cancel` on `purchase.order` via XML-RPC
- Odoo updates the PO state from `sent` to `cancel`
- Portal sends an email notification to the PO creator (the store staff who made the RFQ) informing them the RFQ was rejected
- The store must create a new RFQ if they wish to re-order — rejection is final

**Auto-cancellation:**
- If a vendor does not confirm or reject a Waiting PO within **7 days past the Expected Arrival date** (`date_planned`), the portal automatically cancels it by calling `button_cancel` on `purchase.order` in Odoo
- A scheduled job checks daily for expired Waiting POs and cancels them
- Email notification sent to **both vendor and PO creator (store)** when a PO is auto-cancelled

**Searching and filtering:**
- Vendor can search by PO number (e.g. typing "PO004" filters the list instantly)
- Vendor can filter by date range (e.g. "POs from January to March")
- Results are paginated — 20 POs per page

**PO detail view:** clicking a PO shows the full list of ordered products with quantities and expected delivery dates, plus the linked DO with its status (Draft / Signed / Done / Cancelled).

---

### 3. Delivery Order & Delivery Workflow

This is the core business process of the portal. It replaces the need for vendors to call or email to confirm delivery quantities. The portal separates the **Delivery Order (DO)** — what the vendor plans to deliver — from the **Receipt** — what the store actually receives in Odoo.

```
Purchase Order confirmed on Portal
         │
         ▼
DO auto-created (1 per PO)
         │
         ▼
Vendor edits DO: delivery date + quantities per line
(qty must be <= ordered qty; can save multiple times)
         │
         ▼
Vendor clicks "Sign DO"
         │
         ▼
Vendor draws their digital signature on screen
         │
         ▼
DO is locked — delivery date + set quantities pushed to Odoo Receipt
Signed DO PDF generated (can be printed multiple times)
         │
         ▼
Vendor prints DO, brings goods + paper DO to store
         │
         ▼
Store receives goods, both parties sign 2 paper copies (each keeps 1)
         │
         ▼
Store reviews Receipt in Odoo (can adjust quantities)
         │
         ▼
Store confirms Receipt in Odoo — qty_done finalized
         │
         ▼
Portal notified (sync mechanism TBD)
DO status → Done, final received amounts shown
         │
         ├──▶ Email to vendor: receipt confirmed
         │    (alerts if any quantities differ)
         │
         └──▶ Vendor can export PDF/CSV for invoicing
```

**Key points for stakeholders:**
- The vendor edits the DO and signs — they do not trigger any stock movement in Odoo
- The DO pushes **set quantities** (pre-filled) to the Odoo Receipt, but these are not `qty_done` until the store confirms
- All stock validation decisions remain with the store in Odoo — the store can adjust quantities before confirming
- The signed DO PDF is the vendor's formal delivery document — they print it and bring it to the store
- When store confirms receipt, DO becomes Done and shows final received amounts alongside vendor's delivery amounts
- If quantities differ, the vendor is alerted in the notification email — they can log in to view details and export data

---

### 4. PO and DO Status Lifecycle

**PO Statuses (portal-managed):**

| Portal PO Status | Odoo PO State | Trigger | Vendor can do |
|---|---|---|---|
| **Waiting** | `sent` | Store sends RFQ | Confirm or Reject |
| **Confirmed** | `purchase` | Vendor confirms PO | View DO, export data |
| **Cancelled** | `cancel` | Vendor rejects or store cancels | Read-only |

**DO Statuses (portal-managed):**

| DO Status | Trigger | Vendor can do |
|---|---|---|
| **Draft** | PO confirmed, DO auto-created | Edit delivery date + quantities, save multiple times |
| **Signed** | Vendor signs DO digitally | Read-only, print DO PDF (multiple times). Data pushed to Odoo Receipt |
| **Done** | Store confirms Receipt in Odoo | Read-only, view final received amounts alongside delivery amounts, export PDF/CSV |
| **Cancelled** | PO cancelled (before receipt confirmation) | Read-only |

**Cancellation rules:**
- A PO can only be cancelled if the linked Receipt has **not** been confirmed by the store (no `qty_done`)
- If the Receipt has already been confirmed in Odoo, cancellation is blocked — the portal shows a warning and the vendor must contact procurement to resolve

---

### 5. Post-Signature Locking and Unlocking

**Why DOs are locked after signing:** Once a vendor confirms delivery quantities with their digital signature, the DO becomes the official delivery document. The signed PDF is what the vendor prints and brings to the store. Allowing edits after signing would undermine the document's integrity.

**What "locked" means in practice:**
- All quantity and date fields on the DO are read-only in the portal
- The signed DO PDF remains downloadable and printable at any time
- The vendor cannot re-sign or submit a new signature
- The delivery date and set quantities have already been pushed to Odoo's Receipt

**Unlocking process:**
If quantities were entered incorrectly and the vendor needs to resubmit, a portal admin can unlock the DO directly from the admin section (`/admin/vendors/:id`). Unlocking:
1. Removes the portal lock record
2. Sends an email notification to the vendor informing them the DO has been unlocked
3. The vendor can then update quantities/date and re-sign
4. The unlock action is logged in `delivery_orders` (unlocked_at, unlocked_by) and `audit_log`

---

### 6. Email Notifications Summary

| Event | Recipient | Language | Content |
|---|---|---|---|
| New vendor account created | Vendor | Vietnamese (default) | Vendor's email login + set-password link |
| Password reset requested | Vendor | Vendor's preferred language | Reset link (24h expiry) |
| Vendor rejects RFQ | PO creator (store staff) | Vietnamese | PO rejected + cancelled in Odoo |
| PO auto-cancelled (7 days) | Vendor + PO creator | Vendor's pref lang / Vietnamese | PO auto-cancelled due to no response within 7 days past Expected Arrival |
| Store confirms receipt | Vendor | Vendor's preferred language | Receipt confirmed notification. Alerts if any quantities differ between DO and receipt |
| RPO created by store | Vendor | Sent by Odoo (Send by Email) | Return order notification — vendor should log into portal to confirm |
| DO unlocked by admin | Vendor | Vendor's preferred language | Notification that DO has been unlocked for re-editing |

**Note:** No email is sent when a vendor confirms a PO (the confirmation is pushed to Odoo in real-time) or when a vendor signs a DO/RN (the data is pushed to Odoo automatically). The store email recipient is always the specific person who created the PO in Odoo, not a generic team inbox. RPO email is sent by Odoo natively (not by the portal).

---

### 7. What the Portal Does NOT Do

It is equally important for stakeholders to understand the boundaries of the portal:

- **Does not validate stock movements** — all Odoo validation is done by the store in Odoo
- **Does not create Purchase Orders** — vendors can confirm or reject a Sent RFQ but cannot create POs or edit PO lines
- **Does not handle invoicing or payments** — outside the scope of this portal (vendors can export data for their own invoicing)
- **Does not manage backorders** — the store decides on backorders in Odoo after reviewing quantities
- **Does not modify Odoo's base behaviour** — Odoo states and workflows remain unchanged
- **Does not expose any Odoo credentials to vendors** — vendors have no access to Odoo, directly or indirectly
- **Does not allow vendors to see other vendors' data** — enforced at every layer of the system
- **Does not give stores access to the portal** — stores only interact through Odoo

---

### 8. Returns: RPO & Return Note (Bien Ban Tra Hang)

The portal supports a returns workflow. When a store needs to return goods to a vendor, the process is handled through a **Return Purchase Order (RPO)** and a **Return Note (RN / Bien Ban Tra Hang)**.

**Returns workflow:**
1. Store creates a Return Purchase Order (RPO) in Odoo and clicks "Send by Email" — Odoo sends email to vendor asking them to log into the portal to confirm (same flow as PO/RFQ)
2. RPO appears on the vendor portal — vendor can see the products and quantities being returned
3. Vendor sets a **pickup date** (single date for entire RN) — this is when they will come to collect the returned goods
4. Vendor clicks **"Confirm & Sign"** — confirms the RPO, signs the RN digitally, and prints the RN PDF in one step. **Vendor cannot reject or cancel an RPO**
5. Vendor goes to the store on the pickup date to collect the returned goods, bringing the printed RN (2 copies, both parties sign)
8. Store confirms the return receipt in Odoo
9. RN status becomes Done on the portal

**Key differences from PO/DO flow:**
- Vendor **cannot change quantities** — only the pickup date
- Vendor **cannot reject** an RPO — they can only confirm and set a pickup date
- No auto-cancel rule for RPOs (unlike POs which auto-cancel after 7 days)

**RPO Statuses (portal):** Waiting, Confirmed — no Cancelled status for RPOs

**RN Statuses (portal):** Draft, Signed, Done — same lifecycle as DO (except no Cancelled)

**Visibility:** Returns are shown alongside regular POs in the vendor's portal, clearly labelled as returns. Vendors can filter to view only returns or only regular POs.

**Export:** Return data is included in the vendor's PDF/CSV export for reconciliation.

---

### 9. Data Export

Vendors can export delivery and receipt data for end-of-period invoicing and reconciliation.

**Export formats:**
- **PDF** — formatted report suitable for printing and archival. Two modes:
  - **Individual**: export a single PO/DO or RPO/RN as a standalone PDF
  - **Summary**: export a table of all selected POs/DOs in a date range as one PDF
- **CSV** — machine-readable format for vendor to edit and import into their own systems

**Export scope (all included):**
- PO number
- DO quantities (what vendor delivered)
- Receipt quantities (what store confirmed)
- Delivery date
- Receipt confirmation date
- Date range filter available (e.g., "export all data from March 2026")

**Export UI (no separate export page — integrated into existing pages):**
- **Dashboard**: checkbox selection on PO/RPO rows → "Export" button with format picker (PDF summary / CSV)
- **PO/DO detail page**: "Export" button for individual record (PDF individual / CSV)
- **Returns list**: same checkbox + export pattern as dashboard
- Bulk export generates a single file containing all selected records

**Export includes both regular POs/DOs and returns (RPO/RN).**

---

## Phase 0 — Project Setup & Infrastructure
**Effort: 0.5–1 day**

### Objectives
- Establish monorepo structure with `/frontend`, `/backend`, `/infra` folders
- Configure Docker Compose with all five services: frontend, backend, PostgreSQL, Redis, Nginx
- Set up environment variable strategy and secrets management
- Establish network connectivity between the portal VM and the Odoo VM
- Configure AWS SES: verify sender domain, create IAM user with SES send permissions, store credentials as secrets

### Repo structure
```
vendor-portal/
├── frontend/               # React + Vite
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/            # fetch wrapper + TanStack Query hooks
│   │   └── i18n/           # Vietnamese + English translation files
│   └── Dockerfile
├── backend/                # FastAPI
│   ├── app/
│   │   ├── api/            # route handlers
│   │   ├── services/       # odoo_client, pdf_service, email_service
│   │   ├── models/         # SQLAlchemy models
│   │   ├── schemas/        # Pydantic schemas
│   │   ├── jobs/           # sync_vendors, scheduler
│   │   └── core/           # config, security, dependencies
│   ├── alembic/            # DB migrations
│   └── Dockerfile
├── infra/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   ├── nginx/
│   │   └── nginx.conf
│   └── secrets/            # gitignored
├── .env.example
└── README.md
```

### Key decisions
- **Monorepo** keeps frontend and backend versioned together, simplifying deployment
- **Docker Compose** with separate dev and prod configs — prod uses Docker secrets for sensitive values
- **AWS SES setup:** verify the sending domain in SES, create a dedicated IAM user with `ses:SendEmail` permission only, store `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as Docker secrets. Use `boto3` in the FastAPI email service.
- **Odoo connectivity:** portal VM IP must be whitelisted on Odoo VM firewall for port 8069. A dedicated Odoo service account with an API key is the only credential the portal uses.

### Environment variables needed
- `ODOO_URL`, `ODOO_DB`, `ODOO_USERNAME`, `ODOO_API_KEY`
- `JWT_SECRET_KEY`, `JWT_REFRESH_SECRET`
- `DB_PASSWORD`, `REDIS_URL`
- `FRONTEND_URL` (for CORS and email links)
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SES_REGION`, `SES_SENDER_EMAIL`
- ~~`INTERNAL_NOTIFICATION_EMAIL`~~ — removed: store notifications go to the PO creator's email from Odoo, not a generic inbox
- `ADMIN_INITIAL_PASSWORD` (used to seed the first admin account on first startup)

---

## Phase 1 — Odoo Integration Layer
**Effort: 1–2 days**

### Objectives
- Build the Odoo XML-RPC client as a singleton service
- Implement the vendor-scoped query wrapper (row-level isolation)
- Implement and schedule the partner sync job
- Validate all required Odoo queries work correctly against the live instance

### Key design decisions
- **Singleton OdooClient:** authenticates once at startup, reuses the `uid` for all calls. Reconnects automatically on connection drop
- **VendorScopedOdooClient:** a wrapper that injects `[['partner_id', '=', partner_id]]` into every domain filter at the service layer — no route handler can accidentally omit vendor scoping
- **Sync job scheduling:** APScheduler running inside the FastAPI process, every 6 hours for vendor profiles. Can also be triggered manually via an internal admin endpoint
- **Odoo → Portal sync (TBD):** Mechanism for portal to learn when store confirms a Receipt is pending team confirmation. Options: webhook (requires Odoo module), polling, or nightly batch. Regardless of mechanism, portal will read confirmed `qty_done` via XML-RPC and update DO status to Done
- **PDF session:** a separate `requests.Session()` authenticated via `/web/session/authenticate` is maintained for PDF downloads — the XML-RPC `uid` cannot access `/report/pdf/` endpoints

### Odoo models in scope
| Model | Purpose | Key fields |
|---|---|---|
| `res.partner` | Vendor profile sync | `id`, `name`, `email` (initial account creation only), `company_name`, `phone`, `mobile`, `vat` (Tax ID), `supplier_rank` |
| `purchase.order` | PO list, detail, confirmation, rejection | `name`, `partner_id`, `state`, `date_order`, `date_planned` (Expected Arrival), `amount_total`, `order_line`, `picking_ids`, `user_id` (PO creator for email) — `button_confirm` or `button_cancel` called on vendor action |
| `purchase.order.line` | PO line items | `product_id`, `name`, `product_qty`, `qty_received`, `price_unit`, `product_uom` (UoM for this line) |
| `product.product` | Product info for DO PDF | `id`, `name`, `barcode` |
| `stock.picking` | Receipt header | `name`, `state`, `scheduled_date`, `origin`, `move_line_ids` |
| `stock.move.line` | Receipt line items | `product_id`, `product_uom_qty`, `qty_done`, `lot_id`, `state` |
| `stock.warehouse` | Store ID for DO PDF | `code` (short name, used as Store ID on printed DO) |

### Odoo 16 CE field name gotchas
- `stock.move.line.qty_done` → renamed to `quantity` in Odoo 17+
- `stock.picking.move_lines` → renamed to `move_ids` in Odoo 17+
- Always specify explicit field lists in `search_read` — reading all fields on Odoo 16 can trigger serialization errors on computed fields like `tax_totals`

### Data filtering rules (applied at backend, not frontend)
- POs: only `state IN ('sent', 'purchase', 'cancel')` are returned — `draft` and `done` are excluded
- Receipts: only `state IN ('assigned', 'done')` are returned
- All queries are scoped by `partner_id` via VendorScopedOdooClient

---

## Phase 2 — Auth System
**Effort: 1 day**

### Objectives
- Define the `vendor_users`, `admin_users`, `password_reset_tokens`, `delivery_orders`, `delivery_order_lines`, `return_notes`, `return_note_lines`, and `audit_log` database tables
- Implement JWT access + refresh token issuance and validation, with role claim (`vendor` or `admin`)
- Implement the vendor invite flow (first-time password set via token from AWS SES email)
- Implement the forgot-password flow for vendors (same token mechanism)
- Implement admin account seeding from `ADMIN_INITIAL_PASSWORD` environment variable on first startup
- Implement the FastAPI auth dependency used by all protected routes, with role-based access control

### Database tables

**`admin_users`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal ID |
| `username` | varchar, unique | login identifier for admin (not an Odoo ID) |
| `hashed_password` | varchar | seeded from `ADMIN_INITIAL_PASSWORD` on first startup |
| `full_name` | varchar | manually set |
| `email` | varchar | for notifications and password reset |
| `is_active` | boolean | TRUE by default |
| `created_at` | timestamp | |
| `last_login` | timestamp | |

**Admin account seeding:** On first application startup, if no rows exist in `admin_users`, the backend inserts a default admin account using `ADMIN_INITIAL_PASSWORD` from the environment. The admin must change this password after first login.

**`vendor_users`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal ID |
| `odoo_partner_id` | integer, unique | Odoo partner ID (used for data scoping, not for login) |
| `email` | varchar, unique | **login identifier** — initially copied from Odoo on account creation, then managed on portal only (never overwritten by sync) |
| `hashed_password` | varchar | NULL until invite accepted |
| `full_name` | varchar | synced from Odoo — contact person name |
| `company_name` | varchar | synced from Odoo |
| `phone` | varchar | synced from Odoo — mobile/phone number |
| `tax_id` | varchar | synced from Odoo `res.partner.vat` — for DO PDF |
| `preferred_language` | varchar | `'vi'` or `'en'`, set by vendor, default `'vi'` |
| `is_active` | boolean | FALSE until first password set |
| `created_at` | timestamp | |
| `last_login` | timestamp | updated on each successful login |

**`password_reset_tokens`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `user_id` | FK → vendor_users | |
| `token` | varchar, unique | `secrets.token_urlsafe(32)` |
| `expires_at` | timestamp | 24 hours from creation |
| `used` | boolean | marked TRUE after use |

**`delivery_orders`** (lock fields merged — no separate `do_locks` table)
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal DO ID |
| `po_odoo_id` | integer, unique | Odoo `purchase.order.id` this DO belongs to |
| `picking_id` | integer | Odoo `stock.picking.id` (linked Receipt) |
| `vendor_id` | FK → vendor_users | vendor who owns this DO |
| `delivery_date` | date | vendor-set planned delivery date |
| `status` | varchar | `draft`, `signed`, `done`, or `cancelled` |
| `signature_path` | varchar | NULL until signed; server-side path to stored PNG |
| `pdf_path` | varchar | NULL until signed; server-side path to signed DO PDF |
| `vendor_comment` | text | optional free-text note submitted by vendor at signing |
| `signed_at` | timestamp | NULL until signed |
| `unlocked_at` | timestamp | NULL unless admin unlocked |
| `unlocked_by` | FK → admin_users | admin who unlocked, NULL if not unlocked |
| `created_at` | timestamp | when DO was auto-created (on PO confirm) |
| `updated_at` | timestamp | last edit by vendor |

**`delivery_order_lines`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `do_id` | FK → delivery_orders | parent DO |
| `line_number` | integer | sequential row number |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | cached product barcode from Odoo |
| `product_name` | varchar | cached product name |
| `uom` | varchar | unit of measure, inherited from PO line (e.g., Thùng 12 Chai, Kg). Cannot be changed by vendor |
| `ordered_qty` | decimal | quantity from PO line (read-only reference) |
| `delivery_qty` | decimal | vendor-entered delivery quantity (must be <= ordered_qty) |
| `received_qty` | decimal | NULL until store confirms receipt; final qty_done from Odoo |

**`return_notes`** (same structure as delivery_orders, for returns)
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | internal portal RN ID |
| `rpo_odoo_id` | integer, unique | Odoo `purchase.order.id` (the RPO) |
| `picking_id` | integer | Odoo `stock.picking.id` (linked return receipt) |
| `vendor_id` | FK → vendor_users | vendor who owns this RN |
| `pickup_date` | date | vendor-set date to collect returned goods |
| `status` | varchar | `draft`, `signed`, or `done` (no cancelled — vendor cannot reject RPO) |
| `signature_path` | varchar | NULL until signed |
| `pdf_path` | varchar | NULL until signed |
| `signed_at` | timestamp | NULL until signed |
| `created_at` | timestamp | |
| `updated_at` | timestamp | |

**`return_note_lines`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `rn_id` | FK → return_notes | parent RN |
| `line_number` | integer | sequential row number |
| `product_id` | integer | Odoo `product.product.id` |
| `product_barcode` | varchar | cached product barcode |
| `product_name` | varchar | cached product name |
| `uom` | varchar | unit of measure from RPO line |
| `return_qty` | decimal | quantity being returned (read-only — set by store, vendor cannot change) |

**`audit_log`**
| Column | Type | Notes |
|---|---|---|
| `id` | serial PK | |
| `actor_type` | varchar | `vendor` or `admin` |
| `actor_id` | integer | `vendor_users.id` or `admin_users.id` |
| `action` | varchar | one of: `login`, `po_confirm`, `po_reject`, `po_auto_cancel`, `do_update`, `do_sign`, `do_unlock`, `rn_confirm_sign`, `receipt_validated` |
| `target_model` | varchar | e.g. `receipt`, `vendor` |
| `target_id` | integer | e.g. `picking_id`, `partner_id` |
| `detail` | jsonb | action-specific metadata (e.g. which lines were updated, old/new qty) |
| `created_at` | timestamp | when the action occurred |
| `ip_address` | varchar | client IP for login events |

### Auth endpoints
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/login` | Email + password → access + refresh tokens (role: `vendor`) |
| POST | `/api/admin/auth/login` | Username + password → access + refresh tokens (role: `admin`) |
| POST | `/api/auth/refresh` | Refresh token → new access token (preserves role) |
| POST | `/api/auth/set-password` | First-time invite or password reset via token (vendors only) |
| POST | `/api/auth/forgot-password` | Sends reset email via AWS SES by email address |
| POST | `/api/auth/logout` | Blacklists refresh token in Redis |

### JWT role claim
The JWT access token carries a `role` field (`vendor` or `admin`). All admin-only endpoints check for `role == 'admin'` via a dedicated FastAPI dependency. Vendor endpoints check for `role == 'vendor'`. A vendor token cannot access admin routes, and an admin token cannot be used to impersonate a vendor.

### Security considerations
- **Timing-safe login:** always run `bcrypt.verify` even when the email does not exist
- **Consistent error messages:** same message for unknown email and wrong password — never reveal whether the email exists
- **Token blacklist:** on logout, refresh token is added to a Redis set with TTL matching remaining token lifetime
- **Invite token expiry:** 24 hours. If expired, vendor must request a new one via forgot-password

---

## Phase 3 — Core API Endpoints
**Effort: 2–3 days**

### Objectives
- Implement all data endpoints with vendor-scoped Odoo queries
- Enforce DO locking logic server-side
- Implement the PDF generation and signing pipeline
- Implement webhook receiver for Odoo receipt validation
- Implement all email notification triggers via AWS SES

### Full endpoint list
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/vendors/me` | Current vendor profile |
| PATCH | `/api/vendors/me/language` | Update preferred language (`vi` or `en`) |
| GET | `/api/purchase-orders` | Paginated PO list with portal statuses, filterable by PO number and date range |
| GET | `/api/purchase-orders/{id}` | PO detail with order lines + linked DO + receipt comparison |
| POST | `/api/purchase-orders/{id}/confirm` | Confirm a Sent RFQ — calls `button_confirm` on Odoo; returns 409 if PO is not in `sent` state |
| POST | `/api/purchase-orders/{id}/reject` | Reject a Sent RFQ — calls `button_cancel` on Odoo + email to PO creator; returns 409 if PO is not in `sent` state |
| GET | `/api/delivery-orders/{do_id}` | DO detail with product lines, delivery date, delivery qty, and received qty (when Done) |
| PATCH | `/api/delivery-orders/{do_id}/lines` | Update quantities + delivery date on the DO (blocked if signed/locked); qty must be <= ordered qty |
| POST | `/api/delivery-orders/{do_id}/sign` | Submit signature PNG + optional comment, lock DO, push to Odoo Receipt, generate PDF |
| GET | `/api/delivery-orders/{do_id}/pdf` | Download signed DO PDF (available post-signature, can be called multiple times for printing) |
| GET | `/api/return-orders` | Paginated RPO list for current vendor |
| GET | `/api/return-orders/{id}` | RPO detail with return lines and linked Return Note |
| GET | `/api/return-notes/{rn_id}` | Return Note detail with product lines and pickup date |
| PATCH | `/api/return-notes/{rn_id}/pickup-date` | Set or update pickup date on the RN (blocked if signed) |
| POST | `/api/return-notes/{rn_id}/confirm-and-sign` | Confirm RPO + submit signature PNG in one step. Locks RN, generates PDF |
| GET | `/api/return-notes/{rn_id}/pdf` | Download signed RN PDF (same format as DO PDF) |
| GET | `/api/export` | Export data as PDF or CSV. Query params: `format` (pdf/csv), `type` (individual/summary), `ids` (for bulk), `date_from`, `date_to`. Includes POs, DOs, RPOs, RNs |
| POST | `/api/webhooks/odoo/receipt-validated` | Webhook receiver: Odoo notifies portal when Receipt is validated — triggers DO status → Done + vendor email |

### Admin-only endpoints
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/admin/vendors` | List all vendor accounts with status, last login, signed DO count, pending DO count |
| GET | `/api/admin/vendors/{partner_id}` | Drill into a specific vendor — their POs and DOs (read-only) |
| PATCH | `/api/admin/vendors/{partner_id}/deactivate` | Deactivate a vendor account |
| PATCH | `/api/admin/vendors/{partner_id}/reactivate` | Reactivate a vendor account |
| POST | `/api/admin/sync` | Manually trigger the Odoo partner sync job |
| GET | `/api/admin/sync/status` | Return last sync time, number of vendors synced, list of skipped vendors (no email) |
| GET | `/api/admin/delivery-orders/{do_id}/pdf` | Download any vendor's signed DO PDF |
| POST | `/api/admin/delivery-orders/{do_id}/unlock` | Remove the DO lock — sends email to vendor, vendor can update and re-sign |
| GET | `/api/admin/audit-log` | Paginated audit log view, filterable by actor, action type, and date range |

All admin endpoints require `role == 'admin'` in the JWT. They are prefixed with `/api/admin/` to make the distinction explicit and apply a separate rate limiting policy.

### PO list filtering
The `GET /api/purchase-orders` endpoint accepts the following optional query parameters:
- `q` — partial match on PO number (e.g. `PO004`)
- `date_from` — filter POs with `date_order >= date_from`
- `date_to` — filter POs with `date_order <= date_to`
- `page` and `page_size` — pagination (default page size: 20)

Filtering is applied server-side in the Odoo domain filter, not in the frontend.

### DO locking and unlocking approach
- When `POST /api/delivery-orders/{do_id}/sign` is called, the backend sets `status = 'signed'`, stores signature/PDF paths, and sets `signed_at` on the `delivery_orders` row
- Any subsequent `PATCH /lines` call on a signed DO returns HTTP 423 (Locked)
- The DO detail endpoint returns `status` — the frontend disables all editing when status is not `draft`
- On signing, the backend pushes delivery date + set quantities to the Odoo Receipt via XML-RPC
- **Unlocking:** a portal admin calls `POST /api/admin/delivery-orders/{do_id}/unlock`, which resets status to `draft`, sets `unlocked_at` + `unlocked_by`, clears signature/PDF paths, sends an email to the vendor, and logs the action. The vendor can then update quantities and re-sign.

### Validation approach (no portal-side `button_validate`)
Since the store decides on backorders in Odoo, the portal does **not** call `button_validate`. The vendor's role is:
1. Edit the DO: delivery date + quantities (qty <= ordered qty, multiple saves allowed)
2. Sign the DO (locks the portal record, pushes set quantities to Odoo Receipt)

The Odoo picking remains in `assigned` state after the vendor signs — the store sees the set quantities and validates in Odoo at their discretion. Only when the store confirms the Receipt does `qty_done` get finalized.

### Receipt validated (Odoo → Portal)
When the store confirms a Receipt in Odoo, the portal is notified (sync mechanism TBD — pending team confirmation):
1. Portal receives notification with `picking_id`
2. Portal reads the confirmed `qty_done` values from Odoo via XML-RPC
3. DO status on portal changes to **Done** — final received amounts are stored on DO lines
4. Portal sends email to vendor confirming receipt. If any quantities differ between DO and receipt, the email includes a discrepancy alert
5. Vendor can log into portal to view details and export PDF/CSV

### Audit logging approach
Every key action is written to the `audit_log` table synchronously within the same request. The logged action types are:

| Action | Trigger | Detail stored |
|---|---|---|
| `login` | Successful vendor or admin login | actor type, actor ID, IP address, timestamp |
| `po_confirm` | `POST /purchase-orders/:id/confirm` | PO ID, PO name, previous state (`sent`), new state (`purchase`) |
| `po_reject` | `POST /purchase-orders/:id/reject` | PO ID, PO name, previous state (`sent`), new state (`cancel`) |
| `po_auto_cancel` | Scheduled job (7 days past Expected Arrival) | PO ID, PO name, Expected Arrival date, days overdue |
| `do_update` | `PATCH /delivery-orders/:id/lines` | DO ID, list of lines with old and new quantities |
| `do_sign` | `POST /delivery-orders/:id/sign` | DO ID, vendor comment (if any), PDF path, signature path |
| `do_unlock` | `POST /admin/delivery-orders/:id/unlock` | DO ID, admin who unlocked |
| `receipt_validated` | Odoo → Portal notification | picking ID, DO status → Done, any quantity differences noted |

The audit log is viewable by admins via `GET /api/admin/audit-log`, filterable by actor, action type, and date range, paginated at 50 rows per page. It is never editable or deletable through the portal.

### Email notifications (AWS SES)

| Trigger | Recipient | Content |
|---|---|---|
| New vendor synced | Vendor | Welcome email with vendor's email login + set-password link |
| Forgot password requested | Vendor | Reset link (24h expiry) |
| Vendor rejects RFQ | PO creator (store staff) | PO rejected + cancelled in Odoo |
| PO auto-cancelled (7 days) | Vendor + PO creator | PO auto-cancelled — no response within 7 days past Expected Arrival |
| Store confirms receipt | Vendor | Receipt confirmed. Alerts if any quantities differ between DO and receipt |
| DO unlocked by admin | Vendor | DO has been unlocked for re-editing |
| RPO created by store | Vendor | Sent by Odoo natively (Send by Email button) — not via AWS SES |

**Not emailed:** Vendor confirms PO (data pushed to Odoo in real-time), Vendor signs DO/RN (data pushed to Odoo automatically).

All portal-sent emails use AWS SES in the vendor's `preferred_language` for vendor-facing emails, and in Vietnamese for store-facing emails. Store email goes to the PO creator's email (from Odoo `purchase.order.user_id`), not a generic inbox. RPO notification is sent by Odoo itself, not by the portal.

### DO / RN PDF pipeline approach
1. Check lock record — PDF only generated after signature is submitted
2. Generate the document as HTML rendered to PDF with WeasyPrint, **in Vietnamese**
3. Store the PDF on the server filesystem (retained for 24 months, matching PO retention)
4. Return the PDF as a binary response with `Content-Disposition: attachment` — vendor can call this endpoint multiple times to print

**Note:** The Return Note (RN) PDF uses the **same layout and content** as the DO PDF, with the title changed to "Bien Ban Tra Hang" and the delivery date replaced by the pickup date.

### DO PDF content specification
The printed DO PDF includes the following sections, all in Vietnamese:

**Header:**
- PO number displayed as a **Code128 barcode** (scannable by store's handheld device)
- Vendor ID / Mã NCC (`res.partner.id`)
- Vendor Tax ID / Mã số thuế (`res.partner.vat`)
- Vendor mobile phone / Số điện thoại (`res.partner.phone` or `mobile`)
- Vendor contact email / Email liên hệ (`res.partner.email`)
- Store ID / Mã cửa hàng (`stock.warehouse.code` — the warehouse short name)
- PO Confirmation Date / Ngày xác nhận đơn hàng
- Delivery Date / Ngày giao hàng (the single date set by vendor on the DO)

**Product table columns:**

| # | Column (Vietnamese) | Column (English) | Description |
|---|---|---|---|
| 1 | Số thứ tự | Line number | Sequential row number |
| 2 | Mã vạch | Barcode | Product barcode |
| 3 | Tên sản phẩm | Product name | Product description |
| 4 | Đơn vị tính | UoM | Unit of measure (inherited from PO, e.g., Thùng 12 Chai, Kg) |
| 5 | Số lượng giao | Delivery qty | Quantity vendor plans to deliver |
| 6 | Số lượng thực nhận | Received qty | Left blank — filled by store on paper |
| 8 | SL chênh lệch | Discrepancy qty | Left blank — for store to note differences |
| 9 | Ghi chú | Notes | Left blank — for store notes |

**Footer:**
- Vendor's digital signature (PNG embedded)
- Vendor comment (if provided at signing)
- Timestamp of signature
- Blank signature space for store's physical signature

---

## Phase 4 — React Frontend
**Effort: 3–4 days**

### Objectives
- Build all portal pages with clean, mobile-friendly UI
- Implement bilingual support (Vietnamese + English, user-switchable)
- Implement the `fetch`-based API client with automatic JWT refresh
- Integrate `signature_pad.js` for signature capture
- Implement the DO edit form with quantity validation (qty <= ordered) and lock state handling

### Internationalisation (i18n) approach
- Use `react-i18next` with two locale files: `vi.json` and `en.json`
- Language is stored in `vendor_users.preferred_language` (persisted server-side)
- A language toggle (VI / EN) is visible in the top navigation bar on all pages
- On language switch, the frontend calls `PATCH /api/vendors/me/language` and updates the i18next locale immediately — no page reload needed
- Default language: Vietnamese (`vi`)
- All UI strings, labels, error messages, and button text are translated
- Email content sent by backend follows the vendor's stored `preferred_language`

### Pages & routes
| Route | Page | Role | Description |
|---|---|---|---|
| `/login` | Login | Both | Email + password + language toggle |
| `/admin/login` | Admin Login | Admin | Username + password (separate login page) |
| `/set-password` | Set Password | Vendor | First-time invite and forgot-password reset |
| `/forgot-password` | Forgot Password | Vendor | Enter email to receive reset link |
| `/dashboard` | Dashboard | Vendor | PO list with search/filter, status badges, checkbox selection + "Export" button (PDF individual/summary or CSV) |
| `/purchase-orders/:id` | PO Detail | Vendor | Order lines + linked DO + receipt comparison (vendor qty vs store qty) |
| `/delivery-orders/:id` | DO Detail | Vendor | Product lines with editable delivery qty + single delivery date (locked if signed) |
| `/delivery-orders/:id/sign` | Sign DO | Vendor | Signature pad + comment field + final confirmation step |
| `/delivery-orders/:id/pdf` | DO PDF | Vendor | Download/print signed DO PDF (can print multiple times) |
| `/returns` | Returns List | Vendor | RPO list with search/filter, status badges |
| `/return-orders/:id` | RPO Detail | Vendor | Return lines + linked Return Note |
| `/return-notes/:id` | RN Detail | Vendor | Return product lines + pickup date picker (qty read-only) + Confirm & Sign |
| `/return-notes/:id/pdf` | RN PDF | Vendor | Download/print signed RN PDF |
| `/profile` | Profile | Vendor | Read-only vendor profile + language preference |
| `/admin/dashboard` | Admin Dashboard | Admin | Summary stats: vendor counts, pending DOs/RNs, sync status |
| `/admin/vendors` | Vendor List | Admin | All vendor accounts with status, last login, DO/RN counts |
| `/admin/vendors/:id` | Vendor Detail | Admin | Specific vendor's POs, DOs, RPOs, RNs (read-only drill-down) |
| `/admin/sync` | Sync Status | Admin | Last sync time, skipped vendors, manual trigger button |

### Admin dashboard content
The admin dashboard shows the following at a glance:
- **Vendor summary:** total vendors, active count, inactive count
- **Vendor table:** sortable list showing vendor name, Vendor ID, account status (active/inactive), last login date, number of signed DOs, number of pending DOs (Draft, not yet signed)
- **Sync status panel:** last sync timestamp, number of vendors synced, number of vendors skipped (no email), manual "Run Sync Now" button
- Clicking any vendor row navigates to `/admin/vendors/:id` — a read-only view of that vendor's POs and DOs, with a download button for any signed DO PDF and an unlock button on locked DOs

### Admin navigation
Admin users see the same top navigation bar as vendors, with additional items: **Vendors**, **Sync Status**, **Audit Log**. The language toggle is present — admin section supports both Vietnamese and English. Admin routes are protected — a vendor JWT cannot access `/admin/*` routes.

### HTTP client approach (native fetch)
- A single `apiFetch(path, options)` wrapper handles all API calls
- Attaches Bearer token from `localStorage` on every request
- On 401 response: calls `/api/auth/refresh`, stores new access token, retries once
- On refresh failure: clears storage, redirects to `/login`
- No direct `fetch` calls in components — all calls go through this wrapper

### State management approach
- **TanStack Query** for all server state — PO list, DO detail, profile
- **React local state** (`useState`) for form inputs, signature state, UI toggles
- No global state manager needed at this scope

### Responsive design approach
- The portal is fully responsive — works on desktop and mobile equally
- Tailwind CSS utility classes for layout (or equivalent CSS framework)
- Signature pad canvas resizes to fit the screen width on mobile
- Tables on mobile collapse to card-style layouts for readability

### Vendor dashboard summary
Above the PO list, the vendor dashboard shows summary cards:
- **Waiting** — POs awaiting vendor confirmation
- **Confirmed** — POs confirmed, DOs in various states
- **Cancelled** — POs that have been cancelled

### PO list search & filter UI
- Search bar for PO number (partial match, debounced 300ms before API call)
- Date range pickers for `date_from` and `date_to`
- Status badge on each PO row (Waiting / Confirmed / Cancelled) with colour coding
- DO status shown alongside PO status (Draft / Signed / Done / Cancelled)
- Pagination controls (previous / next), page size fixed at 20

### DO detail — visible fields per product line
| Field | Vietnamese | Source |
|---|---|---|
| Line number | Số thứ tự | Sequential row number |
| Product barcode | Mã vạch | from Odoo `product.product` → `barcode` |
| Product name | Tên sản phẩm | `delivery_order_lines` → `product_name` |
| UoM (read-only) | Đơn vị tính | `delivery_order_lines` → `uom` (inherited from PO line, cannot be changed) |
| Ordered quantity (read-only) | SL đặt hàng | `delivery_order_lines` → `ordered_qty` (from PO line) |
| Delivery quantity (editable) | Số lượng giao | `delivery_order_lines` → `delivery_qty` (must be <= ordered_qty) |
| Received quantity (read-only) | Số lượng thực nhận | `delivery_order_lines` → `received_qty` (NULL until store confirms; shows store's final qty_done) |
| Unit price | Đơn giá | from Odoo `purchase.order.line` → `price_unit` |
| Subtotal | Thành tiền | Calculated: unit price x delivery qty (frontend only, not stored) |

### DO detail behaviour
- Delivery date picker at the top of the DO form
- Each product line shows all fields above; delivery qty is the only editable field (plus delivery date)
- Validation: delivery qty must be <= ordered qty — frontend and backend both enforce this
- "Save" button submits `PATCH /delivery-orders/:id/lines` — can be pressed multiple times before signing
- If `locked: true` is returned by the API, all inputs are disabled and a lock notice is shown
- A "Sign DO" button at the bottom is enabled only when delivery date is set and all delivery qty values are > 0
- Once signed, the page shows a read-only summary with a "Print DO" button
- After store confirms receipt, the `received_qty` column is populated — vendor sees both their delivery qty and the store's received qty side by side

### PO detail — DO and receipt view
The PO detail page shows the linked DO with its status. When the DO is Done (store confirmed receipt):
- **Vendor DO quantities** — what the vendor planned to deliver
- **Store received quantities** — what the store actually confirmed in Odoo
- Per-line comparison, highlighting any differences between delivery and received quantities

### PDF history page
- Lists all DOs the vendor has signed, in reverse chronological order
- Each row shows: PO number, DO reference, delivery date, date signed, vendor comment (if any), download button
- Download calls `GET /api/delivery-orders/{do_id}/pdf` — PDFs are retained for 24 months (matching PO retention)

### Signature and comment capture
- `signature_pad.js` renders on an HTML5 canvas (resizes for mobile)
- Optional free-text comment field above the signature pad — vendor can describe delivery conditions or notes
- "Clear" button resets the canvas only
- "Confirm & Submit" sends both the signature PNG (base64) and the comment text to `POST /api/delivery-orders/:id/sign`
- Pad and comment field are disabled after successful submission
- A loading state is shown while the PDF is being generated server-side

### Key UX considerations
- Login field label: **"Email"** with helper text explaining it was provided in the welcome email
- DO lines show ordered qty, delivery qty, and received qty (when available) side by side for easy comparison
- Waiting POs show "Confirm PO" and "Reject" buttons prominently
- Lock notice on signed DOs: "Phiếu giao hàng đã được ký / Delivery Order has been signed"
- Comment field placeholder: "Ghi chú giao hàng / Delivery notes (optional)"
- All form submissions disable the button while in flight to prevent double submission

---

## Phase 5 — Security Hardening
**Effort: 0.5 day**

### Rate limiting strategy (SlowAPI + Redis)
| Endpoint group | Limit |
|---|---|
| `/api/auth/login` | 5 requests / minute / IP |
| `/api/auth/forgot-password` | 3 requests / minute / IP |
| `/api/auth/set-password` | 5 requests / minute / IP |
| All other `/api/` routes | 60 requests / minute / user |

### CORS
- Allow only the exact `FRONTEND_URL` — never wildcard `*` with credentials
- `allow_credentials=True`

### HTTPS / TLS
TLS termination is handled at the infrastructure level (load balancer or upstream reverse proxy) — the portal's Nginx does not manage SSL certificates. The portal Nginx listens on HTTP internally and trusts the upstream proxy to enforce HTTPS. Ensure the upstream proxy passes `X-Forwarded-Proto: https` and `X-Real-IP` headers to the backend.

### Nginx security headers
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Strict-Transport-Security: max-age=31536000` (set at upstream level if TLS is upstream)
- `Referrer-Policy: no-referrer`

### AWS SES security
- Dedicated IAM user with `ses:SendEmail` permission only — no other AWS permissions
- Sender domain verified in SES
- Credentials stored as Docker secrets, never in `.env` files in production

---

## Phase 6 — Production Deployment
**Effort: 0.5–1 day**

### Service restart policies
All containers: `restart: unless-stopped`. Health check on backend container (`GET /health`), 3 retries before marking unhealthy.

### Nginx reverse proxy routing
- All `/api/*` → FastAPI backend container
- All `/admin/*` → React SPA container (admin routes handled client-side by React Router)
- All other paths → React SPA container, catch-all serves `index.html`

### PostgreSQL backup
- Daily `pg_dump` via cron on the portal VM
- Compressed with gzip, stored in `/backups/`, 30-day retention
- Optionally synced to Azure Blob Storage
- PDF files stored on server filesystem (24-month retention) — include in VM backup strategy
- Scheduled cleanup job removes POs and associated DOs, PDFs older than 24 months

### Go-live checklist
- [ ] Odoo service account API key generated and connectivity tested from portal VM
- [ ] Firewall: portal VM → Odoo VM port 8069 open
- [ ] AWS SES sender domain verified, IAM credentials configured
- [ ] Verify PO creator email field is accessible via Odoo XML-RPC (`purchase.order` → `user_id` → `email`)
- [ ] `ADMIN_INITIAL_PASSWORD` set and first admin account seeded
- [ ] All production secrets set in Docker secrets files
- [ ] Upstream load balancer / proxy configured to forward HTTPS traffic
- [ ] Partner sync job run manually once — initial vendor accounts created
- [ ] Odoo → Portal sync mechanism configured and tested (webhook, polling, or batch — TBD)
- [ ] At least one test vendor: invite received → password set → login → dashboard visible → PO confirmed → DO edited + signed → DO PDF printed → delivered to store → store confirms receipt in Odoo → DO status Done → vendor email received
- [ ] Test RFQ rejection: vendor rejects → Odoo cancels → store email received
- [ ] Notification emails received by vendor and store
- [ ] Locked DO confirmed uneditable by vendor
- [ ] Admin can view vendor, download DO PDF, unlock DO (vendor email received)
- [ ] Audit log shows all actions from the test run
- [ ] Vietnamese and English UI both verified on desktop and mobile

---

## Summary Timeline

| Phase | Description | Effort |
|---|---|---|
| 0 | Repo setup, Docker Compose, Odoo API key, AWS SES | 0.5–1 day |
| 1 | Odoo XML-RPC layer + partner sync job | 1–2 days |
| 2 | JWT auth + role system + invite/reset flow + DB schema | 1–1.5 days |
| 3 | Core API endpoints + admin endpoints + PDF + emails | 3–4 days |
| 4 | React frontend (vendor portal + admin section, bilingual) | 4–5 days |
| 5 | Security hardening | 0.5 day |
| 6 | Production deployment + go-live checklist | 0.5–1 day |
| **Total** | | **~11–15 days** |

---

## Critical Gotchas

1. **Login uses Vendor ID (integer), not email** — frontend login field must be a number input; welcome email must clearly state the Vendor ID
2. **Admin login is separate** — admin uses username (not Vendor ID) on a separate login page `/admin/login`; admin JWT carries `role: admin`
3. **Profile sync never touches `hashed_password`** — only `full_name`, `email`, `company_name`, `phone` are ever overwritten by the sync job; admin cannot edit vendor profiles in the portal
4. **Portal does not call `button_validate`** — vendor submits qty_done and signs; Odoo validation is done by the internal team. The Odoo picking stays in `assigned` state after the vendor signs
5. **Receipt locking is portal-side only** — the `receipt_locks` table lives in the portal DB; Odoo is not aware of it. Admins unlock via the portal admin section
6. **Admin can unlock receipts — vendor cannot** — the unlock endpoint is admin-only; vendors have no self-service unlock
7. **PDF requires a separate HTTP session** — the XML-RPC connection cannot fetch `/report/pdf/` endpoints; a distinct `requests.Session()` authenticated via `/web/session/authenticate` is required
8. **PDFs are stored permanently** — no expiry or cleanup job; ensure the server filesystem or storage volume has adequate space and is included in the VM backup strategy
9. **Vendor comment is optional but must be stored** — even if empty, the `vendor_comment` field in `receipt_locks` must be explicitly stored (empty string, not NULL) to distinguish "no comment provided" from a data error
10. **Audit log is append-only** — no delete or update endpoints for `audit_log`; write failures should be logged but must not cause the parent action to fail (non-blocking)
11. **Odoo 16 field names** — `qty_done` on `stock.move.line` (not `quantity`), `move_lines` on `stock.picking` (not `move_ids`)
12. **Partners with no email are skipped** — sync job must log these; they cannot receive an invite and must be handled manually
13. **Timing-safe authentication** — bcrypt verification must always run, even for unknown Vendor IDs, to prevent user enumeration via response timing
14. **No axios** — all frontend HTTP calls use native `fetch` wrapped in a single `apiFetch` utility with automatic JWT refresh
15. **i18n covers both vendor and admin UI** — all portal pages including admin section support Vietnamese and English; vendor-facing emails follow `preferred_language`; internal alert emails are always in English
16. **TLS is upstream** — the portal Nginx does not manage certificates; ensure the upstream proxy passes `X-Forwarded-Proto` and `X-Real-IP` headers correctly to the FastAPI backend
17. **`button_confirm` is a write action on Odoo** — unlike all other vendor-facing data calls which are read-only, PO confirmation calls `execute_kw` with `button_confirm` on `purchase.order`. The Odoo service account must have write permission on `purchase.order`. Verify this permission explicitly — read-only service accounts will silently fail or return an access error
18. **Guard PO confirmation against double-submit** — the confirm endpoint must re-read the PO state from Odoo before calling `button_confirm`; if the state is already `purchase`, return 409 without calling Odoo again. The frontend must also disable the "Confirm PO" button immediately on click to prevent duplicate calls
