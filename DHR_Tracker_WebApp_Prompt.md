# PROMPT: Build a "DHR Document Tracker" Web App (GitHub → Netlify)

## 1. Role & Goal
You are an expert full-stack web developer. Build a web application to track DHR (Device History Record) documents for a medical-device manufacturing company (ISO 13485 environment). The DHR team receives documents (Job Cards, Sub-Assembly Cards, Assembly Cards), and each document must pass 3 processes: **Review → Scan → Compilation**. The app must record receiving, member-wise work logs, pending observations with full traceability, targets, trackers and a graphical dashboard.

**This is NOT an Excel project. Do not generate .xlsx, macros or VBA.**

## 2. Technical Stack & Constraints (mandatory)
- **Frontend:** React + Vite (or plain HTML/JS if simpler), Tailwind CSS, Chart.js (or Recharts) for charts. Font: Arial. Fully responsive (desktop first, usable on tablet).
- **Hosting:** static build deployed on **Netlify**, source code on **GitHub** (Netlify auto-deploys on every push to `main`).
- **Database:** **Supabase** (free tier, Postgres). Netlify only hosts static files, so all shared data lives in Supabase. Use Row Level Security. All logic that can be done in SQL (views / functions) should be done there for speed.
- **Auth:** Supabase email + password login. Roles: `admin` (manages members, masters, targets, import) and `member` (enters receiving and own work log).
- **Limited internet:** keep the app light (small bundle, paginate/virtualise long tables, no heavy libraries). Make it a **PWA** (service worker, installable) so the app shell loads offline; show a clear "Offline – changes will sync" banner and queue entries in IndexedDB until connection returns.
- **Secrets:** Supabase URL and anon key go in Netlify environment variables (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`). Never commit keys. Add `.env.example`.
- **Colour code:** Yellow = user input fields, Grey = auto/calculated fields, Dark-blue headers with white text. Red bold for warnings.
- **Capacity:** Receiving 8,000+ rows; work log 30,000+ rows; pending trace 3,000+ rows. Use server-side pagination and indexed columns.
- **Data privacy:** repository must be private; no real company data in the repo.

## 3. Masters (Setup page, admin only)
| Document Type | Prefix |
|---|---|
| Job Card | J |
| Sub-Assembly | S |
| Assembly Card | A |

- **Members:** Krimal Sathavara, Avinash Dutta, Harsh Varia, Suraj Tagore, Rahul Kumar (editable, up to 20; add / deactivate, never hard-delete members that have entries).
- **Process Types (fixed):** Review, Scan, Compilation.
- **Status:** Done, Pending.
- **Observation Documents (dropdown, free text also allowed):** Incoming Inspection Report, COA, BI Report, Drawing, First-Piece Inspection, In-Process Inspection, Final Inspection, Job Card, Sub-Assembly Card, IFU, MFC, TC, GDP, Sign Pending, Missing Entry, Wrong Entry.
- **Year Code:** 26 (used in document numbers; editable).

## 4. Document Number Rules
- **Format:** Prefix + Year + "-" + 4-digit card no. Examples: J26-1710, S26-0604, A26-3032.
- **Job Card / Sub-Assembly:** user types only the 4-digit Card No. (e.g. 1710). Document No. is auto-generated (J26-1710) using the Year Code.
- **Assembly Card:** user does NOT type a Card No. The user types the **11-digit Lot No.** and the Assembly Card No. is extracted automatically.
  - Lot structure: `[3 digits][4-digit card no.][MM][YY]`. Example: Lot 43430320826 → card 3032, year 26 → **A26-3032**.
  - Logic: card = substring(lot, 4, 4), year = last 2 digits.
- If a full number such as "J26-1710" or "A25-2015" is typed in the Card No. field (e.g. imported past data), use it as is.
- Implement this as one shared function (frontend) and one SQL function (backend) so both always agree.

## 5. Pages & Features

### 5.1 Receiving (data entry)
Fields: **Date | Team Member | Document Type | Catalogue No. | Card No. (Job/Sub-Assy) | Document No. (auto) | Lot No. | Check (auto) | Remarks**
- **Date:** defaults to today when the entry is created; user can change it. Stored as a fixed value.
- **Team Member, Document Type:** dropdowns. Member defaults to the logged-in user.
- **Card No.:** disabled (grey) when Document Type = Assembly Card, with the message "taken automatically from Lot No."
- **Lot No.:**
  - Assembly Card: **mandatory, exactly 11 digits, numbers only, no decimal point.** Show inline validation error.
  - Job Card / Sub-Assembly: optional.
  - Store as text so leading zeros are kept.
- **Check messages:** "⚠ Duplicate Doc No.", "⚠ Lot No. mandatory" (Assembly only), "⚠ Lot No. must be 11 digits" (Assembly only), "⚠ Invalid Lot No.", otherwise "OK". Warnings red bold. Block saving when there is an error other than duplicate (duplicate shows a confirm).
- Table with search, filter by date/member/type, sort, pagination, inline edit (admin) and CSV export.

### 5.2 Member Work Log (one view per member – a member dropdown/tab, not separate sheets)
- Header: member name, one-line instruction note. Members can only add entries for themselves; admin can view all.
- Fields: **Date | Document Type | Card No. (4-digit) / Lot No. | Document No. (auto) | Lot No. (auto) | Catalogue No. (auto) | Process Type | Status | Observation Document | Reference Document | Remarks | Check**
- **Document No.:** built from Doc Type + 4-digit card no., or from an 11-digit Lot No. for Assembly Cards (same rule as Receiving). Lot No. and Catalogue No. are looked up from Receiving.
- **Status = Pending:** Observation Document (dropdown) and Reference Document are **mandatory**; fields highlighted red until filled.
- **Status = Done:** no observation needed.
- **Clearing a pending point:** the member adds a NEW entry for the same document + process with Status = Done. Existing Pending entries are never overwritten (traceability). Entries are append-only; edits/deletes only by admin and logged in an audit table.
- **Check messages, in priority order:**
  1. "⚠ Doc not in Receiving"
  2. "⚠ Select process"
  3. "⚠ Select status"
  4. "⚠ Observation document mandatory"
  5. "⚠ Reference document mandatory"
  6. "⚠ Duplicate – already <Status> by <Member> on <date>"
  7. "OK"
- **Duplicate rule (across ALL members):**
  - Same Document No. + same Process + same Status = duplicate. Only the later entry is flagged.
  - Krimal S26-0604 Review Done, then Avinash S26-0604 Review Done → Avinash's entry is DUPLICATE.
  - Krimal S26-0604 Review Pending, then Avinash S26-0604 Review Done → NOT a duplicate (clearing).
  - For Pending entries, a duplicate also needs the same Observation Document and Reference Document, so several different observations on one card are allowed.
  - Do NOT use Lot No. for duplicate checks (some lots are shared by multiple cards).
  - Duplicates are excluded from all member counts.
- Quick-entry UX: keyboard friendly, Enter to save and clear for the next row, last-used process/status remembered.

### 5.3 Data layer (Supabase)
Tables: `members`, `masters` (doc types, observation docs, settings such as year code), `receiving`, `work_log`, `targets`, `audit_log`.
Views / functions (SQL):
- `work_log_enriched`: adds Document No., Lot No., Catalogue No., key = DocNo|Process|Status, member count flag (first occurrence only, not a duplicate), duplicate flag + first-occurrence reference.
- `pending_state`: Open / Cleared — "Cleared" if a Done entry exists for the same DocNo + Process, else "Open"; with Cleared On and Cleared By (earliest Done entry after the pending date).
- `process_pending`, `pending_trace`, `dashboard_*`, `tracker_*` views as described below.
Add indexes on document_no, process, status, member_id, date.

### 5.4 Work Log (enquiry & trail page)
- **Inputs:** Date (default today, editable), Process Type filter (All / Review / Scan / Compilation), and a **Search box** accepting a 4-digit Card No., a full Document No., or a Lot No.
- **Section 1 – Matching documents (up to 10):**
  - Columns: Document No., Document Type, Catalogue No., Lot No., Received On, Received By, Review status, Scan status, Compilation status, Overall.
  - Status text: "Done · <member> · dd-mm-yy", "Obs. Pending · <observation doc> · <member>", or "Pending".
  - Colours: green = done, red = obs. pending, amber = pending.
- **Section 2 – Work Trail of ALL members (up to 150 rows, oldest first):**
  - Columns: #, Date, Member, Document No., Document Type, Lot No., Process, Status, Observation Doc, Reference Doc, Remarks, Pending State, Cleared On, Cleared By.
  - Filtered by the Process Type selection.

### 5.5 Process Pending (auto, document-wise)
- One row per received document.
- Columns: Document No., Document Type, Catalogue No., Lot No., Received On, Received By, Review, Scan, Compilation, Pending Processes, Overall.
- **Each process status:** "Done" if any Done entry exists; "Obs. Pending" if only Pending entries exist; "Pending" if no entry exists.
- **Pending Processes** lists what is left, e.g. "Scan, Compilation".
- **Overall:** Completed / In Process / Not Started.
- This is **document-flow** counting and must NOT change with member-wise logic.
- Filters, search, pagination, CSV export.

### 5.6 Pending Docs Trace (auto)
- **KPIs:** Total observations raised, Open, Cleared.
- Lists every Pending entry from all members.
- Columns: #, Date, Member, Document No., Document Type, Lot No., Process, Observation Document, Reference Document, Remarks, Status (Open/Cleared), Cleared On, Cleared By, Days Open.
- Status becomes Cleared automatically when a Done entry is logged for the same document + process.
- Filters: status, member, process, observation document, age bucket. CSV export.

### 5.7 Dashboard
- **KPI cards:** Total Documents Received, Completed (all 3 processes), In Process, Not Started, Open Observations.
- **Table 1 – Document flow (from Process Pending):** rows = Document Type × Process (9 rows) + Total; columns = Total Receiving | Done | No. of Pending Documents.
- **Table 2 – Member-wise (member dropdown):** same 9 rows; columns = Total Receiving (documents received by that member) | Done | Pending | Total Count.
- **MEMBER-WISE COUNTING RULE (important):**
  - Each member gets +1 for every unique (Document + Process + Status) they log, whether Done or Pending.
  - Krimal logs S26-0604 Review Pending → Krimal +1. Avinash later logs S26-0604 Review Done → Avinash +1. Krimal's +1 is **never reduced** when cleared.
  - Several observations on the same document/process by the same member count as 1.
  - Duplicates count 0.
  - Document-flow counts (Table 1) are not affected by this rule.
- **Charts:**
  1. Doughnut – Overall document status (Completed / In Process / Not Started, with %).
  2. Column – Open observations ageing (0-7, 8-15, 16-30, 31-60, >60 days).
  3. Stacked bar – Done vs Pending by Document Type × Process.
  4. Combo – Daily trend for last 30 days: Received (columns) plus Review/Scan/Compilation counts and observations raised (lines). End date is editable.
  5. Stacked column – Member-wise Done + Pending count per process, with a "Pending Raised" line on a secondary axis.

### 5.8 Target
- **Target Master (admin input):** Member × Process × Document Type → Daily Target (45 rows).
- **Checker inputs:** Member, Process, Document Type (incl. "All"), From Date, To Date.
- **Checker outputs:** Working days (exclude Sunday – weekly off); Daily Target; Target for the period; Achieved (member count rule: Done + Pending entries); Achievement %; Result: Achieved / Not Achieved / No target set.
- **All-members grid** for the selected period: Target, Done, Result per process.

### 5.9 Tracker (Daily / Weekly / Monthly)
- **One Reference Date input** (default today).
- **Three blocks:** Daily (that date), Weekly (Mon–Sun of that week), Monthly (that month).
- **Each block:** rows = members + Total; columns = Received, Review count, Scan count, Compilation count (member count rule), Obs. Raised, Obs. Cleared.

### 5.10 Admin pages
- **Members:** add / deactivate / reorder; create login for the member.
- **Masters:** edit observation documents, year code, doc type prefixes.
- **Import:** CSV upload screens for the three past-data files below, with preview, validation and an import summary (counts) before committing.
- **User guide:** a Help page with how to enter receiving, log work, clear a pending point, and add a new member.

## 6. Import of Existing Past Data
Provide CSV import (admin only). Load as static values.

**(a) Receiving file** — Columns: Date, Team Member, Document Type, Catalogue No., Card No. (full no., e.g. A26-3032), Lot No., Remarks. Load oldest first; store the full number as given.

**(b) Work Log export** — Columns: Date, Team Member, Process, Doc Type, Doc No., Status, Observation, Referenced Doc No. This holds only the latest status per document + process. Import only its **Done** rows.

**(c) Pending Documents file** — Columns: Date, Document Number, Document Type, Team Member (Entered By), Process Type, Status, Observation, Ref Doc No. It holds every observation. Import all its rows as **Pending** entries. Do not use the Work Log's pending rows (only one observation each).

**Other import rules**
- Keep past values exactly as written: 2025 numbers such as A25-xxxx, free-text observations, past Assembly Cards whose Lot No. does not match the card number.
- Imports must be idempotent (re-running must not create duplicates) and show rejected rows with reasons.
- Expected totals after import (for checking): 3,225 received; 7,789 Done entries; 1,573 open observation points over 977 document-process pairs; 0 duplicates.

## 7. Acceptance Tests
1. Receiving: Job Card + card 9876 → J26-9876.
2. Receiving: Assembly Card + Lot 43499990826 (Card No. blank) → A26-9999.
3. Receiving: Assembly Card + Lot 1234 → blocked / "⚠ Lot No. must be 11 digits".
4. Work log: Assembly Card + 11-digit Lot No. (or the 4-digit card no.) finds the same document.
5. Krimal: S26-0604 Review Pending (observation and reference filled) → Process Pending shows Review = Obs. Pending; Trace shows Open; Krimal count +1.
6. Avinash: S26-0604 Review Done → Review = Done, Trace = Cleared (with date and Avinash), Avinash +1, Krimal still +1.
7. Avinash enters S26-0604 Review Done again → "⚠ Duplicate…", not counted.
8. Pending without an observation document → red fields + "⚠ Observation document mandatory".
9. Work Log search "0604" → shows the document, its 3 statuses and the full trail of all members in date order.
10. Dashboard document-flow totals do not change when member counts change.
11. No NaN, undefined, null or error values displayed anywhere; no console errors.
12. App loads and the shell works offline; queued entries sync when back online.
13. A member cannot add entries under another member's name.

## 8. Deliverables
1. Complete source code in a GitHub-ready folder:
   ```
   dhr-tracker/
     src/ (pages: Dashboard, Receiving, WorkLog, MemberLog, ProcessPending, PendingTrace, Target, Tracker, Setup, Import, Help)
     supabase/schema.sql      (tables, views, functions, indexes, RLS policies)
     supabase/seed.sql        (masters, members)
     public/ (PWA manifest, icons)
     netlify.toml             (build command, publish dir, SPA redirect /* → /index.html)
     .env.example
     README.md
   ```
2. **README.md** with step-by-step: create Supabase project → run schema.sql and seed.sql → create admin user → push to a private GitHub repo → connect repo to Netlify (build: `npm run build`, publish: `dist`) → add the two environment variables → deploy → import past data.
3. Short user guide inside the app (Help page).

## 9. Build Order
Build in stages and test after each: (1) schema + masters, (2) auth + Setup, (3) Receiving, (4) Member Work Log with Check logic, (5) views for Process Pending / Pending Trace, (6) Work Log search page, (7) Dashboard + charts, (8) Target, (9) Tracker, (10) Import, (11) PWA/offline, (12) run all acceptance tests with sample data, then clear the sample data.
