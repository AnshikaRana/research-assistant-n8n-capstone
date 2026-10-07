# 5. Setup and Running

## 5.1 Setup

**n8n version:** n8n Cloud (2026). Node versions used: Data Table 1.1, Webhook 2.1, Code 2, IF 2.3, Switch 3.2, Merge 3.2, HTTP Request 4.2/4.5, Execute Workflow 1.2, Basic LLM Chain 1.9, OpenAI Chat Model 1.3, Gmail 2.1, Google Drive 3, Form 2.5, Form Trigger 2.6.

**Environment variables:** none. n8n Cloud blocks `$env`, so every secret lives in n8n credentials. No API key appears in any workflow JSON.

**Credentials to create** (Settings → Credentials). Then open each node that is marked with a red warning and select the credential:

| Credential | Type | Used by |
|---|---|---|
| OpenAI | OpenAI API | 02 *OpenAI Chat Model*, 03, 05 |
| Semantic Scholar | **Header Auth**, name `x-api-key`, value = your S2 key | 02 *Search Semantic Scholar* |
| Gmail | Gmail OAuth2 (Sign in with Google) | 05 *Send Digest Email*, 06 *Email Alert to Admin* (re-enable this node after connecting) |
| Google Drive | Google Drive OAuth2 | 04 *Upload PDF to Drive*, *Upload Library to Drive* |

**Status on the author's instance (7 Oct 2026):**

- OpenAI is connected and live-tested.
- Semantic Scholar, Gmail and Google Drive are **not connected yet**. Their nodes fail cleanly into their error branches (see `06-testing.md`).

Before importing into your own instance, change the admin address in 06 *Email Alert to Admin*.

**Data Tables:** `research_topics`, `research_papers`, `audit_log`, `digest_log`, with the columns listed in §4.

**Import steps** (for a fresh instance):

1. Create the four Data Tables.
2. Import `06` first and note its ID.
3. Import 01.2 → 01 → 01.1 → 02 → 03 → 04 → 05.
4. In each workflow, re-select the Data Table in every Data Table node, and select workflow 06 in every *Log to Audit Trail* node and in *Settings → Error Workflow*.
5. In 01.1, set the HTTP URL to `https://<your-instance>/webhook/research-topic-processor`.
6. Add the credentials, then publish: 06 → 01.2 → 01 → 01.1 → 02 → 03 → 04 → 05. n8n requires referenced sub-workflows to be published first.

## 5.2 Running the complete pipeline

1. Register a topic, either through the **01.1 form URL** or by API:

   ```bash
   curl -X POST https://anshika-rana.app.n8n.cloud/webhook/research/topic \
     -H 'Content-Type: application/json' \
     -d '{"researcher":"Anshika Rana","email":"you@example.com","topic":"Retrieval Augmented Generation","keywords":"RAG, retrieval, LLM","frequency":"weekly"}'
   ```

2. Run **02** (or wait for 08:00). This collects relevant papers.
3. Run **03** with *Manual Run (testing)* (or wait an hour). This summarises them.
4. Run **04** manually (or wait for 10:00). This creates references, archives PDFs to Drive and uploads the `.bib` file.
5. Run **05** manually. A manual run sends a digest for *every* active topic. Scheduled runs follow each topic's frequency.
6. Query the repository:
   - `GET /webhook/research/papers/search?q=membership+inference`
   - `GET /webhook/research/papers/search?paper_id=<id>` returns related papers.
7. Inspect `audit_log` to see what ran, what it found and what failed.
