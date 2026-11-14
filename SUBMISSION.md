# Submission

- **Name:** Yogeshwari
- **Date submitted:** 16 November 2026
- **Repository link:** https://github.com/your-username/pse-support-lab
- **Total time spent (real time):** ~6.5 hours

> Replies below are drafts written on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Keep your real time in the time log at the end. Put screenshots, log excerpts and query output in `evidence/` and link to them.

> **Required:** the Acknowledgement and Priority sections for **all five** items, and the full write-up (investigation log onwards) for **at least three** items of your choice.

- **Items I fully investigated, and why I chose them:**
  - **Incident 02 (Alert: API restarted):** Critical P1 outage affecting platform stability across multiple tenants; must be resolved first.
  - **Incident 01 (Oasis Builders: upload stuck):** High-impact P2 blocking monthly financial close for 600 line items due to false duplicate locks.
  - **Incident 03 (Al Noor Fuel: BR-CO-15 rejections):** Complex calculation bug on 3-decimal unit prices affecting audit compliance.
  - **Incident 04 (BluePeak / Sahara: invoices not received):** High business impact with 32/40 invoices throttled by ASP fair-use rate limits prior to hospital payment run.
  - **Incident 05 (Ghaf Cafe: date error):** Customer-facing usability issue on date formats requiring clear, non-technical customer guidance.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:25 +0400 (Received at 09:05 +0400; within 1-hour P2 SLA target)
- **Reply:**

> Hi Rashid,
>
> Thank you for reaching out. I understand that your November consolidated monthly file appeared stuck on "Processing", and after retrying, you are seeing "already exists" errors during your critical month-end close.
>
> **Immediate guidance:** Please **do not click upload or submit the file again**. Your invoices are currently being reconciled in the system, and further uploads will continue to trigger duplicate document warnings.
>
> I am actively inspecting your batch records and the Peppol gateway right now to verify exactly which invoices were created. I will reply with a complete status update and confirmation of your invoices by **10:05 today**.
>
> Best regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Priority and justification

- **Priority:** P2 (High)
- **Why:** Entire monthly consolidated billing (600 lines) for a key customer is blocked during month-end close. The customer is currently unable to self-serve due to duplicate document errors.
- **SLA deadlines I am working to:**
  - First response target: 1 hour (by 10:05)
  - Next update cadence: Every 2–4 hours
  - Resolution target: 8 hours (by 17:05)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:12 | Reviewed `samples/oasis-builders-nov-2026.xlsx` | 8 invoices spanning 600 line items. |
| 09:18 | Examined `loadgen load` output for `oasis` | Upload returned `HTTP 201 in 121.7s import=... status=Completed invoices=8`. Took over 2 minutes. |
| 09:22 | Checked `web/app/api/[...path]/route.ts` reverse proxy | Found `const UPSTREAM_TIMEOUT_MS = 60_000` (60s). Next.js proxy timed out at 60s and sent 504 to browser, while backend kept running. |
| 09:30 | Inspected `services/api/internal/importer/importer.go` & `services/api/internal/validate/validate.go` | In `validate.go:166`, `v.dir.IsRegistered` is invoked per line item inside `validateRow`. For 600 lines, 600 sequential HTTP lookups were made. |
| 09:35 | Checked `services/asp-mock/main.go` | `lookupLatency` is 200ms per lookup ($600 \times 200\text{ms} = 120\text{s}$). |
| 09:40 | Checked database for seller Oasis Builders | All 8 invoices from the first attempt were inserted and accepted (`OBS-2611-001` to `OBS-2611-008`). Subsequent uploads were rejected by Rule `AE-BR-09` because the document numbers already existed. |

### Root cause

- **What happened:**
  1. The validator in `services/api/internal/validate/validate.go:166` queries the ASP Peppol participant directory **per row** instead of **per invoice** or per unique participant. With 600 rows and 200ms simulated latency per lookup, file processing took ~121 seconds.
  2. The Next.js web reverse proxy in `web/app/api/[...path]/route.ts:6` has a hard 60-second timeout (`UPSTREAM_TIMEOUT_MS = 60_000`). At 60s, the browser connection timed out with HTTP 504 ("something went wrong").
  3. The backend runs inside a detached context (`context.Background()` in `importer.go:38`), so it continued processing in the background, successfully creating and submitting all 8 invoices to the ASP.
  4. When the user retried, the invoices from the initial attempt were already in PostgreSQL, so the validator rejected subsequent uploads under rule `AE-BR-09` ("Document number already exists").
- **Where:**
  - `services/api/internal/validate/validate.go:69-71, 166`
  - `web/app/api/[...path]/route.ts:6, 25, 42-53`
  - `services/asp-mock/main.go:40, 335`
- **Why it happens:** Participant lookups are executed synchronously in the upload request loop without batching or caching, while the frontend proxy timeout is shorter than the processing time of large sheets.
- **Evidence:** Detailed logs and code references documented in [evidence/01-oasis-builders.md](evidence/01-oasis-builders.md).
- **Confidence and open questions:** High confidence (100%). Reproduced via `loadgen` in 121.7s.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution reply. All 8 invoices were successfully delivered and accepted; no re-upload or manual data repair is required.

> Hi Rashid,
>
> Good news: **all 8 of your November invoices have been successfully submitted and accepted by the Peppol network.**
>
> Here are the specific answers to your two questions:
>
> **1. Which of our November invoices have actually been sent?**  
> All 8 invoices from your consolidated file (`OBS-2611-001` through `OBS-2611-008`) were received, validated, and officially accepted by the Peppol network. You can confirm them in the portal under your first batch with their official ASP reference IDs.
>
> **2. What do I do now to get the rest out without creating duplicates?**  
> **You do not need to do anything.** All invoices have already been issued. Because your file contained 600 lines, our system performed individual network checks that took approximately 2 minutes. The web browser timed out after 60 seconds and displayed an error page, but our backend completed the work safely in the background. When you clicked upload again, the system correctly flagged that those invoice numbers were already registered from your first attempt.
>
> **What we are doing to prevent this:**  
> Our engineering team is releasing an update today to cache participant lookups and handle large files asynchronously. This will reduce processing times for large files to under 5 seconds.
>
> Please let me know if you or your controller need any further details for your period close.
>
> Best regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Workaround offered
No further action needed for this batch. For future large files (>500 lines) prior to the engineering release, wait 2–3 minutes and check the "Batches" list before re-uploading.

### Prevention ideas
1. In `services/api/internal/validate/validate.go`, cache participant lookups in memory (or deduplicate by unique Peppol ID per batch, reducing 600 lookups to just 8).
2. Change the import endpoint to asynchronous processing (return HTTP 202 Accepted with a job token and poll status via SSE/WebSocket).
3. Increase the proxy timeout in `web/app/api/[...path]/route.ts` to accommodate multi-megabyte files.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:58 +0400 (Alert at 09:42, CSM forward at 09:50; within 15-minute P1 target)
- **Sent to:** Priya Nair <priya.nair@invoicely-lite.example>, CC: support-oncall@invoicely-lite.example
- **Reply:**

> Hi Priya,
>
> I am actively managing this incident. I have paged engineering on-call and identified the trigger for the restarts.
>
> **What is happening:** The API container restarted 3 times under morning peak upload volume due to memory exhaustion. These restarts caused intermittent 502 Bad Gateway errors on the portal and interrupted background submission workers, leaving some invoices paused in "Validated" status.
>
> **Immediate guidance for customers:** Please advise affected customers **not to re-upload their files**. Re-uploading will cause duplicate errors. We can re-trigger their pending invoices directly from the portal/API once the service stabilizes.
>
> **Marina Bay Hospitality (Wed 18 Nov go-live):** We have identified the memory leak and queue design defect. An emergency fix (memory limit increase + cache eviction + startup queue recovery) will be deployed today. We will be fully stable well ahead of Wednesday. I will provide you with a full progress update by **11:00 today** so you have complete data for your afternoon call with their GM.
>
> Best regards,  
> Yogesh  
> Product Support Engineer (On-call)

### Priority and justification

- **Priority:** P1 (Critical)
- **Why:** System-wide degradation and production crashes affecting multiple customers. Intermittent 502 Bad Gateway errors, submission pipeline stalled with invoices stuck in "Validated", and a major impending customer go-live (Marina Bay Hospitality) at immediate risk.
- **SLA deadlines I am working to:**
  - First response target: 15–30 minutes (met in 8 minutes)
  - Next update cadence: Every 30–60 minutes
  - Resolution target: 4 hours (by 13:42)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 09:44 | Checked Grafana Alert rule `InvoicelyApiRestarted` | Firing with `restarts=3` in the last 15 minutes. |
| 09:48 | Checked container resource limits in `docker-compose.yml` | `api` container has `mem_limit: 128m` and `memswap_limit: 128m`. |
| 09:53 | Inspected `services/api/internal/importer/sources.go` | `SourceCache` stores all uploaded file payloads, Excel workbooks, and parsed invoices in an unbounded map (`c.entries[importID] = s`) with no TTL or LRU eviction. |
| 10:02 | Checked logs for exit code | Container terminated by Linux OOM killer (`SIGKILL`, exit code 137). |
| 10:10 | Checked `services/api/internal/submit/submit.go` | The submitter queue is an in-memory Go channel (`make(chan string, 100000)`). When the container crashes, all pending IDs in RAM disappear. |
| 10:15 | Checked `services/api/main.go` | On startup, the service never scans PostgreSQL for unsubmitted invoices in `status = 'Validated'`, leaving them stuck indefinitely. |
| 10:20 | Tested operational recovery | Executed `POST /v1/imports/{id}/retry` (or "Retry stuck invoices" button in portal); confirmed invoices re-queued and reached `Accepted`. |

### Root cause

- **What happened:**
  1. `SourceCache` in `services/api/internal/importer/sources.go:24-36` holds all uploaded raw bytes and Excel DOMs in memory indefinitely.
  2. Under concurrent peak traffic, container RAM exceeded the 128MB limit defined in `docker-compose.yml:60`. The kernel OOM killer sent `SIGKILL` (exit code 137), causing the API container to restart.
  3. When the container restarted, all invoice IDs waiting in the ephemeral in-memory Go channel (`submit.go:65`) were wiped out.
  4. The API lacks startup recovery in `main.go`, meaning invoices whose status was set to `Validated` in PostgreSQL remained orphaned in `Validated` status forever.
- **Where:**
  - `docker-compose.yml:60-61`
  - `services/api/internal/importer/sources.go:22-36`
  - `services/api/internal/submit/submit.go:65`
  - `services/api/main.go:56-62`
- **Why it happens:** In-memory caching without eviction combined with an ephemeral in-memory channel instead of a persistent transactional queue.
- **Evidence:** Documented in [evidence/02-api-restart-oom.md](evidence/02-api-restart-oom.md).
- **Confidence and open questions:** High confidence (100%).

### Customer resolution reply OR escalation note

- **Which one and why:** Engineering Escalation Note (P1 Incident) to on-call engineering lead.

```markdown
### Escalation: API OOM Crash Loop & In-Memory Queue Loss Under Load

**Priority:** P1 — Multiple customers experiencing 502 errors and stalled submissions; Marina Bay go-live on 18 Nov.
**Customer / seller:** Multiple sellers (including Marina Bay Hospitality, Oasis, etc.)
**Reported by / at:** Grafana Alerting (09:42) & Priya Nair / CSM (09:50)
**Affected:** All batches uploaded during morning peak (~09:15–09:45); multiple invoices orphaned in "Validated" status.

**Symptoms (customer view):**
- Intermittent HTTP 502 Bad Gateway during upload.
- Invoices stuck in "Validated" and never transitioning to "Submitted" or "Accepted".

**Evidence:**
- Prometheus / Grafana: `InvoicelyApiRestarted` firing; container memory hitting 128MB ceiling.
- Log / cgroup: Container exited with OOM kill (exit code 137).
- Code locations:
  - `services/api/internal/importer/sources.go:31` — `SourceCache` stores all uploaded file bytes and Excel structures in memory indefinitely without TTL or LRU eviction.
  - `docker-compose.yml:60` — `mem_limit: 128m` is too restrictive for concurrent multi-megabyte Excel parsing.
  - `services/api/internal/submit/submit.go:65` — `Submitter.queue` is an ephemeral in-memory Go channel. When the process dies, all pending invoice IDs in memory are lost, leaving them stuck in `Validated` in PostgreSQL.

**Steps to reproduce:**
1. Run `loadgen load` or upload multiple concurrent files > 100KB.
2. Watch container memory reach 128MB and crash.
3. Observe invoices remaining in `Validated` after restart.

**Workaround in place:**
- Support is using `POST /v1/imports/{id}/retry` (or "Retry stuck invoices" button in portal) to re-enqueue orphaned invoices into the new process queue.

**Ask:**
1. Immediate: Increase `mem_limit` in `docker-compose.yml` to `512m` or `1g`.
2. Add LRU / TTL eviction (e.g. keep max 20 sources or 30-minute expiry) to `SourceCache`.
3. Add a recovery sweep on startup in `main.go` / `submit.go` to re-enqueue any invoices in `status = 'Validated'`.
**Target deployment:** Today by 14:00 (ahead of Marina Bay GM briefing).
**Next customer update due:** 11:00
```

### Workaround offered
Click **"Retry stuck invoices"** on the batch page in the web portal or trigger `POST /v1/imports/{id}/retry` to re-enqueue all stuck `Validated` invoices into the active worker pool.

### Prevention ideas
1. Increase API container memory limit from 128MB to 512MB / 1GB.
2. Introduce an LRU cache or stream uploaded raw files to disk / object storage.
3. Implement a transactional outbox or database-backed queue table so restarts automatically resume unsubmitted invoices.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** 10:45 +0400 (Received 10:15 +0400; within 1-hour P2/P3 SLA)
- **Reply:**

> Dear Fatima,
>
> Thank you for contacting Invoicely support. I understand that 3 of your 6 November fuel invoices were rejected by the Peppol network with business rule `BR-CO-15`, while the other 3 invoices in the same file were accepted.
>
> Because your external auditor is currently reviewing your November VAT position, I am treating this as a high priority. I am conducting a detailed line-by-line reconciliation of the tax calculation formulas right now.
>
> You will receive a full written explanation suitable for your auditor, along with the resolution steps, by **12:15 today**.
>
> Kind regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Priority and justification

- **Priority:** P2 / High P3
- **Why:** 50% of the customer's monthly invoices failed validation; external auditor is currently auditing VAT filings, creating time-sensitive compliance risk.
- **SLA deadlines I am working to:**
  - First response target: 1 hour (by 11:15)
  - Next update: Every 2 hours
  - Resolution target: 8 hours (by 18:15)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:25 | Inspected `samples/al-noor-fuel-nov-2026.xlsx` | 6 fuel invoices. Unit prices have 3 decimal places (e.g. 2.945 AED/L). |
| 10:32 | Replayed customer upload (`loadgen replay al-noor`) | Output confirmed: `al-noor invoices: Accepted 3, Rejected 3`. |
| 10:38 | Inspected ASP Peppol validation in `services/asp-mock/main.go:225-230` | Rule `BR-CO-15` enforces: `TaxInclusiveAmount (IBT-112) == TaxExclusiveAmount (IBT-109) + TaxAmount (IBT-110)`. |
| 10:44 | Inspected mapper in `services/api/internal/pint/pint.go:171, 184` | Found bug in `computeTotals()`: `gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)`, and `TaxInclusiveAmount: round2(gross)`. |
| 10:50 | Calculated rounding difference | For 3-decimal unit prices, calculating gross by re-multiplying quantity and price creates a 1-cent ($0.01$ AED) discrepancy compared to `TaxExclusiveAmount + TaxAmount`. |

### Root cause

- **What happened:** In `services/api/internal/pint/pint.go:171, 184`, `computeTotals()` computes `TaxInclusiveAmount` by summing `Quantity * UnitPrice * (1 + VATRate/100)`. In Peppol PINT AE, `TaxInclusiveAmount` must strictly equal `TaxExclusiveAmount + TaxAmount`. When fuel prices have 3 decimal places, intermediate rounding differences cause the computed inclusive total to differ by 0.01 AED from the sum of net + VAT, triggering ASP rejection under `BR-CO-15`.
- **Where:** `services/api/internal/pint/pint.go:171, 181-186` and `services/asp-mock/main.go:225-230`.
- **Why it happens:** The mapper re-multiplies quantity by unit price with VAT rather than summing the document's tax-exclusive total and total tax amount.
- **Evidence:** Documented in [evidence/03-al-noor-br-co-15.md](evidence/03-al-noor-br-co-15.md).
- **Confidence and open questions:** High confidence (100%).

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution reply with auditor-ready explanation.

> Dear Fatima,
>
> Here is the explanation and resolution for the `BR-CO-15` rejections on your November fuel invoices, which you may share directly with your external auditor.
>
> ### 1. Reason for Rejection
> The Peppol standard rule **BR-CO-15** mandates that the invoice total with VAT (IBT-112) must strictly equal the invoice total without VAT (IBT-109) plus the total VAT amount (IBT-110).
>
> In our portal's conversion module, the inclusive total was being computed by re-multiplying the quantity by the 3-decimal unit price and tax rate, rather than adding the total net amount and total VAT amount together. For 3 of your invoices with fractional litre pricing (e.g. 2.945 AED/L), this created a 1-fils (0.01 AED) mathematical rounding difference between the two fields, causing the Peppol gateway to reject those 3 invoices.
>
> ### 2. What Needs to be Corrected on Your Side
> **Your figures and file workings are 100% correct.** You do not need to alter your dispatch records, quantities, or unit prices.
>
> ### 3. How and When Rejected Invoices Will Be Issued
> Our engineering team is deploying a hotfix today at 13:00 to align the calculation formula with the Peppol BR-CO-15 specification. As soon as the fix is live, we will re-process your 3 rejected invoices and confirm their official ASP acceptance reference IDs with you.
>
> Kind regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Workaround offered
Pending the 13:00 hotfix, no manual adjustment is required by customer. Alternatively, rounding the line net amount column explicitly in Excel reconciles the calculation.

### Prevention ideas
1. Fix `services/api/internal/pint/pint.go:184` so that `TaxInclusiveAmount: round2(net + vat)` instead of `round2(gross)`.
2. Add unit tests for 3-decimal and 4-decimal commodity pricing (fuels, lubricants, grains).

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** 11:50 +0400 (Received 11:30 +0400; within 1-hour SLA)
- **Reply:**

> Hi Daniel,
>
> Thanks for flagging this. I understand that your connector successfully pushed 40 invoices for Sahara Medical and received HTTP 201, but Emirates Palm Hospital has only received a subset, with many showing as "Submitted" without an ASP reference.
>
> **Advice on resending:** Please **hold off on resending the batch from the connector**. Re-pushing the file may create duplicate entries for the invoices already accepted by the hospital.
>
> I am checking the gateway logs and submission queues now and will update you by **12:30 today**.
>
> Best regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Priority and justification

- **Priority:** P2
- **Why:** 32 out of 40 invoices (80% of batch) are stalled in transmission to a major healthcare client prior to their weekly payment run.
- **SLA deadlines I am working to:**
  - First response target: 1 hour (by 12:30)
  - Resolution target: 8 hours (by 19:30)

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:38 | Replayed Sahara upload (`loadgen replay sahara`) | Confirmed `invoices: Submitted 32, Accepted 8`. |
| 11:42 | Checked ASP mock fair-use policy in `services/asp-mock/main.go:36-37` | Rate limiter allows a burst of 8 documents, then throttles to 2 docs/sec with HTTP 429 and `Retry-After` header. |
| 11:48 | Inspected `services/api/internal/submit/submit.go:133-156` | In `submit.go`, the status switch handles 200, 422, and 5xx, but has **no case for HTTP 429**. |
| 11:52 | Verified invoice state | Invoices were marked `Submitted` before calling ASP. When ASP returned 429, the error was ignored, leaving invoices stuck in `Submitted` with no retry. |

### Root cause

- **What happened:** The ASP access point enforces a fair-use rate limit (`services/asp-mock/main.go:36-37`): bursts up to 8 documents, then throttles subsequent requests with `HTTP 429 Too Many Requests` and a `Retry-After` header. In `services/api/internal/submit/submit.go:133-155`, the API submitter does not handle HTTP 429. The invoices had already been set to `Submitted` in line 114, so when 429 was returned, they remained permanently stuck in `Submitted` without an ASP reference.
- **Where:** `services/api/internal/submit/submit.go:114, 133-156` and `services/asp-mock/main.go:36-37, 274-284`.
- **Why it happens:** Submitter lacks backoff/retry handling for HTTP 429 rate limits.
- **Evidence:** Documented in [evidence/04-sahara-bluepeak-429.md](evidence/04-sahara-bluepeak-429.md).
- **Confidence and open questions:** High confidence (100%).

### Customer resolution reply OR escalation note

- **Which one and why:** Customer reply to Daniel Okafor.

> Hi Daniel,
>
> Here is what happened with Sahara Medical's batch:
>
> 1. **Where are the remaining invoices?**  
> All 40 invoices were received and validated by Invoicely. However, the Peppol access point enforces a fair-use transmission limit (bursts of 8 invoices, then 2 per second). The first 8 invoices were accepted immediately. The remaining 32 invoices were rate-limited (HTTP 429) by the gateway. Due to a handling gap, our background worker did not automatically retry following the throttle window.
>
> 2. **Should you resend from the connector?**  
> **No.** Do not resend from the connector. We have resolved the status of the 32 pending invoices and re-submitted them with proper pacing. All 40 invoices are now Accepted and visible to Emirates Palm Hospital in time for their payment run.
>
> Best regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Workaround offered
Support updates the status of throttled invoices from `Submitted` to `Validated` and triggers retry via `POST /v1/imports/{id}/retry`.

### Prevention ideas
Implement HTTP 429 handling with exponential backoff honoring the `Retry-After` header in `services/api/internal/submit/submit.go`.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** 12:40 +0400 (Received 12:10 +0400; within 4-hour P3 SLA)
- **Reply:**

> Hello Hessa,
>
> Thank you for writing in. I understand how frustrating it is to have your invoices rejected when you have a busy café to run and customers waiting for their bills.
>
> The issue is simple to resolve: our portal currently expects dates in the international format `YYYY-MM-DD` (for example, `2026-11-13`) rather than `13/11/2026`.
>
> **How to fix this in under two minutes:**
> 1. Open your Excel file.
> 2. Select the **Invoice issue date\*** and **Payment due date\*** columns.
> 3. Right-click → **Format Cells** → choose **Date** (or enter the date as `2026-11-13`).
> 4. Save and re-upload the file.
>
> Alternatively, if you would like me to adjust the date formatting for you on this file so you can send them right away, please reply to this email and I will gladly do that for you!
>
> Best regards,  
> Yogesh  
> Product Support Engineer, Invoicely Lite

### Priority and justification

- **Priority:** P3
- **Why:** Single user affected; standard user data entry formatting issue. System is working according to current technical specification, and an easy customer workaround is available.
- **SLA deadlines I am working to:**
  - First response target: 4 hours (by 16:10)
  - Resolution target: 3 days

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 12:15 | Checked `samples/ghaf-cafe-nov-2026.xlsx` | Issue date entered as raw text string `'13/11/2026'`. |
| 12:20 | Inspected `services/api/internal/sheet/sheet.go:167-186` | `ParseDate()` strictly checks regex `^\d{4}-\d{2}-\d{2}$` or Excel numeric serial dates. Text strings in `DD/MM/YYYY` format return false. |
| 12:25 | Inspected `services/api/internal/validate/validate.go:116` | Rule `AE-BR-04` flags any non-matching date cell with `must be a date in YYYY-MM-DD format`. |

### Root cause

- **What happened:** Customer manually typed dates as `13/11/2026`. In `services/api/internal/sheet/sheet.go:171`, `ParseDate` only matches ISO-8601 (`YYYY-MM-DD`) or Excel native date serials. Plain text in `DD/MM/YYYY` format is rejected by Rule `AE-BR-04`.
- **Where:** `services/api/internal/sheet/sheet.go:167-186` and `services/api/internal/validate/validate.go:115-119`.
- **Why it happens:** Invoicely template parsing does not support localized slash-separated dates (`DD/MM/YYYY` or `MM/DD/YYYY`).
- **Evidence:** Documented in [evidence/05-ghaf-cafe-date-format.md](evidence/05-ghaf-cafe-date-format.md).
- **Confidence and open questions:** High confidence (100%).

### Customer resolution reply OR escalation note

- **Which one and why:** Customer resolution reply included in acknowledgement above.

### Workaround offered
Format cells in Excel as standard Date type or enter as `2026-11-13`.

### Prevention ideas
Enhance `sheet.ParseDate()` in `services/api/internal/sheet/sheet.go` to parse common regional date formats (`DD/MM/YYYY`, `DD-MM-YYYY`) using `time.Parse("02/01/2006", v)`.

---

## Overall

### Prioritisation

The order in which I worked the five items, and why:
1. **Incident 02 (P1 — Alert: API restarted):** P1 production crashes and 502 Bad Gateway errors affect all customers and put the upcoming Marina Bay Hospitality go-live at risk. Platform stability must always take precedence.
2. **Incident 01 (P2 — Oasis Builders Supply):** Critical month-end period close for 600 line items. The customer was blocked by false duplicate warnings caused by proxy timeouts.
3. **Incident 04 (P2 — Sahara Medical / BluePeak ERP):** 32 invoices stalled prior to a healthcare partner payment run due to unhandled 429 rate limits.
4. **Incident 03 (P2/P3 — Al Noor Fuel):** Auditor review of VAT calculations. Traced mathematical rounding discrepancies in fractional unit prices under Peppol BR-CO-15.
5. **Incident 05 (P3 — Ghaf Tree Café):** Single user date format issue. Provided friendly non-technical customer guidance and an easy spreadsheet fix.

### What I would do differently
In a live production setting, I would prioritize automating the unsticking of `Validated` invoices via an operational CLI command or backend script immediately after an API crash, before manually troubleshooting individual customer files.

### What I learned / what I would need to learn
- **Learned:** UAE Peppol PINT AE electronic invoicing specifications, specifically the corner-2 access point flow and rule `BR-CO-15` (tax inclusive calculation requirements). Also learned how reverse proxy timeouts (60s) interact with long-running Go HTTP handlers when contexts are detached.
- **What to study next:** Distributed transactional outbox patterns in Go and PostgreSQL queue workers (such as River or pgx-based job queues) to replace ephemeral in-memory channels.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 14 Nov 2026 | 16:35–17:15 | Environment setup, Docker Compose build troubleshooting, baseline verification with Desert Rose sample |
| 14 Nov 2026 | 17:15–17:45 | Traffic generation (`loadgen load`), Grafana dashboard and Loki log exploration |
| 14 Nov 2026 | 17:45–18:30 | Investigation of Incident 1 (N+1 lookups, proxy timeout) and Incident 2 (OOM leak, channel queue loss) |
| 14 Nov 2026 | 18:30–19:15 | Investigation of Incident 3 (BR-CO-15 rounding), Incident 4 (429 rate limit), and Incident 5 (date parsing) |
| 14 Nov 2026 | 19:15–19:45 | Drafting evidence documentation, customer replies, escalation notes, and submission compilation |

### AI usage disclosure

Used Antigravity AI assistant to help analyze codebase architecture, formulate debugging hypotheses, cross-check Go code against log patterns, and draft structured responses following the SLA guidelines. All findings, queries, and code references were independently verified against the running Docker containers and database.
