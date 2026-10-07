# 4. Data Model (n8n Data Tables)

**research_topics**: `topic_id` (`TOPIC-<ms>-<rand>`), `researcher`, `email` (lower-cased), `topic` (whitespace-normalised), `topic_key`, `keywords` (lower-case, de-duplicated, comma-separated), `frequency` (daily | weekly | monthly), `active` (bool), plus the built-in `createdAt` / `updatedAt`.

- `topic_key = lower(email) | lower(topic)` is the idempotency key. The same researcher can't register the same topic twice, even with different casing or spacing.

**research_papers** has one row per *(topic, paper)*. The same paper can legitimately belong to two different researchers' topics.

| Group | Columns |
|---|---|
| Identity | `paper_id`, `topic_id`, `dedupe_key`, `source` (arxiv / semantic_scholar), `arxiv_id`, `doi`, `url` |
| Metadata | `title`, `authors` (JSON array), `abstract`, `publication_date`, `keywords` (from topic), `researcher` |
| Relevance (02) | `relevance_score`, `relevant` |
| AI processing (03) | `summary`, `key_findings`, `methodology`, `datasets`, `models`, `limitations` (JSON arrays as strings), `research_gap`, `research_significance`, `extracted_keywords`, `research_category`, `processed`, `processing_status` (`pending_ai` → `summarized` / `ai_failed`), `processing_error` |
| References & PDF (04) | `citation_apa`, `bibtex`, `pdf_url`, `pdf_status` (`link_found`, `no_link`, `stored`, `download_failed`, `drive_upload_failed`), `drive_file_url`, `archived` |
| Notification (05) | `digested` |

A known PDF link and a downloaded PDF are tracked as separate states (`pdf_url` vs `pdf_status`).

**audit_log**: `log_id`, `workflow`, `event`, `level`, `message`, `details` (JSON), `execution_id`, `execution_url`.

**digest_log**: `digest_id`, `topic_id`, `topic`, `email`, `paper_count`, `status` (sent / failed).

## State diagram of a paper

```
new candidate --(02 relevance < 0.70)--> filtered (not stored, logged)
      |
      +--(02 relevance >= 0.70)--> pending_ai --(03 ok)--> summarized --(04)--> archived --(05 sent)--> digested
                                       |
                                       +--(03 invalid / failed)--> ai_failed (manual retry: set processed=false)
```

`pdf_status` (set by 04): `link_found` → `stored` | `drive_upload_failed` | `download_failed`, or `no_link`.
