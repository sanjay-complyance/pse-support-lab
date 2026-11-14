# Submission

- **Name:**
- **Date submitted:**
- **Repository link:**
- **Total time spent (real time):**

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Keep your real time in the time log at the end. Put screenshots, log excerpts and query output in `evidence/` and link to them.

> **Required:** the Acknowledgement and Priority sections for **all five** items, and the full write-up (investigation log onwards) for **at least three** items of your choice. For an item you did not fully investigate, replace the remaining sections with "Not investigated — what I would check next: …".

- **Items I fully investigated, and why I chose them:**

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:25 (email received 09:05; P2 first-response target is 1 h, so due by 10:05)
- **Reply:**

> Subject: RE: URGENT - monthly invoice upload stuck on "Processing"
>
> Hello Rashid,
>
> Thank you for the detail. I understand your November file stayed on "Processing", that retrying led to "already exists" messages, and that you need to know which invoices have gone out so you can close the month. I know this is holding up your month-end, and I'm treating it as urgent.
>
> I'm checking exactly what was received for each of your invoice numbers now. **Please don't upload the file again for now**, so we avoid creating duplicates.
>
> I'll send you a list of what has been sent and what hasn't, and the safe next step, by **11:00 today**. If a call would help, tell me a good time.
>
> Regards,
> Nitheesh, Product Support, Invoicely Lite

### Priority and justification

- **Priority:** P2 (raised to P1 if the investigation shows other customers cannot upload either)
- **Why:** Invoices from this seller can still reach the ASP, so it is not P1 on the first test in `docs/sla-matrix.md`. But a whole month-end file is affected, with a close deadline and customers waiting, so I treated it as the top of P2. Any file that takes longer than about 60 s to process hits the same problem, so I will re-check the priority if more customers report it.
- **SLA deadlines I am working to:** first response 10:05 (sent 09:25) / next update 11:00 / resolution target 8 business hours, so 17:05 the same day.

### Investigation log

> Times are mapped onto the exercise clock (Monday 16 Nov 2026). The work itself was done in real time; see the time log at the end for real times.

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:30 | Read the email and the attached `oasis-builders-nov-2026.xlsx` (8 invoices, 600 lines) | File layout matches the template, so this is not a bad file |
| 09:40 | Read `importer.go`, `validate.go`, `directory.go` | Import runs inside one HTTP request. Validation makes one Peppol lookup per row, in sequence (`validate.go:165`) |
| 09:55 | Timed the ASP lookup with curl | About 0.2 s each (0.203 s, 0.203 s, 0.224 s) |
| 10:05 | Replayed the Oasis upload on a clean system | HTTP 201 after **120.7 s**, all 8 invoices Accepted, API restarts 0 ([replay output](evidence/01-oasis-replay-output.txt)) |
| 10:20 | Hypothesis: the API crashes on large files (link to the restart alert) | **Ruled out.** `restarts=0`, `oomkilled=false` during the replay |
| 10:30 | Reset the database and uploaded the file three times, 30 s apart, directly to the API | All returned 201 after about 121 s. Upload 1 = 1 Accepted / 7 Rejected, upload 2 = 8 Rejected, upload 3 = 7 Accepted / 1 Rejected. Each invoice number was Accepted exactly once, and the ASP received 8 documents ([per invoice](evidence/01-oasis-retry-per-invoice.txt), [ASP count](evidence/01-oasis-asp-received.txt)) |
| 10:45 | Read `route.ts` in the web app | The portal proxy stops waiting after 60 s (`UPSTREAM_TIMEOUT_MS`, line 7) |
| 10:50 | Uploaded through the portal (port 3000) | **HTTP 504 "Gateway Timeout" after exactly 60 s**; the API kept working in the background ([proxy test](evidence/01-oasis-via-portal-proxy.txt)) |
| 10:55 | Checked how the duplicate rule treats in-progress invoices | `store.go:300` ignores only Rejected invoices, and `importer.go:243` saves invoices as Parsed before validating them. The mid-run screenshot shows "Parsed 7, Validated 1" ([screenshot](evidence/01-oasis-portal-processing.png)) |

### Root cause

- **What happened:** A normal-sized monthly file takes about 2 minutes to process, but the portal stops waiting after 60 s and shows an error page. The import carries on in the API. Rashid retried, and each retry's invoices collided with invoices from the earlier uploads that were still being processed, producing "already exists" messages.
- **Where:**
  - `services/api/internal/validate/validate.go:165-166`: one Peppol lookup per row, no cache
  - `services/api/internal/importer/importer.go:193`: whole import runs inside the upload request
  - `web/app/api/[...path]/route.ts:7`: 60 s proxy timeout
  - `services/api/internal/store/store.go:300-302` together with `importer.go:243`: invoices still in progress count as duplicates
- **Why it happens:** The work grows with the number of rows (600 rows × 0.2 s = 120 s), the timeout in front of it is fixed at 60 s, and the portal gives no sign that the first upload is still running.
- **Evidence:** [replay output](evidence/01-oasis-replay-output.txt), [retry run](evidence/01-oasis-retry-repro.txt), [per-invoice outcomes](evidence/01-oasis-retry-per-invoice.txt), [ASP count](evidence/01-oasis-asp-received.txt), [portal proxy test](evidence/01-oasis-via-portal-proxy.txt), screenshots [processing](evidence/01-oasis-portal-processing.png), [completed](evidence/01-oasis-portal-completed-120s.png) and [Grafana](evidence/01-grafana-overview-oasis-replay.png)
- **Confidence and open questions:** High on the timing, the 504 and the duplicate mechanism, because I reproduced each one. Medium on Rashid's exact outcome, because his click timing was different from mine. I have not tested a file split into smaller parts yet.

### Customer resolution reply OR escalation note

- **Which one and why:** Both. The customer gets an update now, because he asked what was sent. I am also escalating to L3 because this is a product defect, not a data error I can fix myself.

> **Update to Rashid (sent 11:00):**
>
> Hello Rashid,
>
> Here is what I found.
>
> **What happened:** Your November file has 600 lines, and the portal needs about two minutes to check it. After one minute the portal stops waiting and shows an error, even though your file keeps being processed behind the scenes. Each time you uploaded again, the new copy ran into the earlier copy, which is why you saw "already exists". You did nothing wrong.
>
> **1. Which invoices have been sent?** [Before sending, check the ASP records and the portal for each invoice number and paste the list here: invoice number, status, ASP reference. "Accepted" means our delivery partner has received it and it is on its way to your customer. Only say that nothing was sent twice if the ASP records show it.]
>
> **2. What to do now:** Please don't upload the whole file again. The "already exists" messages on invoices that also show Accepted need no action. If any invoice has no Accepted copy, tell me and I'll confirm the safe way to send just those.
>
> I've passed the cause to our engineering team with all the details. I'll update you again by **14:00**, whatever the news. If a call would help, I'm available.
>
> Regards,
> Nitheesh

> **Escalation note (L3):**
>
> ### Escalation: Month-end uploads over about 60 s return 504 and cause duplicate-number rejections
>
> **Priority:** P2 — a seller's month-end file cannot be uploaded reliably; close deadline today.
> **Customer / seller:** Oasis Builders Supply LLC, seller id 2
> **Reported by / at:** Rashid Al Mansoori, email, 09:05 Mon 16 Nov
> **Affected:** every upload that takes longer than 60 s. Oasis file `oasis-builders-nov-2026.xlsx`: 8 invoices, 600 rows.
>
> **Symptoms (customer view):** upload sits on "Processing", then an error page; retries produce AE-BR-09 "already exists" with status Parsed or Validated.
>
> **Evidence:**
> - Web log: `POST /imports status 504 durationMs 59999 TimeoutError`
> - Import duration 120.7 s for 600 rows; participant lookup 0.2 s each (`validate.go:165`)
> - Three overlapping uploads left 8 invoices Accepted once each and 16 Rejected with AE-BR-09 (`evidence/01-oasis-retry-per-invoice.txt`)
> - ASP received 8 documents for sender 0235:1003182746
>
> **Steps to reproduce:**
> 1. Reset the system and upload `samples/oasis-builders-nov-2026.xlsx` for Oasis through http://localhost:3000.
> 2. After 60 s the portal returns 504. The import completes at about 120 s.
> 3. Upload again before it completes: AE-BR-09 against Parsed invoices.
>
> **What I have already tried / ruled out:** API crash or out-of-memory (restarts 0, no OOM kill).
>
> **Suspected cause (with confidence):** High. `route.ts:7` timeout against `validate.go:165` per-row lookup, plus `store.go:300` counting in-progress invoices as duplicates.
>
> **Workaround in place:** yes. The customer is told not to re-upload, and large files can be split into smaller parts (to be verified).
>
> **Timeline:**
> - 09:05 — email received
> - 09:25 — acknowledgement sent
> - 10:50 — 504 at 60 s reproduced through the portal
> - 11:00 — update sent to customer
>
> **Ask:** (1) cache or batch participant lookups per file, (2) make the import asynchronous or raise the timeout, (3) make the duplicate check ignore invoices from imports still being processed, or show a clear message. Please reply by 13:00.
> **Next customer update due:** 14:00

### Workaround offered

- Do not re-upload the whole file. Wait until the batch list shows no batch in "Processing".
- Send only invoices that have no Accepted copy. If an invoice is Validated or Failed, use its **Retry submission**. A Rejected invoice does not use up its number, so a corrected re-upload of just those invoices is allowed once nothing else is in flight.
- For future months, split the file into smaller uploads so each finishes inside 60 s. At 0.2 s per row that is roughly 250 rows or fewer. This is an estimate and still needs testing.

### Prevention ideas

- Cache the Peppol lookup per buyer within one import. That would take this file from 120 s to a few seconds.
- Process imports asynchronously and show real progress.
- Make the proxy timeout longer than the processing budget, or return immediately with an import id.
- Make the duplicate message say that another upload of this file is still running.
- Add a dashboard panel and an alert for import duration.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Sent to:**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):**
- **Reply:**

> …

### Priority and justification

- **Priority:**
- **Why:**
- **SLA deadlines I am working to:**

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| | | |

### Root cause

- **What happened:**
- **Where** (component, `file:line`):
- **Why it happens:**
- **Evidence:**
- **Confidence and open questions:**

### Customer resolution reply OR escalation note

- **Which one and why:**

> …

### Workaround offered

### Prevention ideas

---

## Overall

### Prioritisation

The order in which you worked the five items, and why. Did you change your mind as you learned more?

### What I would do differently

### What I learned / what I would need to learn

Be specific: concepts, tools or parts of the system that were new to you, how you got up to speed, and what you would study next.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| | | |

### AI usage disclosure

Which tools you used (or "none"), what for, and how you checked their output against the system.
