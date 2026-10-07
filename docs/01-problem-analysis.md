# 1. Problem Analysis

**Assignment 8: AI Research Assistant Platform**
Domain: Research & Knowledge Management

## 1.1 Business context

Universities and research labs depend on people keeping up with the literature. A student starting a thesis, or a faculty member supervising several projects, has to:

- search several databases (arXiv, Semantic Scholar, publisher sites),
- decide which of the results are actually relevant,
- read and summarise each paper,
- keep references in a consistent citation format,
- remember to check again next week for new work.

Today most of this is done by hand, with browser tabs, PDF folders and copy-pasted citations. The work is repetitive and grows with every new project. Because nothing is shared, the next student repeats it.

The goal of this project is a self-running service. A researcher states their interest once, and the service keeps a curated, summarised and citable collection of papers up to date. It then tells the researcher when something new arrives.

## 1.2 Stakeholders

| Stakeholder | Role | What they need from the system |
|---|---|---|
| **Researchers / students** (primary users) | Register topics and read the digests | Relevant papers only, short trustworthy summaries, ready-made citations, a digest at a frequency they choose |
| **Faculty supervisors** | Guide students' literature reviews | Quick view of what a student's topic currently looks like (digest, search API); consistent references |
| **Lab / department admin** | Runs the n8n instance and pays for the APIs | Credentials kept out of workflow code, an audit trail of every run, alerts when something breaks, controlled LLM cost |
| **Library / knowledge management** | Maintains shared research resources | A searchable repository with stable records (DOI, arXiv ID, BibTeX), plus archived PDFs |
| **External providers** (arXiv, Semantic Scholar, OpenAI, Google) | Supply data and services | Requests that respect their rate limits and terms of use |

## 1.3 Pain points

1. **Search is manual and repeated.** The same queries are re-typed on several sites every week.
2. **There is too much to read.** Keyword search returns many papers that only mention the term, so relevance has to be judged one paper at a time.
3. **Summaries are inconsistent.** Notes are written in different styles and often miss the method, datasets or limitations.
4. **References get messy.** Citations are copied by hand, in mixed formats, with duplicates of the same paper from different sources.
5. **No central repository.** PDFs sit in personal folders and the group cannot search them.
6. **New work gets missed.** No one is notified when a matching paper is published.
7. **Failures are silent.** With scripts or manual routines, nobody notices when a step stops working.

## 1.4 Objectives (from the brief) and how they are met

| Objective | Implemented by | Measurable outcome |
|---|---|---|
| Automate literature collection | **02** Paper Search & Collection: daily arXiv + Semantic Scholar search per active topic, cross-source de-duplication | New candidate papers per topic per day with no manual search. Live run: 37 candidates for 3 topics, de-duplicated |
| Generate AI-powered research summaries | **02** LLM relevance filter (score ≥ 0.70) and **03** structured summary (summary, key findings, methodology, datasets, models, limitations, research gap, significance, keywords, category) | Every stored paper gets the same 10-field summary. Live run: 36 / 36 papers summarised, 0 AI failures |
| Organise references efficiently | **04** APA citation and BibTeX entry per paper, a `.bib` library file, PDF archive to Google Drive | Each archived paper has `citation_apa` and `bibtex` |
| Track new publications | **02** daily schedule + **05** digest at daily / weekly / monthly frequency per topic | The researcher gets only papers they haven't seen yet (`digested = false`) |
| Maintain a research knowledge base | n8n Data Tables (`research_topics`, `research_papers`, `digest_log`, `audit_log`) and **04** search API (`GET /webhook/research/papers/search`) | Ranked keyword search and "related papers" over the stored collection |

## 1.5 Scope and constraints

- **Platform:** n8n Cloud (hosted). `$env` is blocked there, so every secret is stored as an n8n credential.
- **Storage:** n8n Data Tables. These keep the solution self-contained, with no external database to manage.
- **Cost control:** the LLM is called only for *new* papers. Papers already stored are skipped before the relevance call. 03 processes at most 20 papers per run, and failed papers are not retried automatically.
- **Out of scope:** full-text PDF reading by the LLM (titles and abstracts only), user authentication on the public webhooks, a front-end beyond the n8n form.
