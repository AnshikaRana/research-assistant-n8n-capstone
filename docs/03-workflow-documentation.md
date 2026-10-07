# 3. Workflow Documentation

There are 8 workflow files and **114 functional nodes**; sticky notes are not counted. The brief asks for 4–6 workflows and 20–25+ nodes. The five sub-workflows suggested in the brief map to 01–05, and 06 adds the "Advanced features" (error handling, retries, logging and audit trail).

| # | Workflow | Nodes | Trigger(s) | Brief's sub-workflow |
|---|---|---|---|---|
| 01 | Research Topic Registration (API) | 12 | Webhook `POST /webhook/research/topic` | 1. Research Topic Registration |
| 01.1 | Research Topic Registration – Form | 6 | n8n Form | 1. (user-facing form) |
| 01.2 | Research Topic Registration – Processor | 13 | Webhook `POST /webhook/research-topic-processor` | 1. (form back-end) |
| 02 | Paper Search & Collection | 24 | Schedule, daily 08:00 | 2. Research Paper Search & Collection |
| 03 | AI Summarization & Keyword Extraction | 11 | Schedule, hourly · Manual | 3. AI Summarization & Keyword Extraction |
| 04 | Reference Management & Repository | 23 | Schedule, daily 10:00 · Manual · Webhook `GET /webhook/research/papers/search` | 4. Reference Management & Repository |
| 05 | Weekly Research Digest & Notifications | 17 | Schedule, daily 09:00 · Manual | 5. Weekly Research Digest & Notifications |
| 06 | Error Handler & Audit Log | 8 | Error Trigger · Execute Workflow Trigger | Advanced features |

Advanced features from the brief, and where they are implemented:

| Feature | Where |
|---|---|
| AI-powered decision making | 02 relevance score (keep ≥ 0.70), 03 structured analysis, 05 digest overview |
| Error handling and retry logic | Retries with back-off on every HTTP, LLM, Gmail and Drive node · `continueErrorOutput` branches in 04 and 05 · Error Workflow = 06 on 01–05 |
| Logging and audit trail | 06 writes every run summary and every failure to `audit_log` |
| Scheduled workflows (cron) | 02 08:00, 03 hourly, 04 10:00, 05 09:00 |
| Webhook-triggered workflows | 01, 01.2, 04 search API |
| Conditional branching and loops | IF / Switch routing in every workflow · per-topic and per-paper item loops with batching (arXiv 1 request / 3 s, Semantic Scholar 5 / 3 s, PDF 1 / 2 s) |
| Human approval step | Not automated. The researcher is the reviewer: the digest links each paper with its AI summary, and topics can be switched off with `active = false` |

---

## 01 · Research Topic Registration (API)

**Purpose:** a public REST endpoint that registers a research interest.

**Input** (JSON body):

```json
{ "researcher": "Anshika Rana", "email": "you@example.com",
  "topic": "Retrieval Augmented Generation",
  "keywords": "RAG, retrieval, LLM", "frequency": "weekly" }
```

`keywords` may be a comma-separated string or an array. `frequency` is `daily`, `weekly` or `monthly`.

**Flow:**

1. *Webhook* receives the request, and *Validate Input* checks every field. It collects **all** errors, not just the first.
2. *Valid Input?* sends invalid requests to **400** `{success:false, errors[]}`.
3. *Generate Topic Record* creates the following, then *Check Duplicate topic* looks up `topic_key`:
   - `topic_id = TOPIC-<ms>-<rand>`
   - lower-cased email
   - whitespace-normalised topic
   - de-duplicated lower-case keywords
   - `topic_key = email|topic` (case-insensitive)
4. If the topic already exists, the response is **409** `{duplicate:true}`.
5. Otherwise *Insert Row* writes to `research_topics` (`active = true`) and returns **200** `{success:true, topic_id}`.
6. *Build Audit Event* calls *Log to Audit Trail* (06), fire-and-forget.

## 01.1 · Registration Form, and 01.2 · Processor

- **01.1** is the n8n web form (name, email, topic, keywords, frequency dropdown). It POSTs the submission to 01.2 (with 3 retries) and a *Switch* shows one of three completion pages: **Registration Successful – Topic ID …**, **Duplicate Entry**, or **Registration Failed** with the list of errors.
- **01.2** uses the same logic as 01, but always answers HTTP 200 with `{success, duplicate, message, topic_id?, errors?}`. This lets the form branch on the body instead of failing on a 4xx.

## 02 · Paper Search & Collection

**Purpose:** find new, relevant papers for every active topic.

**Flow:**

1. **Daily 08:00** → *Get row(s)* reads active topics → *Build Search Query* turns the topic and keywords into a query.
2. The query is searched in two places in parallel:
   - *Search arXiv*: Atom API, 20 results per topic, 1 request every 3 s, 30 s timeout, 3 retries.
   - *Search Semantic Scholar*: Header-Auth credential, 5 requests every 3 s, 3 retries, with `openAccessPdf` among the fields.
   - Both use `continueRegularOutput`, so if one source fails, the other still works.
3. *Parse arXiv / Semantic Scholar Results* normalise both formats: title, authors, abstract, dates, DOI, arXiv ID and PDF link.
4. *Generate Dedupe Key* → *Remove Duplicates* → *Merge* → *Remove Cross-Source Duplicates*:
   - Papers are matched by arXiv ID, then DOI, then normalised title + year.
   - Records are **merged**, so a field missing in one source is filled from the other.
5. *Normalize Paper Metadata*, then *Skip Already Stored Papers* drops `(topic_id, dedupe_key)` pairs that already exist. This happens **before** the LLM call, so no tokens are spent twice.
6. *Basic LLM Chain* + *OpenAI Chat Model* (gpt-5-mini) returns `{relevance_score 0–1, reason}`. *Parse Relevance Result* strips code fences and validates the JSON, and on failure sets score 0 with the reason.
7. *Is Relevant?* (score ≥ **0.70**):
   - **yes** → *Insert row* into `research_papers` with `processing_status = pending_ai`, `processed = false`
   - **no** → *Log Filtered Paper*
8. *Collect Results* → *Build Run Summary* → 06. The summary counts topics, candidates, new candidates, saved, filtered and AI failures.

## 03 · AI Summarization & Keyword Extraction

**Purpose:** turn each stored abstract into a structured research note.

**Flow:**

1. **Every hour** (or manually), *Get Unprocessed Papers* reads at most 20 rows with `processed = false`.
2. *Summarize Paper* (LLM chain, gpt-5-mini, batches of 5) is asked for strict JSON with these fields, using only the title and abstract and without inventing datasets, models or numbers:
   - `summary`, `key_findings[]`, `methodology`
   - `datasets[]`, `models[]`, `limitations[]`
   - `research_gap`, `research_significance`
   - `extracted_keywords[]` (5–8, lower-case), `research_category`
3. *Validate AI Output* parses and type-checks the JSON, then *AI Output Valid?* routes it:
   - **valid** → *Save Summary*: fields saved, `processing_status = summarized`, `processed = true`
   - **invalid** → *Mark AI Failure*: `processing_status = ai_failed` + `processing_error`, so the paper is not retried in a loop
4. *Build Run Summary* → 06.

## 04 · Reference Management & Repository

**Purpose:** citations, PDF archive and a searchable knowledge base. It has two independent entry points.

**A. Archive run** (daily 10:00 or manual):

1. *Get Papers to Archive* reads up to 25 rows that are summarised and not yet archived. *Build Citations* creates an **APA** citation, a **BibTeX** entry (key `firstauthorYEARfirstword`) and a safe PDF file name.
2. *Has PDF Link?*:
   - **no** → *Mark No PDF Link*
   - **yes** → *Download PDF* (1 per 2 s, 60 s timeout, retries) → *Attach Reference* keeps the paper's fields with the file → *Upload PDF to Drive*
3. Each upload outcome is recorded:
   - *Mark PDF Stored*: `pdf_status = stored` + Drive link
   - *Mark Drive Upload Failed*: `pdf_status = drive_upload_failed`
   - A failed download goes to *Mark PDF Unavailable*: `pdf_status = download_failed` with the HTTP error
4. *Collect PDF Outcomes* → *Save Reference Record* sets `archived = true` and saves `citation_apa`, `bibtex`, `pdf_status` and `drive_file_url`.
5. *Build BibTeX Library* → *Convert to .bib File* → *Upload Library to Drive* (`research_library_<yyyy-MM-dd_HHmm>.bib`) → *Build Run Summary* → 06.

**B. Search API** (`GET /webhook/research/papers/search`):

| Query | Result |
|---|---|
| `?q=membership inference` | Papers ranked by weighted term matches: title ×3, extracted keywords ×2, summary ×1, abstract ×1. Ties are broken by the relevance score |
| `?topic_id=TOPIC-…` | Papers of one topic |
| `?paper_id=…` | **Related papers**: uses the source paper's extracted keywords and category as the query, and excludes the source paper. Unknown id → 404 |
| `?limit=n` | Result size (default 10, maximum 50) |
| no parameters | HTTP 400 with usage help |

## 05 · Weekly Research Digest & Notifications

**Purpose:** a personalised email per topic, sent at the frequency the researcher chose.

**Flow:**

1. **Daily 09:00**: *Get Active Topics* → *Select Topics Due Today* picks topics that are due:
   - daily topics every day
   - weekly topics on Mondays
   - monthly topics on the 1st
   - a manual run includes every topic
2. *Get Undigested Papers* (summarised, `digested = false`) → *Group Papers by Topic*.
3. *Write Digest Overview* (LLM) returns `overview`, `trends`, `recommended_first` and `open_questions` across the topic's papers. If the call fails, the digest opens with a plain "N new papers" line instead.
4. *Build Digest Email*: an HTML email with the AI overview, emerging trends, "read first" and open questions. Then one card per paper with title (linked), authors, date, category, relevance, summary, key findings, a PDF or Drive link, and the APA citation.
5. *Send Digest Email* (Gmail, `continueErrorOutput`):
   - **sent** → *Log Digest Sent* (`digest_log.status = sent`) → *Expand Paper IDs* → *Mark Papers Digested*
   - **failed** → *Log Digest Failed*; the papers stay undigested, so the next run retries
6. *Collect Outcomes* → *Build Run Summary* → 06.

## 06 · Error Handler & Audit Log

**Purpose:** central observability for 01–05.

**Two entry points:**

- **Error Trigger.** This is the *Error Workflow* setting of 01–05. *Format Error Event* builds `{workflow, event: workflow_error, level: error, failed node, message, execution_id, execution_url}`.
- **When Called by Another Workflow.** The run events sent by every other workflow.

**Flow:**

1. *Normalize Log Event* coerces an unknown level to `info` and serialises the details.
2. *Write Audit Log* adds a row to `audit_log`.
3. *Is Error?* decides whether to alert:
   - `level = error` → *Email Alert to Admin* (Gmail; disabled until the Gmail credential is connected)
   - otherwise → *No Alert Needed*

**Guarantee:** callers use `waitForSubWorkflow = false`, so a logging failure can never break the calling pipeline.
