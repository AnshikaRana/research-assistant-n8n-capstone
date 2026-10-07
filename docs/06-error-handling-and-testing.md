# 6. Error Handling and Testing

## 6.1 Error handling

| Situation | Behaviour |
|---|---|
| Invalid registration input | 400 (01) or `success:false, errors[]` (01.2/01.1). Nothing is written. |
| Duplicate topic (any casing or spacing) | 409 / `duplicate:true`. No new row. |
| arXiv or Semantic Scholar down, timing out, rate-limited or missing a credential | 30 s timeout, 3 retries with back-off, batched requests (arXiv 1 per 3 s, S2 5 per 3 s). After that the node continues with **zero results for that source**, and the other source still works. |
| No papers found | The run completes successfully with zero rows. |
| Same paper from both sources | One record per topic. Missing fields are filled from the other source, so good metadata is never overwritten with null. Priority: arXiv ID → DOI → normalised title + year. arXiv ID comes first because arXiv results rarely carry a DOI while Semantic Scholar exposes the arXiv ID. Similar but different titles are kept as separate papers. |
| Paper already stored on an earlier run | Skipped before the LLM call (`topic_id + dedupe_key`), so there are no duplicates and no wasted tokens. |
| LLM call fails or returns malformed JSON | Code fences are stripped, then the output is parsed and validated. 02: score 0 → filtered, and a `warning` audit event says how many papers failed the AI check. 03: `processing_status = ai_failed` + `processing_error` (set `processed=false` to retry). 05: falls back to a plain paper list instead of the AI overview. |
| PDF link broken | `pdf_status = download_failed` with the HTTP status. The paper stays registered and the run continues. |
| Drive upload fails | `pdf_status = drive_upload_failed`. The run continues. |
| Gmail send fails | `digest_log.status = failed` and papers stay `digested = false`, so the next run retries. |
| Any unhandled node error (including a Data Table write failure) | The workflow's Error Workflow (06) logs `workflow_error` with the failed node, message and execution ID, and emails the admin. |

## 6.2 Scenario tests (6 Oct 2026, before OpenAI was connected)

| # | Scenario | How | Result |
|---|---|---|---|
| T1 | Happy path – registration | POST to 01.2 and 01, and submit the 01.1 form in the browser | ✅ Rows created with `topic_id` and `topic_key`. The form showed "Registration Successful – Topic ID …". Audit rows written via 06. |
| T2 | Invalid input | Empty researcher, bad email, 2-char topic, empty keywords, `frequency=hourly` | ✅ 400 (01) / `success:false` (01.2) with all 5 errors. No row written. |
| T3 | Duplicate topic | Same topic with different case, extra spaces and upper-case email; also across 01 ↔ 01.2 | ✅ 409 / `duplicate:true`. Still one row. |
| T4 | Repeated execution | Same registration sent twice | ✅ Second call → duplicate, no new row. |
| T5 | Paper search (live arXiv) | Run 02 for 3 topics | ✅ 37 real papers (20 + 17). All 37 had PDF links parsed. |
| T6 | No paper results | Topic "Agentic AI workflow automation" (exact-phrase arXiv search) | ✅ 0 results for that topic. The run still succeeded. |
| T7 | API failure | Semantic Scholar with no credential (simulates an outage or auth failure) | ✅ Retried, then treated as 0 results. arXiv results still processed. Execution status: success. |
| T8 | LLM failure | 02, 03 and 05 run with no OpenAI credential | ✅ 02: 37 papers filtered with reason "AI relevance call failed", plus a `warning` audit event. 03: paper marked `ai_failed` with the error. 05: digest built with the plain-list fallback. |
| T9 | Cross-run idempotency (02) | One paper pre-stored, then 02 re-run | ✅ `new_candidates` 37 → 36. The stored paper was skipped before the LLM. |
| T10 | Cross-source duplicates | Unit test of the merge code (sandbox, node.js) | ✅ S2 record (no abstract) + arXiv record merged into one with abstract and PDF filled. Same paper under a second topic kept. Similar titles kept separate. |
| T11 | PDF handling (04) | 3 fixture rows: valid arXiv PDF, broken URL, no link | ✅ Real PDF downloaded → `drive_upload_failed` (no Drive credential yet). Broken URL → `download_failed` (HTTP 404). No link → `no_link`. All 3 archived with APA + BibTeX. |
| T12 | Search API | `?q=membership inference retrieval`, `?paper_id=…`, no params | ✅ Ranked results (match scores 21/8/7). Related-paper mode excluded the source paper. Missing query → 400. |
| T13 | Digest send failure (05) | Run 05 without Gmail | ✅ `digest_log` status `failed`. Papers stayed `digested=false` for retry. `warning` audit event. |
| T14 | Error Workflow (06) | Error Trigger fed a simulated 429 failure payload | ✅ `workflow_error` row with failed node and message. The email step is skipped until Gmail is connected. |

The `TEST-` fixture rows used for T11 and T13 were deleted before the live run.

## 6.3 Live end-to-end run with OpenAI (7 Oct 2026)

OpenAI was connected (gpt-5-mini) and the whole pipeline was run manually on the real data, in order. All numbers come from the execution logs and from `audit_log`.

| # | Step | Execution | Result |
|---|---|---|---|
| L1 | 02 Paper Search & Collection | #86 · 66 s | ✅ 3 topics searched. 37 candidates after de-duplication. **36 saved, 1 filtered** (relevance < 0.70), 0 AI failures. Stored relevance scores ranged from 0.92 to 0.98. "Agentic AI workflow automation" again returned 0 arXiv hits. |
| L2 | 03 AI Summarization (batch 1) | #88 · 50 s | ✅ 20 papers summarised, 0 AI failures |
| L3 | 03 AI Summarization (batch 2) | #90 · 54 s | ✅ 16 papers summarised, 0 AI failures. **36 / 36** papers now have the full 10-field analysis. Categories: NLP 19, Computer Vision 16, Multimodal 1 |
| L4 | 04 Reference Management | #92 | ❌ **Bug found:** with 25 PDFs, Google Drive (no credential) failed as a whole. The error items had no paired-item link, so *Mark Drive Upload Failed* crashed. |
| L5 | 04 after the fix | #93, #95 | ✅ 25 + 11 = **36 papers archived** with APA + BibTeX. Real PDFs downloaded; `pdf_status = drive_upload_failed` (expected: Drive not connected). |
| L6 | 05 Research Digest | #97 · 17 s | ✅ 2 digests built (RAG: 10 papers, ViT for medical imaging: 10 papers), both with an AI overview, trends, a "read first" list and open questions. Gmail not connected, so both went to the failure branch: `digest_log = failed`, papers stay `digested = false` for retry. |
| L7 | 04 Search API | #99 | ✅ `?q=faithfulness attribution` returned 6 ranked results (top match score 11). |

**Fix applied for L4.** A new *Attach Reference* node carries each paper's citation fields together with its downloaded PDF. *Mark PDF Stored* and *Mark Drive Upload Failed* now use the paired item when it exists and the item position when it doesn't. This makes the PDF branch independent of how Google Drive reports a failure. The fix is in the exported `04_Reference_Management_and_Repository.json`.

Sample outputs from this run are in [`samples/`](../samples): a full paper record, the AI digest overview, and a search response.

## 6.4 What is still not live-tested

These parts need credentials that are not connected yet. Their failure branches are tested above (T7, T11, T13, L5, L6):

- Semantic Scholar results (success path). The S2 key goes in a Header Auth credential.
- Google Drive upload (success path).
- Gmail delivery of the digest and of the admin alert (success path).
