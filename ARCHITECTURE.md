# PBIS Digital People & Talent Ecosystem — Architecture (Phase 1: PBIS Careers)

## 1. Layered architecture

```
Presentation Layer      → index.html (public PBIS Careers site), admin.html (HR console)
Application Layer       → recruitment.js entity model + business rules (see §3)
Data Layer               → Google Sheets (Phase 1) behind an abstraction layer
Automation Layer         → Google Apps Script (Code.gs)
Storage Layer            → Google Drive (PBIS Recruitment / {Year} / {Job Title} / {Candidate})
Communication Layer      → Gmail via Apps Script (MailApp / GmailApp)
Authentication Layer     → Google Workspace SSO, role-based (HR Staff / HR Manager / Administrator)
Analytics Layer          → Sheet-derived aggregates, rendered in admin.html
```

The frontend never talks to Google Sheets directly — it calls a small set of
logical endpoints (below). This is what lets PBIS swap Sheets for Postgres/
Supabase later without touching index.html or admin.html.

## 2. Logical entities (data-model, DB-agnostic)

| Entity | Key fields |
|---|---|
| **Candidate** | id, name, email, phone, nationality, location, experienceYears, qualification, expertise, source, consentAt |
| **Application** | id (PBIS-HR-YYYY-#####), candidateId, jobId, status, coverLetter, cvDriveFileId, submittedAt, assignedHrId |
| **Job** | id, title, department, campus, type, level, description, responsibilities[], requirements[], deadline, status |
| **TalentPoolEntry** | candidateId, preferredRole, preferredDepartment, preferredCampus, joinedAt |
| **Interview** | id, applicationId, interviewer, date, time, format, status |
| **Email** | id, applicationId, template, sentAt, sentBy |
| **Task** | id, title, candidateId, assignedHrId, dueDate, priority, status |
| **User** (HR) | id, name, email, role (staff/manager/admin) |
| **AuditEvent** | id, entity, entityId, action, actorId, timestamp, diff |

`Candidate → Application → Job` is the recruitment core; `Candidate` is
designed to later gain `employeeId` once hired, enabling the Phase 3
Onboarding conversion without re-entering data (§35 of the brief).

## 3. API surface (abstraction layer)

The frontend calls these — implemented by Apps Script `doGet`/`doPost` today,
swappable for a real REST/GraphQL API later:

```
GET  /jobs                     list published jobs (filters: dept, campus, type)
GET  /jobs/{id}                job detail
POST /applications             create application (multipart: fields + CV file)
GET  /applications/track       ?ref=...&email=...  → status only, no HR notes
POST /talent-pool               join talent pool

# Authenticated (HR):
GET/PATCH /admin/applications/{id}   view/update status, notes
POST      /admin/emails              send templated email
POST      /admin/tasks               create follow-up task
GET       /admin/analytics           KPIs + funnel + sources
```

## 4. Google Sheets schema (Phase 1 data layer)

Each sheet = one entity above, one row per record, header row = field names.
Use generated IDs (never candidate name) as the primary key. Sheets: `Applications`,
`Candidates`, `Jobs`, `TalentPool`, `Interviews`, `Emails`, `Tasks`, `HRUsers`,
`AnalyticsCache`, `AuditLogs`, `Settings`.

## 5. Google Drive structure

```
PBIS Recruitment/
  2026/
    Secondary English Teacher/
      Sarah Mitchell/
        CV.pdf
        Cover Letter.pdf
        Supporting Documents/
```
Folders created automatically on submission (see `Code.gs → createCandidateFolder`).
Permissions are restricted to the HR Google Group — never publicly shared, and
URLs are never exposed to candidates or in public API responses.

## 6. Roadmap (do not build ahead of need)

| Phase | Module | Trigger to start |
|---|---|---|
| 1 (this build) | PBIS Careers & Recruitment | now |
| 2 | Recruitment Intelligence (scoring, AI CV summaries, source analytics) | once Phase 1 has ≥3 months of real application data |
| 3 | Employee Onboarding (Candidate → Employee conversion) | once first hires need onboarding tracked digitally |
| 4 | PBIS People Hub (directory, performance, leave, self-service) | once onboarding is stable |

The entity list in §2 and the sheet names in §4 already reserve space for
Phase 2–4 fields (e.g. `Candidate.employeeId`) so those phases are additive,
not a rebuild.

## 7. Security & privacy notes

### What's enforced automatically (as of this hardening pass)

- **Server-side authentication.** `admin.html`'s login screen is a UI convenience,
  not the real security boundary — `google.script.run` exposes every function in
  `Code.gs` to any browser that has a page open, regardless of what the UI shows.
  The actual authorization check now lives in `Code.gs`: `apiLogin()` verifies
  the loginId + hashed passcode against the server-side `AUTH_USERS` roster and
  issues a short-lived session token (6h TTL, via `CacheService`). Every
  sensitive action (currently: sending email) requires that token and is
  rejected server-side if it's missing, invalid, or expired — even if someone
  bypasses the login screen entirely via devtools.
- **Sheet formula-injection protection.** Every user-submitted string field is
  sanitized before being written to a Sheet cell (`sanitizeForSheet_`). Without
  this, a candidate could submit a "name" like `=IMPORTXML(...)` that executes
  as a formula when HR opens the spreadsheet.
- **Server-side file validation.** CV uploads are re-validated on the server
  (file extension allowlist, 10MB size cap) — the client-side checks in
  `index.html` are for UX, not security, since they're trivially bypassable by
  calling the API directly.
- **Honeypot spam guard + duplicate-submission guards** on both public forms
  (Applications, Talent Pool) — bots seeing a fake "success" response with
  nothing actually written, and repeat submissions from the same email
  deduplicated rather than creating duplicate records.
- **Generic error responses.** Server exceptions are logged via `console.error`
  (visible only in the Apps Script execution log) and never echoed back to the
  client — avoids leaking sheet names, stack traces, or other internals to
  anyone probing the API.
- **Drive folders and files are private by default**, set explicitly at
  creation time (defense in depth — both the folder and each file get their
  own `setSharing(PRIVATE, NONE)` call).
- Candidate data is never publicly queryable; the tracker endpoint returns
  status only, never HR notes or internal fields.

### What still needs a human to configure (can't be enforced in code)

- **Google Sheet sharing.** Open the Sheet's Share settings and confirm it's
  restricted to specific PBIS staff Google accounts — not "Anyone with the
  link." The deployment ID and Sheet ID living in this codebase (especially if
  the repo is public) means the URL alone shouldn't be treated as a secret;
  the Sheet's own permissions are the real gate.
- **Consider a Google Workspace-restricted deployment for `?page=admin`** if
  PBIS uses Google Workspace. Apps Script's "Anyone" access level applies to
  the whole deployment (can't be restricted per-page), so `index.html` (needs
  to stay public for candidates) and `admin.html` currently share one
  deployment's access level. A cleaner long-term setup is two separate Apps
  Script deployments — one public (candidate-facing), one restricted to your
  Workspace domain (HR-facing) — so the platform-level access control matches
  the session-token control already in place.
- **Rotate the 4 admin passcodes periodically**, and immediately if anyone
  who had one leaves the team — there's no per-user revocation short of
  changing the hash in `Code.gs`.
- **CORS caveat for non-embedded mode:** if `admin.html` is hosted outside
  Apps Script (not via `?page=admin`), a failed `fetch()` response can't always
  be distinguished from a successful-but-unreadable one due to missing CORS
  headers — the frontend optimistically reports "sent" in that case. This
  doesn't weaken the server-side auth check itself, only the confidence of the
  success message; serving `admin.html` via `?page=admin` (the recommended
  setup) avoids the ambiguity entirely via `google.script.run`.

### Original notes (still true)

- Consent text and checkbox are required on both the multi-step application
  and the Talent Community form (§39 of the brief).
- Rate limiting is bounded by Apps Script's own daily quotas (`MailApp` sends,
  execution time); there's no per-IP throttling since Apps Script web apps
  don't expose caller IP to `doGet`/`doPost`.
