# Toast Job Progress Log — Web App Build Plan

Companion to `docs/ToastJobProgressLog-LLM-Guide.md`, which describes the ChatGPT bot this app replaces.
Goal: a link anyone at Sora can open that takes a tech name, email, and appointment screenshots, walks the
tech through the same interview the bot performs, and submits the Cognito form for them — with no LLM in the loop
for the routine path.

---

## 1. What the bot actually does (decomposed)

| Step | What the bot does today | Needs AI? | How the web app does it |
|---|---|---|---|
| A. Read screenshots | Reads SA number, restaurant, type, status, date, times, customer, TR Project Owner, address from photos | **Yes — the only step** | The server sends each photo to a small vision model (Claude Haiku 4.5) that returns the fields as JSON, then a **review screen** where the tech confirms/corrects every field. Cost is well under a cent per photo (§4). |
| B. Interview | Asks Main Category → Sub Category → Notes → Status → Owner per line, then attachments, then POS testing | No | A fixed wizard with dropdowns. The option lists are already known (guide §2). |
| C. Fill page 1 (Job Details) | Types defaults + extracted values | No | Deterministic Playwright script. |
| D. Fill page 2 (Progress Notes) | One line per interview answer | No | Playwright, "Add Item" per extra line. |
| E. Fill page 3 (Operational Status) | Maps appointment type + outcome → six dropdowns | No | Pure lookup table (guide §5), shown to tech with override before submit. |
| F. Fill page 4 (Wrap Up) | Test result, CST times, wrap-up status/date, attachments, Customer Notified = Yes | No | Playwright, including file upload. |
| G. Verify + submit | Reads back values, clicks Submit once, captures confirmation number | No | Playwright screenshots each page for audit, reads confirmation text, stores it. |
| H. Duplicate check + record keeping | Checks prior receipts, records SA + confirmation | No | Database table of submissions; block re-submit of same SA unless "correction" is checked. |

**Bottom line:** everything except reading the photos is deterministic. The LLM was only ever a browser driver and a
under a cent per photo (§4).
vision model for step A (§4).

---

## 2. Recommended architecture

```
 Browser (tech)                       Server (one Docker container)
 ┌────────────────────┐   HTTPS   ┌──────────────────────────────────────────┐
 │ Tech name + email  │──────────▶│ Web app (Next.js, TypeScript)             │
 │ Upload screenshots │           │  - no login; name/email are form data     │
 │ Review extraction  │           │  - upload → vision model API → JSON       │
 │ Interview wizard   │           │  - interview state per SA (DB)            │
 │ Final review       │           │  - job queue                              │
 │ Live submit status │◀──SSE────│ Submission worker (Playwright + Chromium) │
 │ Results + receipts │           │  - fills Cognito form, screenshots pages  │
 └────────────────────┘           │  - stores confirmation #                  │
                                  └──────────────┬───────────────────────────┘
                                                 │
                                   Postgres (Supabase) + file storage
                                   Anthropic API (screenshot reading only)
```

**Nothing runs on anyone's personal machine.** The whole app — web pages, screenshot reading, and the robot browser
that fills the Cognito form — runs on one hosted server. Techs only open a link in their phone or laptop browser.

Why this shape:

- **One codebase, one container.** Next.js serves the UI and the API; a background worker in the same container runs
  Playwright. Keeps hosting simple and cheap (one small VM or a $5–10/mo container host).
- **Why the server needs Chromium.** This has nothing to do with logging techs in. Cognito Forms has no API for
  anyone but the form's owner (Toast), so the only way to submit is the same way a person does: open the form page,
  type into the fields, click Next through the four pages, upload attachments, click Submit, read the confirmation.
  Playwright drives a copy of the Chrome browser (Chromium) on the server to do exactly that — it is the replacement
  for the ChatGPT bot's "browser tool". It runs invisibly ("headless") and is free; the only requirement is that the
  host lets us install it, which any Docker‑based host or VPS does and serverless hosts (Vercel, Netlify) do not.
- **Postgres via Supabase** (free tier is plenty): submissions log, interview answers, users. Supabase Storage holds
  uploaded screenshots and attachments. SQLite on the VM is an acceptable simpler alternative if you'd rather have
  zero external services.
- **No user accounts.** The first screen asks for tech name and email; those are stored in the browser so the tech
  doesn't retype them, and they go straight into the form's Tech / Tech Email fields. See §9 for the one risk this
  creates and a cheap optional guard (a shared company passcode).

### Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend + API | Next.js 15 (App Router), TypeScript, Tailwind | Simple multi‑step wizard; no heavy state library needed |
| Browser automation | Playwright (Chromium) | Runs headless in the container; screenshots every page for audit |
| Screenshot reading | Anthropic API, Claude Haiku 4.5 (`claude-haiku-4-5`) with structured JSON output | Server‑side API call; ~$0.005 per screenshot |
| Image prep | `sharp` | Resize/compress uploads before sending to the model (keeps cost and upload time down) |
| DB | Postgres (Supabase) — or SQLite via `better-sqlite3` | Prisma or Drizzle for schema |
| File storage | Supabase Storage — or local disk volume | Screenshots + attachments; auto‑delete after N days |
| Auth | None (tech name + email captured as form data, remembered in the browser) | Optional shared passcode env var, see §9 |
| Job queue | In‑process queue (`p-queue`) with DB‑backed job table | One submission at a time is fine; no Redis needed |
| Hosting | Docker on Railway / Fly.io / Render / a DigitalOcean droplet | Must allow Chromium; Vercel/Netlify serverless will **not** run Playwright reliably |

---

## 3. User flow (UI spec)

1. **Landing**
   - Two fields: **Tech name** and **Tech email**. Remembered in the browser so they're pre‑filled next time.
   - These become the form's `Tech` and `Tech Email` values (guide §3 currently hard‑codes Jaice; the app makes it
     per‑tech). No password, no account.
   - Admin‑editable defaults page: Partner Manager, Vendor Company, VC Email, time‑zone label (CST), OC email domain.

2. **Upload screenshots**
   - Drag‑and‑drop, multiple files, phone‑friendly (camera roll upload works on mobile).
   - Server sends each photo to the vision model immediately and shows a progress bar.

3. **Extraction review** (this replaces "keep the browser visible so the user can watch")
   - One card per detected SA. Each field is an editable input pre‑filled from the model's JSON, with a confidence
     colour (green = read clearly, amber = model flagged it as uncertain, red = not found).
   - Fields: SA, Restaurant, Appointment type (dropdown mapped to Primary Work Type), Outcome (Completed / CNC),
     Date, Scheduled start/end, Customer first+last, OC name, OC email (derived, editable), Country.
   - Buttons: "Add appointment manually", "Remove", "Split" (if two rows were merged).
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

## 4. Reading the screenshots with a cheap vision model

The screenshots come from a scheduling tool (fields like `SA-1234567`, "TR Project Owner", "Cannot Complete Details",
Scheduled Start/End). The server sends each uploaded photo to **Claude Haiku 4.5** with a fixed prompt and a strict
JSON schema (structured outputs), so the model can only return the fields we define — SA, restaurant, type, outcome,
date, start/end, customer, OC name, OC email, country, plus a per‑field "uncertain" flag. A schedule screenshot with
several rows returns an array. The tech then confirms everything on the review screen; the model never submits anything.

**Where it runs:** on the hosted server, as an API call. Nothing is installed on your machine or the techs' machines.
The API key lives in the server's environment variables; techs never see it.

**Cost** (Anthropic list prices, Sept 2026):

| Model | Input / output per 1M tokens | Approx. cost per screenshot* | When |
|---|---|---|---|
| Claude Haiku 4.5 (`claude-haiku-4-5`) | $1 / $5 | ~$0.005 | Default. Screenshots of rendered text are easy; this is plenty. |
| Claude Sonnet 5 (`claude-sonnet-5`) | $2 / $10 | ~$0.01 | Step up only if Haiku is making mistakes on blurry phone photos. |

\*A phone screenshot is roughly 1,500 image tokens plus a short prompt and a ~300‑token JSON reply.
A tech submitting 10 appointments a day from ~15 screenshots is about **$0.08/day**, or roughly **$1.50–2/month per tech**.
Prepaid API credit; there is no subscription. Screenshots are downscaled with `sharp` before sending, which keeps the
token count (and cost) near the floor.

**Guardrails:** the model reads pixels into JSON and that is all. It never decides categories, never writes notes,
never chooses form values, and never touches the browser — all of that stays deterministic per your guide. If the API
is down or the key runs out of credit, the review screen simply opens empty and the tech types the fields manually.

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
4. **Anthropic API key** for screenshot reading. Create an account at console.anthropic.com, add a small prepaid
   balance ($10–20 will last months at the volumes above), create a key, and set a monthly spend limit in the console
   so a bug can't run up a bill. The key goes into the server's environment variables only.
5. **Database/storage.** Supabase project (free tier) — you already have Supabase connected to this workspace, so
   this is the path of least resistance — or "just use SQLite on the server".

**Configuration values (confirm or change)**

| Setting | Current value from the guide |
|---|---|
| Partner Manager | Lindsey Shea |
| Vendor Company | Sora |
| VC | blank |
| VC Email | Service@sorapartners.com |
| Tech / Tech Email | entered on the landing page, remembered per browser |
| OC email convention | `firstname.lastname@toasttab.com` |
| Time zone label | always `CST` on scheduled times |
| Customer Notified | always Yes |
| Duration | Single Day (multi‑day asks the tech) |
| Screenshot retention | proposed 30 days |

**Optional**

6. A shared passcode for the site (see §9). One word you give to techs; without it the link is fully public.
7. A domain or subdomain (e.g. `forms.sorapartners.com`) if you want a branded link instead of the host's default URL.

---

## 8. Build phases

| Phase | Deliverable | Rough effort |
|---|---|---|
| 0 | Repo scaffold, Docker, CI, Supabase schema, landing page | 1 day |
| 1 | Form mapping + health‑check script; Playwright engine filling all 4 pages up to (not including) Submit, with per‑page screenshots and read‑back verification | 2–3 days (needs form access) |
| 2 | Upload → vision model → JSON → review screen, prompt tuned on your samples | 1–2 days (needs samples + API key) |
| 3 | Interview wizard, ops‑status/wrap‑up computation, conditional‑note enforcement, final review | 2 days |
| 4 | Submit path, confirmation capture, duplicate guard, history page, CSV export | 1 day (needs test‑submission plan) |
| 5 | Pilot with 2–3 techs, tune the extraction prompt | 1 week calendar |

Phases 1 and 2 can run in parallel once items 1 and 2 in §7 are provided.

---

## 9. Risks and open questions

- **Form changes.** Toast can edit the form at any time. Mitigation: nightly health check + "form changed" lockout +
  the mapping stored as versioned data rather than hard‑coded selectors.
- **CAPTCHA / bot detection on Cognito.** Unknown until tested headless. See §5.
- **No login means the link is the only gate.** Anyone who gets the URL can submit Job Progress Logs to Toast under
  Sora's name with any tech name they type. Recommended cheap guard: one shared passcode set as an environment
  variable, asked once per browser and remembered. It costs nothing to build and keeps a forwarded link from being
  usable by outsiders. Your call; the plan works either way.
- **Model misreads** (a wrong digit in an SA, a swapped time) are caught by the review screen, which is mandatory —
  the tech cannot skip from upload to submit.
- **Ambiguous names for OC email derivation** (nicknames, hyphens, two‑word first names) — the app shows the derived
  email and requires the tech to confirm it, never submits a derived email unseen.
- **Multiple techs submitting at once** — fine; the queue serializes browser sessions.
- **PII in screenshots** — retention window + private storage bucket + the optional passcode. Screenshots sent to
  the Anthropic API are covered by its standard commercial terms (not used for training).
- **Duplicate submissions** — the guide's biggest concern. Handled by the submissions table check, the single‑click
  rule, and the "correction" checkbox.
