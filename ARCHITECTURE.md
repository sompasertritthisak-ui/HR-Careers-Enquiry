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

- Candidate data is never publicly queryable; the tracker endpoint returns
  status only, never HR notes.
- Consent text and checkbox are required on both the multi-step application
  and the Talent Community form (§39 of the brief).
- Rate limiting + duplicate-submission detection belong in `Code.gs`
  (check by email + jobId before creating a new Application row).
