---
name: enterprise-doc-analysis
description: >
  Use this skill whenever you have one or more retrieved enterprise documents to analyze in order
  to answer a user query, partially or fully. Triggers include: queries where the answer may live
  in contracts, emails, meeting notes, legal filings, correspondence, reports, or any free-text
  organizational content. Also use when documents were returned alongside tables and must be
  reconciled with tabular findings. Use even when only one document is available — this skill
  governs the full analysis workflow from ingestion to structured evidence output.
  Pair with enterprise-table-analysis when both tables and documents are present.
---

# Enterprise Document Analysis Skill

This skill governs how to analyze free-text enterprise documents — individually and jointly — to
answer user queries. Documents arrive pre-retrieved by semantic search and wrapped in Pydantic
models. Your job is **not** to re-rank or re-filter — it is to read carefully, extract evidence,
reconcile across documents, and produce structured output for a downstream synthesis model.

The LLM executing this skill is a mid-tier model. **Follow every step explicitly. Do not skip
steps. Do not try to do multiple steps at once.**

---

## Step 0: Read What You Have

Before doing anything else, enumerate your inputs.

For each document, extract and write down:
- `doc.name` or `doc.id`
- `doc.description` (treat as a weak hint, not ground truth — the description may have been
  auto-generated or may be stale)
- `doc.metadata` fields if present: date, author, source system, document type label, associated
  entity IDs (customer ID, case number, lawyer ID, etc.)
- Content length (approximate: short < 300 words, medium 300–1500, long > 1500)
- A 2–3 sentence summary of what the content appears to be about, in your own words

Also note:
- What tables (if any) were retrieved alongside the documents
- The user's query, restated in your own words (one sentence)

**Do not proceed until you have written this inventory.**

---

## Step 1: Understand Each Document's True Nature

The `description` and `document_type` metadata fields are unreliable. Infer the true nature of
each document from the content itself:

1. **Opening and closing lines.** Legal documents typically open with parties and dates; emails
   open with headers; meeting notes open with attendees and agenda.
2. **Named entities in the content.** People, organizations, dates, monetary amounts, case
   numbers — these tell you who the document is about and when it was created.
3. **Tone and register.** Formal legal language vs. casual internal memo vs. structured report.
4. **Structural signals.** Numbered clauses → contract or legal filing. Bullet points → notes or
   summary. Continuous prose → narrative report or email.
5. **Entity IDs in content vs. metadata.** If the document body mentions an ID, name, or case
   number that also appears in metadata — or in a retrieved table — that is a confirmed link.

For each document, write one sentence: *"This document appears to be [type] concerning [subject],
dated approximately [date if known], involving [key entities]."*

---

## Step 2: Classify Documents by Type

Label each document as one of:

- **Reference** — describes a standing state: contract, agreement, policy, profile
- **Event record** — captures a discrete event: meeting notes, filed complaint, incident report
- **Correspondence** — communication between parties: email, letter, memo
- **Financial** — monetary transactions or obligations: invoice, expense report, billing summary
- **Legal** — formal legal instruments: lawsuit, filing, court order, legal opinion
- **Uncertain** — type cannot be determined yet; revisit after Step 4

---

## Step 3: Extract Entities and Anchors

These are your **anchors** — they connect documents to each other and to any tables.

### 3a. Call the Internal Entity Extractor

For each document, call the internal NER API on the document's content:

```python
import requests

def extract_entities(text: str) -> dict:
    response = requests.post(
        "http://localhost:8080/extract_identifiers",
        json={"text": text},
        timeout=10
    )
    response.raise_for_status()
    return response.json()

entities = extract_entities(doc.content)
```

The API returns structured identifiers — expect fields such as:
`NationalID`, `AccountID`, `Phone`, `Name`, and any other ontology types configured
in your system. These are the same ontology types used in table column tags, which
means API results are your primary bridge between document content and table columns.

**Use the API results as authoritative.** If the API identifies a string as a
`NationalID`, treat it as one even if it looks like a generic number in the text.

### 3b. Supplement with Manual Extraction

The entity extractor covers typed identifiers well but may miss:
- Dates and time references (explicit dates, "last Tuesday", durations)
- Monetary amounts (prices, fees, totals)
- Locations (addresses, offices, venues)
- Key claims or assertions (what the document states happened or is true)
- Organization names and roles not in the ontology

For these, read the document and extract manually. Add them to the entity list from 3a.

### 3c. Handle API Failures Gracefully

If the API call fails (connection error, timeout, non-200 response):
- Log the failure: `"entity_extraction_api": "unavailable"`
- Fall back to manual extraction for all anchor types
- Lower your confidence rating by one level for any finding that would have
  benefited from typed identifier resolution
- Do **not** halt the analysis

### 3d. Write the Anchor List

After 3a and 3b, write a flat list per document:

```
Doc "invoice_2024_03":
  - NationalID: 123456789  [API]
  - AccountID: ACC-4421    [API]
  - Name: "דוד לוי"        [API]
  - Date: 2024-03-15       [manual]
  - Amount: ₪4,200         [manual]
  - Claim: payment for legal services rendered in February  [manual]
```

Tag each item with `[API]` or `[manual]` so downstream steps know how it was identified.
Do not interpret yet — just extract and label.

---

## Step 4: Plan the Analysis

Look at the user's query. For each document (or group of documents), decide what you need to
extract or determine. Write the plan before executing it.

Common analysis tasks:

| Need | Approach |
|---|---|
| Find when something happened | Locate all date references; anchor to event described |
| Find who was involved | Extract all named entities; check roles and context |
| Find what was agreed or decided | Look for declarative clauses, signatures, resolutions |
| Count occurrences across docs | Tally mentions of the target event across all documents |
| Check if a claim is supported | Find direct assertion; check for corroboration or contradiction |
| Reconstruct a timeline | Extract all dated events; sort chronologically |
| Find an amount | Locate monetary figures; check context to confirm what they refer to |
| Infer a relationship | Two entities co-appear in a document → they have a documented relationship |

Write your plan like this:
```
Doc "invoice_2024_03": extract date, amount, and which lawyer it is billed to
Doc "meeting_notes_jan": extract attendees, date, and stated agenda items
Across all docs: find all dates on which lawyer X and customer Y co-appear
```

---

## Step 5: Execute Analysis — One Document at a Time

Execute the plan from Step 4. For each document:

1. Read the full content. Do not skim.
2. Apply the planned extraction step by step.
3. After extracting from each document, note:
   - What you found that directly answers the query (or part of it)
   - What is mentioned but ambiguous
   - What is notably absent (e.g. no date, no signatory, no resolution)
4. If the document is long (> 1500 words), work section by section. Summarize each section in
   one sentence before extracting from it.
5. If a document appears irrelevant to the query, still note why — do not silently discard it.
   Irrelevance is information.

---

## Step 6: Cross-Document Reasoning — Fuse and Infer

After individual analysis is complete, perform cross-document evidence fusion.

### 6a. Find Document Links
For every pair of documents, check:
- Do they share named entities (same person, same organization, same case number)?
- Do they share dates or overlapping time ranges?
- Does one document reference or reply to another?
- Do monetary amounts or identifiers match across documents?

Write down all links found.

### 6b. Reconstruct Timelines (if relevant to query)
If the query involves sequence of events or frequency:
1. Collect all dated events across all documents
2. Sort chronologically
3. Note gaps — periods with no documentation
4. Note clustering — multiple events in a short window

### 6c. Look for Implicit Evidence
A fact may be inferrable from a document even if it is never stated directly.

Common inference patterns:

| Implicit fact | Evidence to look for |
|---|---|
| Two people met | Both names appear in a single document with a shared date |
| An agreement was reached | A later document references terms that only make sense if an earlier decision was made |
| A payment was made | An invoice exists and a later document references fulfillment or closure |
| A person was present | Their name appears as author, CC, or signatory on a dated document |
| A relationship existed | Two parties co-appear across multiple independent documents |
| A dispute arose | Language shifts from collaborative to adversarial across a document sequence |

For each inferred fact:
- State the inference
- State which document(s) provide the evidence
- Rate confidence: **HIGH**, **MEDIUM**, or **LOW** (see calibration guide at end of skill)

### 6d. Reconcile with Tables (if tables were also retrieved)
If tabular data is also available:
- Does any document mention an entity ID, amount, or date that can be matched to a table row?
- Does a document confirm or contradict something found in a table?
- Does a table record an event that a document provides context for?

Note each reconciliation point explicitly.

---

## Step 7: Detect Contradictions

Before producing output, scan for contradictions across documents:
- Does document A say event X happened on date D1, while document B implies date D2?
- Does one document identify person P as having role R, while another document shows P acting
  outside that role?
- Does a document claim an agreement was never reached, while another document references terms
  of that agreement?
- Are there two versions of the same document with differing content?

For each contradiction:
- Describe it clearly
- Do **not** silently resolve it by picking one side
- Flag it in the output for the downstream synthesis model to handle

---

## Step 8: Assess Document Quality and Completeness

For each document, note:
- **Completeness**: Is the document intact, or does it appear truncated, missing pages, or
  partially extracted (e.g. garbled OCR, missing attachments)?
- **Authority**: Is this an original document or a summary/copy? Does it bear signatures,
  official headers, or case numbers that confirm its authority?
- **Recency**: Is the document current relative to the query, or potentially superseded by a
  later version?
- **Language/format issues**: Is the content in a non-primary language, heavily abbreviated,
  or using domain-specific shorthand that may affect extraction accuracy?

Flag any document where quality issues may limit your confidence in findings derived from it.

---

## Step 9: Produce Structured Evidence Output

Output the following structure. Fill in every field. If a field is unknown, write `null` and
explain why.

```json
{
  "query_interpretation": "One sentence restatement of what the user is asking",

  "documents_analyzed": [
    {
      "id": "doc identifier",
      "inferred_type": "contract | event_record | correspondence | financial | legal | uncertain",
      "inferred_subject": "one sentence: what this document is about",
      "key_entities": ["list of people, orgs, IDs found"],
      "date_range": "earliest and latest dates mentioned, or null",
      "quality_flags": ["truncated | copy_not_original | language_issues | none"]
    }
  ],

  "direct_findings": [
    {
      "finding": "fact stated explicitly in one or more documents",
      "source_docs": ["doc_id_1", "doc_id_2"],
      "confidence": "HIGH | MEDIUM | LOW",
      "verbatim_anchor": "shortest phrase from the document that proves this (under 15 words)"
    }
  ],

  "inferred_findings": [
    {
      "finding": "fact inferred from document content or cross-document patterns",
      "reasoning": "step-by-step explanation of the inference chain",
      "source_docs": ["doc_id_1", "doc_id_2"],
      "confidence": "HIGH | MEDIUM | LOW"
    }
  ],

  "timeline": [
    {
      "date": "ISO date or approximate",
      "event": "what happened",
      "source_doc": "doc_id",
      "confidence": "HIGH | MEDIUM | LOW"
    }
  ],

  "contradictions": [
    {
      "description": "what conflicts",
      "sources": ["doc_id_1", "doc_id_2"],
      "unresolved": true
    }
  ],

  "table_reconciliations": [
    {
      "doc": "doc_id",
      "table": "table_name",
      "reconciliation": "what matched, confirmed, or contradicted"
    }
  ],

  "missing_data": [
    {
      "what_is_missing": "description of gap",
      "impact_on_answer": "how this limits the answer"
    }
  ],

  "answer_summary": "2–4 sentence plain-language summary of the answer, including confidence caveats"
}
```

---

## Confidence Calibration Guide

| Level | Meaning |
|---|---|
| HIGH | Directly and unambiguously stated in document text; no inference required; no contradicting evidence |
| MEDIUM | Requires one inference step, OR supported by two indirect signals, OR document quality is imperfect |
| LOW | Requires multiple inference steps, or evidence is a single indirect or ambiguous signal, or contradicting evidence exists |

When in doubt, go one level lower. Do not inflate confidence to produce a cleaner answer.

---

## Common Failure Modes to Avoid

- **Treating metadata as ground truth.** A document labeled "meeting notes" may be an email
  summary. Always verify against the content.
- **Stopping at explicit statements.** If the query asks how many times X happened and the
  documents only log half of those events, look for implicit evidence of the rest.
- **Ignoring document quality.** A truncated document that says nothing about a topic is not
  evidence the topic was never mentioned.
- **Conflating author with subject.** A document written by lawyer X about customer Y is about Y,
  not X. Extract both; assign roles correctly.
- **Merging contradictions.** If two documents disagree, report both versions. Do not average
  or pick the more recent one without flagging the conflict.
- **Over-confident inference.** Each inference step reduces confidence. If you inferred A from B
  and then inferred C from A, that is two steps — confidence should be LOW regardless of how
  plausible the chain feels.
- **Discarding "irrelevant" documents silently.** Note why a document was not used. It may be
  relevant to the synthesis model even if it does not directly answer the query.
