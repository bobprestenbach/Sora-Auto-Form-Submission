# Toast Job Progress Log — Web App Build Plan

Companion to `docs/ToastJobProgressLog-LLM-Guide.md`, which describes the ChatGPT bot this app replaces.
Goal: a link anyone at Sora can open that takes a tech name, email, and appointment screenshots, walks the
tech through the same interview the bot performs, and submits the Cognito form for them — with no LLM in the loop
for the routine path.

---

## 1. What the bot actually does (decomposed)

| Step | What the bot does today | Needs AI? | How the web app does it |
|---|---|---|---|
| A. Read screenshots | Reads SA number, restaurant, type, status, date, times, customer, TR Project Owner, address from photos | **Only step where AI adds value** | OCR (Tesseract) + a parser written against real sample screenshots, then a **review screen** where the tech confirms/corrects every field. Optional AI fallback toggle (see §4). |
| B. Interview | Asks Main Category → Sub Category → Notes → Status → Owner per line, then attachments, then POS testing | No | A fixed wizard with dropdowns. The option lists are already known (guide §2). |
| C. Fill page 1 (Job Details) | Types defaults + extracted values | No | Deterministic Playwright script. |
| D. Fill page 2 (Progress Notes) | One line per interview answer | No | Playwright, "Add Item" per extra line. |
| E. Fill page 3 (Operational Status) | Maps appointment type + outcome → six dropdowns | No | Pure lookup table (guide §5), shown to tech with override before submit. |
| F. Fill page 4 (Wrap Up) | Test result, CST times, wrap-up status/date, attachments, Customer Notified = Yes | No | Playwright, including file upload. |
| G. Verify + submit | Reads back values, clicks Submit once, captures confirmation number | No | Playwright screenshots each page for audit, reads confirmation text, stores it. |
| H. Duplicate check + record keeping | Checks prior receipts, records SA + confirmation | No | Database table of submissions; block re-submit of same SA unless "correction" is checked. |

**Bottom line:** everything except reading the photos is deterministic. The LLM was only ever a browser driver and a
form wizard; both are cheap to replace with code. The one real question is how much to trust OCR versus a cheap
vision model for step A (§4).

---

## 2. Recommended architecture

```
 Browser (tech)                       Server (one Docker container)
 ┌────────────────────┐   HTTPS   ┌──────────────────────────────────────────┐
 │ Login: name + email│──────────▶│ Web app (Next.js, TypeScript)             │
 │ Upload screenshots │           │  - auth (domain‑restricted magic link)    │
 │ Review extraction  │           │  - upload → OCR (Tesseract) → parser      │
 │ Interview wizard   │           │  - interview state per SA (DB)            │
 │ Final review       │           │  - job queue                              │
 │ Live submit status │◀──SSE────│ Submission worker (Playwright + Chromium) │
 │ Results + receipts │           │  - fills Cognito form, screenshots pages  │
 └────────────────────┘           │  - stores confirmation #                  │
                                  └──────────────┬───────────────────────────┘
                                                 │
                                   Postgres (Supabase) + file storage
```

Why this shape:

- **One codebase, one container.** Next.js serves the UI and the API; a background worker in the same container runs
  Playwright. Keeps hosting simple and cheap (one small VM or a $5–10/mo container host).
- **Playwright, not the Cognito API.** Cognito's REST API only works for the form's owner (Toast). Sora is a submitter,
  so the only door is the public form in a browser. Playwright is headless Chromium and is free.
- **Postgres via Supabase** (free tier is plenty): submissions log, interview answers, users. Supabase Storage holds
  uploaded screenshots and attachments. SQLite on the VM is an acceptable simpler alternative if you'd rather have
  zero external services.
- **Domain‑restricted login.** Anyone with an `@sorapartners.com` email gets a magic link. No passwords to manage,
  and the link is useless to outsiders. (Alternative: Google sign‑in restricted to your Workspace domain.)

### Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend + API | Next.js 15 (App Router), TypeScript, Tailwind | Simple multi‑step wizard; no heavy state library needed |
| Browser automation | Playwright (Chromium) | Runs headless in the container; screenshots every page for audit |
| OCR | Tesseract (`tesseract.js` or the `tesseract` binary via `node-tesseract-ocr`) | Free, local, no usage fees |
| Image prep | `sharp` | Upscale/threshold screenshots before OCR to raise accuracy |
| DB | Postgres (Supabase) — or SQLite via `better-sqlite3` | Prisma or Drizzle for schema |
| File storage | Supabase Storage — or local disk volume | Screenshots + attachments; auto‑delete after N days |
| Auth | Auth.js (NextAuth) email provider, domain allow‑list | Needs an email sender (Resend free tier: 3k emails/mo) |
| Job queue | In‑process queue (`p-queue`) with DB‑backed job table | One submission at a time is fine; no Redis needed |
| Hosting | Docker on Railway / Fly.io / Render / a DigitalOcean droplet | Must allow Chromium; Vercel/Netlify serverless will **not** run Playwright reliably |
| Optional AI fallback | Anthropic Claude Haiku 4.5 vision, behind a feature flag | Off by default; see §4 |

---

## 3. User flow (UI spec)

1. **Landing / sign‑in**
   - Email field → magic link. On first visit also ask **Tech name**. Both are remembered.
   - These become the form's `Tech` and `Tech Email` values (guide §3 currently hard‑codes Jaice; the app makes it
     per‑user).
   - Admin‑editable defaults page: Partner Manager, Vendor Company, VC Email, time‑zone label (CST), OC email domain.

2. **Upload screenshots**
   - Drag‑and‑drop, multiple files, phone‑friendly (camera roll upload works on mobile).
   - Server runs OCR immediately and shows a progress bar.

3. **Extraction review** (this replaces "keep the browser visible so the user can watch")
   - One card per detected SA. Each field is an editable input pre‑filled from OCR, with a confidence colour
     (green = matched pattern cleanly, amber = guessed, red = not found).
   - Fields: SA, Restaurant, Appointment type (dropdown mapped to Primary Work Type), Outcome (Completed / CNC),
     Date, Scheduled start/end, Customer first+last, OC name, OC email (derived, editable), Country.
   - Buttons: "Add appointment manually", "Remove", "Split" (if OCR merged two rows).
   - Warnings surfaced here: future date, missing status, SA already submitted before (with date + confirmation #).

4. **Interview wizard, one SA at a time** (exact order from guide §2)
   - Progress note line: Main Category → Sub Category (dependent list) → Notes (250‑char counter) → Status → Owner
     → "Add another line?"
   - Attachments: Yes/No → file picker (40 MB combined cap enforced client‑side).
   - Installs only: POS testing → Fully tested / Not fully tested.
   - "Apply these answers to other appointments in this batch" checkbox (the guide allows explicit reuse).
   - Nothing is defaulted: the wizard will not let the tech skip a dropdown.

5. **Operational status + wrap‑up preview**
   - The six status fields are computed from type + outcome (guide §5 table) and shown with override dropdowns.
   - If any POS field is "Cannot Complete", the app checks that a matching Hardware/Network/Software note line exists
     and, if not, sends the tech back into the interview for that line (guide §5 conditional prompts).
   - Shows computed Wrap Up: test result, `9:00 AM CST` / `1:00 PM CST`, wrap‑up status, wrap‑up date, Customer Notified = Yes,
     Name of Customer Notified.

6. **Final review + authorize**
   - Full read‑back of every SA. Explicit **"Submit N appointments"** button. This is the authorization step the guide
     requires; nothing is submitted before it.

7. **Submission progress (live)**
   - Per‑SA status: queued → filling page 1…4 → verifying → submitted / failed.
   - Each page gets a screenshot the tech can expand. On failure the draft is preserved and the SA is marked
     "needs attention" with the reason; it is never auto‑retried past the Submit click.

8. **Results**
   - Table: SA, restaurant, date, outcome, confirmation number, timestamp, submitted by. Exportable as CSV.
   - "History" page shows all past submissions (this is the duplicate‑check source).

---

## 4. The screenshot problem — how to avoid AI fees without breaking the workflow

The screenshots come from a scheduling tool (fields like `SA-1234567`, "TR Project Owner", "Cannot Complete Details",
Scheduled Start/End). Screenshots of rendered text are the *easiest* case for OCR, so Tesseract will get most fields
right if we:

1. Pre‑process (upscale 2×, greyscale, sharpen) with `sharp`.
2. Parse with patterns written against **real samples** — e.g. `SA-\d{7}`, `Scheduled Start\s*[:\-]?\s*(.+)`,
   the row structure of the schedule list, the status values.
3. Always show the review screen (§3 step 3). The tech spends ~20 seconds confirming instead of typing everything.

Three tiers, cheapest first:

| Tier | Cost | When |
|---|---|---|
| **T0 — Manual entry** | $0 | Always available; the review screen *is* the manual form if OCR finds nothing |
| **T1 — Tesseract + parser** (default) | $0 | Expect 80–95% field accuracy on clean screenshots once the parser is tuned to your layouts |
| **T2 — Vision LLM fallback** (optional, off by default) | Fractions of a cent per screenshot on a small model | A per‑batch "Use AI extraction" toggle for blurry or unusual photos. A tech doing 10 appointments/day would spend well under $1/month. |

Recommendation: build T0 + T1 first, ship, measure how often techs have to correct fields, then decide whether T2
is worth wiring in. **The parser cannot be written without sample screenshots** — see §7.

---

## 5. Submission engine (Playwright) — design notes

- **Selector mapping first.** Cognito renders fields with labels, not stable IDs. Step one of development is a
  mapping script that opens the form, walks all four pages, and records label → control → option lists. It also
  serves as a **health check** that runs nightly: if a label or option disappears, the app flags "form changed"
  and disables submissions until the mapping is updated, instead of silently mis‑filling.
- **Fill order and re‑verification.** Set Primary Work Type *before* OC Email (changing work type rebuilds controls
  and clears values — guide §3). After each page, read every value back from the DOM and compare to intended;
  mismatch = stop, screenshot, mark SA "needs attention".
- **Emails.** The guide notes emails sometimes don't show in text snapshots. Verify via the input's `.value` and a
  screenshot, not accessibility text.
- **Attachments.** `setInputFiles` on the file control; wait for Cognito's upload indicator; verify filenames listed.
- **Submit exactly once.** Click, wait for the success message containing "reported to Toast" + confirmation
  number; store it. On timeout, do **not** re‑click — read the current page state, screenshot, and surface to the
  tech. Same‑SA re‑submission requires the tech to tick "this is a correction".
- **Concurrency.** One browser context per submission; process one SA at a time per worker. Plenty for a team.
- **Bot protection.** Need to confirm Cognito doesn't throw a CAPTCHA at headless Chromium (some forms enable it).
  If it does, options are running headed in a small VM with a persistent profile, or asking Toast whether they can
  disable it for this form.
- **Audit trail.** Every submission stores page screenshots, the exact values sent, who submitted, and the
  confirmation number. Uploaded source screenshots (which contain customer names/addresses) are auto‑deleted after
  a configurable window (suggest 30 days).

---

## 6. Data model (Postgres)

```
users            id, email, tech_name, role(admin|tech), created_at
batches          id, user_id, created_at, status
screenshots      id, batch_id, storage_path, ocr_text, ocr_json, created_at, purge_after
appointments     id, batch_id, sa, restaurant, work_type, outcome, sched_date,
                 sched_start, sched_end, customer_name, oc_name, oc_email, country,
                 extraction_confidence(json), status(draft|ready|submitting|submitted|failed)
note_lines       id, appointment_id, line_no, main_category, sub_category, notes, status, owner
attachments      id, appointment_id, storage_path, filename, size_bytes
ops_status       appointment_id, pos_hardware, pos_network, pos_software, pos_operations, gls, training
wrap_up          appointment_id, pos_test_result, start_cst, end_cst, wrap_status, wrap_date, customer_notified_name
submissions      id, appointment_id, submitted_by, submitted_at, confirmation_number, page_screenshots(json),
                 sent_values(json), error, is_correction
form_mapping     version, captured_at, mapping(json)   -- output of the selector‑mapping/health‑check script
settings         key, value   -- partner_manager, vendor_company, vc_email, tz_label, oc_email_domain, ai_fallback
```

---

## 7. What I need from you to build and connect this

**Must have before coding starts**

1. **Sample screenshots — at least 15–20**, covering: schedule list with several rows; single‑appointment detail;
   onsite install, remote install, onsite/remote training, GLS, post‑live, site survey; Completed and Cannot Complete
   (with the CNC details visible); one with an explicit OC email in notes; one phone photo of a screen if techs ever do
   that. Redact nothing — the parser has to see real layouts. (Customer data stays in the private repo / a private
   share, not in a public place.)
2. **Access to the live form for mapping/testing.** I can't reach cognitoforms.com from this sandbox. Either:
   (a) run the mapping script from your machine and send me its output, or (b) do that step in a session with open
   network access. Also: **is there a safe way to test a submission?** A test form from Toast, or agreement that one
   clearly‑labelled test entry (e.g. SA‑0000000, restaurant "SORA TEST — IGNORE") is acceptable. Without this we can
   only test up to the Submit click.
3. **Hosting decision.** Pick one: Railway / Fly.io / Render (Docker, ~$5–10/mo) or a small VPS you already have. It
   needs to run Chromium, so serverless platforms are out.
4. **Login method.** Magic‑link email restricted to `@sorapartners.com` (needs a Resend or SendGrid account — free
   tiers suffice) **or** Google sign‑in restricted to your Workspace (needs a Google Cloud OAuth client). Tell me which
   and I'll list the exact keys to create.
5. **Database/storage.** Supabase project (free tier) — you already have Supabase connected to this workspace, so
   this is the path of least resistance — or "just use SQLite on the server".

**Configuration values (confirm or change)**

| Setting | Current value from the guide |
|---|---|
| Partner Manager | Lindsey Shea |
| Vendor Company | Sora |
| VC | blank |
| VC Email | Service@sorapartners.com |
| Tech / Tech Email | now per‑user from login |
| OC email convention | `firstname.lastname@toasttab.com` |
| Time zone label | always `CST` on scheduled times |
| Customer Notified | always Yes |
| Duration | Single Day (multi‑day asks the tech) |
| Screenshot retention | proposed 30 days |

**Optional**

6. Anthropic API key **only** if you want the AI extraction fallback toggle (§4, T2). Not needed for v1.
7. A domain or subdomain (e.g. `forms.sorapartners.com`) if you want a branded link instead of the host's default URL.

---

## 8. Build phases

| Phase | Deliverable | Rough effort |
|---|---|---|
| 0 | Repo scaffold, Docker, CI, Supabase schema, magic‑link login | 1–2 days |
| 1 | Form mapping + health‑check script; Playwright engine filling all 4 pages up to (not including) Submit, with per‑page screenshots and read‑back verification | 2–3 days (needs form access) |
| 2 | Upload → OCR → parser → review screen, tuned on your samples | 2–3 days (needs samples) |
| 3 | Interview wizard, ops‑status/wrap‑up computation, conditional‑note enforcement, final review | 2 days |
| 4 | Submit path, confirmation capture, duplicate guard, history page, CSV export | 1 day (needs test‑submission plan) |
| 5 | Pilot with 2–3 techs, tune parser, decide on AI fallback | 1 week calendar |

Phases 1 and 2 can run in parallel once items 1 and 2 in §7 are provided.

---

## 9. Risks and open questions

- **Form changes.** Toast can edit the form at any time. Mitigation: nightly health check + "form changed" lockout +
  the mapping stored as versioned data rather than hard‑coded selectors.
- **CAPTCHA / bot detection on Cognito.** Unknown until tested headless. See §5.
- **OCR accuracy on phone photos** (vs. screenshots) will be noticeably worse; the review screen absorbs this, and
  T2 fallback exists if it becomes a daily annoyance.
- **Ambiguous names for OC email derivation** (nicknames, hyphens, two‑word first names) — the app shows the derived
  email and requires the tech to confirm it, never submits a derived email unseen.
- **Multiple techs submitting at once** — fine; the queue serializes browser sessions.
- **PII in screenshots** — retention window + private storage bucket + login required.
- **Duplicate submissions** — the guide's biggest concern. Handled by the submissions table check, the single‑click
  rule, and the "correction" checkbox.
