# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|---|---|---|---|---|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial PDD created from docs/jactiv-779-request-details.docx. |

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (UiPath Cartographer, v1.0, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-779 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-779 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Automate the daily identification and notification of Coupa invoices that lack a properly linked purchase order, replacing manual AP filtering with a consistent weekday run that sends one Slack summary to the AP responsible.

**Owning department:** Accounts Payable

| Role | Name / Contact |
|------|---------------|
| SME / Process Owner | Irina Capatina (irina.capatina@uipath.com, Slack member ID WLX9BD8FN) |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|-----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Accounts Payable — invoice exception monitoring |
| Short description | Queries Coupa daily for invoices (status draft/new, past 7 days, no linked PO, excluding credit notes) and sends one Slack Block Kit message with the count and a filtered Coupa link to the SME |
| Required roles | Unattended robot; SME receives notification |
| Trigger and schedule | Weekday schedule at 10:00 Romania time (EET/EEST) |
| Volume (items per day / peak) | ~194 qualifying invoices cited in sample run; typical daily volume [SME REVIEW] |
| Average handling time | Manual: ~daily ad-hoc review; Automated target: <2 min per run [DEFAULT] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — deterministic rules on structured data; credit-note and description-only-PO exclusions are the known cases |
| Input data | Coupa invoice list: invoice date, status, PO linkage field, invoice type |
| Output data | Slack Block Kit direct message to SME with invoice count, policy note, action request, and filtered Coupa URL; no message on clean day |

## 4. To-Be Process (High Level)

The automation runs unattended at 10:00 Romania time on weekdays. It reads the Coupa invoice list for the past seven days, filters to status draft or new, excludes credit notes, and identifies invoices where no purchase order is properly linked in the Coupa PO linkage field. It counts the qualifying invoices and, if the count is greater than zero, sends one Slack Block Kit direct message to the SME containing the count, a policy note, an action request, and a URL pointing to the same filtered Coupa view.

**Steps replaced by automation:**
- Manual navigation and filtering of Coupa invoice list
- Manual identification of invoices with missing PO linkage
- Manual copying of invoice fields and grouping by requester
- Manual composition and sending of Slack message

**What stays human:** The SME reviews the Slack notification and takes action to ensure purchase orders are raised and linked. Requester follow-up is out of scope.

**Boundaries:** The automation reads only. It does not create or modify purchase orders, approve invoices, alter any Coupa record, or track requester completion.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Weekday schedule triggers run at 10:00 Romania time | Scheduler | Process starts | EET (UTC+2) / EEST (UTC+3) depending on DST. Only weekdays. |
| 1.2 | Calculate date window: window_end = today's date; window_start = today minus 7 days | Automation | window_start and window_end values set | Both dates used to build Coupa query and Slack message footer |
| 2.1 | Query Coupa invoice list with filter: invoice_date >= window_start AND invoice_date <= window_end AND status IN (draft, new) | Coupa | Paginated invoice records returned | Access method [SME REVIEW] — API or web UI. BR-04 |
| 2.2 | For each invoice record: check invoice type — if type = credit note, exclude record | Automation | Credit notes removed from working set | BR-03; log exclusion reason per record |
| 2.3 | For each remaining record: check PO linkage field — if PO is properly linked (not null, not description-text only), exclude record | Automation | Only invoices without a properly linked PO remain | BR-01, BR-02; a PO number in the description field does not satisfy the check |
| 2.4 | Count remaining records → invoice_count | Automation | Integer count of qualifying invoices | BR-05 |
| 3.1 | If invoice_count = 0: end run without sending a message | Automation | Process exits cleanly; no Slack message sent | BR-07. **OQ-01** — section 7.1 of source states a congratulatory message should be sent on a clean day; BR-007 says send nothing. Awaiting SME decision; default is send nothing. |
| 3.2 | If invoice_count > 0: build Coupa filtered URL using window_start and window_end | Automation | coupa_url constructed: `https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D=<window_start>&q%5Binvoice_date_lteq%5D=<window_end>&q%5Bstatus_eq%5D=draft` | URL template from source; confirm hostname for production [SME REVIEW] |
| 3.3 | Build Slack Block Kit JSON payload with placeholders: invoice_count, coupa_url, window_start, window_end, run_date | Automation | Complete Block Kit payload ready | Title: `:receipt: {{invoice_count}} invoices need a purchase order`. Body: three icon-led lines. Button: "Open the list in Coupa" (primary). Footer: dates and run_date. BR-06 |
| 4.1 | Send Slack direct message to SME (member ID WLX9BD8FN) via Slack HTTP Request | Slack | Message delivered to SME DM | Recipient identified by Slack member ID, not email. BR-06 |
| 4.2 | Confirm HTTP 200 response from Slack API | Automation | Delivery confirmed; process ends | If non-200, classify as system error S2 |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|-|
| Coupa | API or Web [SME REVIEW] | [SME REVIEW] | [SME REVIEW] | UiPath Orchestrator credential asset [DEFAULT] | Source URL: uipath-test.coupahost.com; confirm production hostname [SME REVIEW] |
| Slack | API (HTTP) | Slack Web API — HTTP Request activity with Bot token | OAuth Bot token | UiPath Orchestrator credential asset [DEFAULT] | Recipient addressed by member ID WLX9BD8FN; Block Kit payload; protocol is HTTPS POST to api.slack.com/api/chat.postMessage |
| Weekday Scheduler | Orchestrator trigger | UiPath Orchestrator time trigger | N/A | N/A | 10:00 Romania time (EET/EEST); weekdays only |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | Apply the no-PO-no-pay policy: invoices without a properly linked purchase order must be reported | BR-001 | 2.3 |
| BR-02 | A PO number typed into the invoice description but not properly linked in the PO linkage field does not satisfy the PO requirement | BR-002 | 2.3 |
| BR-03 | Exclude credit notes from the qualifying population | BR-003 | 2.2 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days | BR-004 | 2.1 |
| BR-05 | Count the qualifying invoices; report that count only — do not list individual invoices | BR-005 | 2.4 |
| BR-06 | Send the count to the SME (Irina Capatina, Slack member ID WLX9BD8FN) by Slack direct message, with a policy note, an action request, and a filtered Coupa link | BR-006 | 3.3, 4.1 |
| BR-07 | Send nothing when a successful query returns no qualifying invoices | BR-007 | 3.1 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is reported as a failed run | BR-008 | All |
| BR-09 | A failed run produces no notification; no second message path exists | BR-009 | All |
| BR-10 | The automation does not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | All |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-----------------|--------|
| B1 | Credit note encountered | 2.2 | Invoice type = credit note | Exclude record; log exclusion with reason; continue processing remaining records |
| B2 | Description-only PO | 2.3 | PO text present in description field but PO linkage field is null or not properly linked | Treat as missing PO; include in qualifying count if other rules pass |
| B3 | No qualifying invoices (clean day) | 3.1 | invoice_count = 0 after all filtering | Send no Slack message; exit cleanly — see OQ-01 for conflicting source instruction |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-----------------|----------|-------------|--------|
| S1 | Coupa unavailable | Coupa query returns connection error or timeout at step 2.1 | High | No retry per BR-08 | Log error; mark run failed; no Slack notification sent per BR-09 |
| S2 | Slack delivery failure | Slack API returns non-200 at step 4.2 | High | No retry per BR-08 | Log error; mark run failed |
| S3 | Application unresponsive | Target application does not respond within timeout | High | No retry [DEFAULT] | Log error; mark run failed |
| S4 | Element not found | Expected UI/API element absent during Coupa interaction | Medium | No retry [DEFAULT] | Log error; mark run failed |
| S5 | Credential expiry | Authentication rejected by Coupa or Slack | High | No retry [DEFAULT] | Log error; alert Orchestrator; mark run failed |
| S6 | Unhandled exception | Any uncaught exception in the process | High | No retry [DEFAULT] | Log full stack trace; mark run failed; no notification |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Clean-day behaviour conflict.** [SME REVIEW] BR-007 says send nothing on a clean day; section 7.1 says send a congratulatory Slack message — SME must decide which applies.
2. **OQ-02 - Coupa access method.** [SME REVIEW] Source does not specify API vs. web UI; SME to confirm Coupa REST API availability and credentials.
3. **OQ-03 - Coupa production hostname.** [SME REVIEW] Source URL is uipath-test.coupahost.com; production hostname must be confirmed before go-live.
4. **OQ-04 - Coupa PO linkage field name.** [SME REVIEW] The exact API field or UI element name for PO linkage on the invoice must be confirmed.
5. **OQ-05 - Coupa pagination.** [DEFAULT] Automation will handle paginated Coupa responses; page size and max-page defaults to be confirmed against Coupa API spec.
6. **OQ-06 - Slack Bot token scope.** [DEFAULT] Bot token requires `chat:write` scope for DM to WLX9BD8FN; Orchestrator asset name to be agreed with developer.
7. **OQ-07 - Romania timezone DST handling.** [DEFAULT] Orchestrator trigger will be set to EET/EEST (Europe/Bucharest); DST transitions handled by Orchestrator.
8. **OQ-08 - Coupa URL status filter.** Source URL template includes `status_eq=draft` only; SME to confirm whether `status_eq=new` also needs to appear in the link or whether the count covers both statuses.
9. **OQ-09 - Block Kit JSON location.** Source states the exact Block Kit JSON is in "architectural considerations, section 4" — that document was not present in the source file; developer must obtain it from the SME.
10. **Unattended robot assumed.** Process is fully automated with no human-in-the-loop steps; an unattended Orchestrator robot licence is required.

## 11. Success Criteria

1. A weekday test run triggers at 10:00 Romania time and completes without manual intervention.
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated; credit notes are excluded.
3. An invoice with a PO number in the description field only is counted as missing a linked PO.
4. A Slack Block Kit direct message is delivered to Slack member ID WLX9BD8FN containing the correct invoice count and a working filtered Coupa URL.
5. When the qualifying count is zero, no Slack message is sent (pending OQ-01 resolution).
6. A run that cannot complete is recorded as failed in Orchestrator and no Slack message is sent.
7. No Coupa record is created or modified during any run.
