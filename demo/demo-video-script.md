# Demo Video Script: AI Research Assistant Platform (n8n)

**Target length:** about 8 minutes (the brief asks for 5–10) · **Speaker:** Anshika Rana · **Recording:** screen + voice (OBS, Loom or the Windows Game Bar)

Every number in this script comes from the real run on 7 Oct 2026. Your new demo topic will give its own numbers; read them off the screen.

---

## Before you record (10 minutes)

1. Log in to **anshika-rana.app.n8n.cloud**. Open these browser tabs in this order:
   1. Workflows list
   2. **Data Tables** → `research_papers`
   3. Form: `https://anshika-rana.app.n8n.cloud/form/f5e40afe-8291-4382-a6f7-b98bb0aae967`
   4. Workflow **02 - Paper Search & Collection**
   5. Workflow **03 - AI Summarization & Keyword Extraction**
   6. Workflow **04 - Reference Management & Repository**
   7. Workflow **05 - Weekly Research Digest & Notifications**
   8. Workflow **06 - Error Handler & Audit Log**
   9. Data Table `audit_log`
2. *(Optional, makes the demo stronger.)* Connect **Gmail** (OAuth2) in 05 *Send Digest Email*. The digest then really arrives in your inbox. Without it, the script uses the "failure branch" version of scene 6.
3. Pick a **new** demo topic, so that scene 3 finds fresh papers. Suggested:
   - Topic: `Graph Neural Networks for drug discovery`
   - Keywords: `graph neural networks, molecular property prediction, drug discovery`
   - Frequency: `weekly`
4. Close notifications. Zoom the browser to 110–125 % so node names are readable.
5. Have the architecture slide (slide 4 of the deck) ready to show in scene 1.

---

## Scene 1 · Introduction and architecture (0:00 – 1:00)

**Show:** the title slide, then the architecture slide.

> "Hi, I'm Anshika Rana. This is my n8n capstone for the IIT Jammu AI/LLM Summer School: an **AI Research Assistant Platform**, assignment 8.
>
> Researchers lose hours every week searching arXiv and Semantic Scholar, reading abstracts, writing summaries and formatting citations. My platform automates that whole loop. You register a topic once. From then on it collects papers every day, uses an LLM to keep only the relevant ones, writes a structured summary of each paper, builds APA and BibTeX references, keeps a searchable repository, and emails you a research digest.
>
> It's built as **eight n8n workflows with 114 nodes**. They don't call each other directly. They hand work over through **status fields in n8n Data Tables**, so each stage can run and fail on its own. Workflow 06 audits every run and every failure."

**Point at:** the five pipeline boxes from left to right, the Data Tables row, and 06 on the right.

## Scene 2 · Registering a topic (1:00 – 2:00)

**Show:** the form tab.

1. Fill in the form with your demo topic, then **Submit**. The page shows *"Registration Successful – Topic ID TOPIC-…"*.
2. Press back and submit the **same topic again**, but type it in a different case (`graph neural networks FOR drug discovery`). The page now shows *"Duplicate Entry"*.
3. Optional: submit with a bad email (`abc@`) and show *"Registration Failed"*, which lists the errors.

> "Topic registration has a web form and a REST API. Input is validated: email format, topic length, keywords and frequency. Duplicates are detected without caring about case or extra spaces, because I store a normalised key made of email plus topic. So the same researcher can't register the same topic twice, and invalid input never reaches the database."

**Show:** the `research_topics` table with the new row (`topic_id`, `keywords` lower-cased, `active = true`).

## Scene 3 · Paper search and AI relevance filter (2:00 – 3:30)

**Show:** workflow 02. Click **Execute workflow**. It takes about a minute, so talk over it.

> "Workflow 02 runs every morning at eight. For every active topic it builds a query and searches **arXiv and Semantic Scholar in parallel**, with rate limiting, timeouts and three retries. If one source is down, the other still works.
>
> Results from both sources are normalised and de-duplicated by arXiv ID, then DOI, then title. Duplicates are merged, so missing fields get filled in. Then the important step: papers that are already stored are skipped **before** the LLM, so we never pay twice for the same paper.
>
> Each new paper then goes to **GPT-5-mini** for a relevance score between 0 and 1. Only papers scoring at least 0.70 are stored."

**When it finishes:**

- Click **Skip Already Stored Papers**. Most candidates from the three older topics are skipped; this shows idempotency.
- Click **Parse Relevance Result** and open one item to show `relevance_score` and `reason`.
- Click **Build Run Summary** and read the counts. On 7 Oct, the first full run gave *37 candidates, 36 saved, 1 filtered*.

## Scene 4 · AI summarisation (3:30 – 4:30)

**Show:** workflow 03. Click **Execute workflow** (it processes up to 20 papers per run).

> "Workflow 03 turns every new abstract into a structured research note. The prompt asks for strict JSON: summary, key findings, methodology, datasets, models, limitations, research gap, significance, keywords and a category. It also says: *don't invent anything that isn't in the abstract*. I treat LLM output as untrusted. A Code node parses and validates it, and invalid output is marked `ai_failed` instead of breaking the run."

**Show:** the `research_papers` table and open one row. For example, FAIR-RAG from the RAG topic has `summary`, `key_findings` (F1 0.453 on HotpotQA), `datasets` (HotpotQA, 2WikiMultiHopQA, MusiQue) and `research_gap`.

> "Across the live run, all 36 papers were summarised with zero AI failures."

## Scene 5 · References, PDF archive and search (4:30 – 5:45)

**Show:** workflow 04, then click **Execute workflow**.

> "Workflow 04 is reference management. Citations are built **in code, not by the LLM**, so they can't be hallucinated: APA 7 and BibTeX from the stored metadata. Then it downloads each PDF and uploads it to Google Drive, and writes a `.bib` library file for the run."

**Point at:**

- **Build Citations** output: one APA string and one BibTeX entry.
- The PDF branch, with its four outcomes:
  - stored
  - Drive upload failed
  - download failed
  - no link

> "Every outcome is recorded in `pdf_status`, so a broken link or a Drive problem never stops the run. Testing this with 25 PDFs at once also exposed a real bug: when Drive failed for the whole batch, the items lost their link back to the paper. I fixed it with the *Attach Reference* node, which carries each paper's data along with its PDF."

**Search API:** in 04, click the **Search Request** node → **Listen for test event**. In a new tab, open:
`https://anshika-rana.app.n8n.cloud/webhook-test/research/papers/search?q=faithfulness attribution`

> "The repository is searchable through this API. Results are ranked by where the terms match: the title counts three times, keywords twice, then summary and abstract. With `paper_id` instead of `q`, it recommends related papers."

## Scene 6 · Research digest (5:45 – 6:45)

**Show:** workflow 05, then click **Execute workflow**. A manual run includes every active topic.

> "Workflow 05 sends each researcher a digest at the frequency they chose: daily, weekly on Mondays, or monthly on the 1st. The LLM writes an overview across the new papers, with emerging trends, which papers to read first, and open questions. Each paper then gets a card with its summary, findings, PDF link and citation."

**Show:** the **Write Digest Overview** output for the RAG topic. On 7 Oct it said that RAG is moving "toward engineering-first, trust-oriented systems that prioritize faithfulness, attribution, and domain adaptivity", and it recommended reading FAIR-RAG, the RAG-stack review and the TREC 2025 RAG track overview first.

- **If Gmail is connected:** switch to your inbox and open the email *"Research digest: … – N new paper(s)"*.
- **If not:** open **Build Digest Email**, then the HTML view. Point at **Log Digest Failed**:

  > "Gmail isn't connected on this instance yet, so the send fails safely. It's logged as failed, and the papers stay undigested, so the next run retries automatically."

## Scene 7 · Error handling and audit log (6:45 – 7:30)

**Show:** workflow 06, then the `audit_log` table.

> "Workflow 06 is the observability layer. Every workflow sends a run summary here, and every workflow has 06 set as its **Error Workflow**, so any unhandled failure in a scheduled or production run is caught too. Each event becomes a row in `audit_log`, and errors also send an email alert to the admin once Gmail is connected."

**Point at the rows from the live run:**

- `36 relevant new paper(s) saved, 1 filtered out`
- `20 paper(s) summarized, 0 AI failure(s)`
- `25 reference(s) archived` (a warning, because Drive is not connected)
- `0 digest(s) sent, 2 failed`

> "Every failure path was tested: invalid input, duplicates, no search results, an API outage, the LLM failing, broken PDF links, a failed email and a 429 error. Fourteen scenario tests in total, plus this live run. They're all documented in the repository."

## Scene 8 · Wrap-up (7:30 – 8:00)

**Show:** the GitHub README, then the final slide.

> "To sum up: eight workflows, 114 nodes, five integrations. AI is used where judgement is needed (relevance, summaries and the digest), and deterministic code is used where accuracy matters (citations, de-duplication and search). Every run is audited. Next steps are connecting Gmail, Drive and Semantic Scholar on this instance, and adding a full-text PDF summary. Thank you!"

---

## Backup plan if a live run is slow or fails

- All the runs from 7 Oct are saved. Open each workflow's **Executions** tab and show these instead:
  - 02: #86
  - 03: #88, #90
  - 04: #93, #95 (and #92 for the bug)
  - 05: #97
  - search: #99
- If 02 finds no papers for the new topic, say so: *"No new papers today. The run still succeeds, which is test T6."* Then continue with the existing data.
- Record scene by scene and join the clips. Trim the waiting time while workflows run.
