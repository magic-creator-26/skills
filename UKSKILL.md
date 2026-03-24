---
name: enterprise-knowledge
description: >
  Authoritative glossary and semantic reference for the organization's two core data systems —
  **Entities** (the entity knowledge base) and **AllTables** (the org-wide canvas table system) —
  plus the internal domain language used across the data platform.

  ALWAYS consult this skill when:
  - An agent or query references a term it cannot resolve (field name, table name, entity type, metric name)
  - The agent is unsure which system owns a specific data concept
  - A Hebrew or transliterated term appears in a query or table schema
  - A column connection or relationship in AllTables is ambiguous
  - Any term feels like internal jargon that isn't self-explanatory from schema alone
  - The agent is about to guess at a definition — stop and look it up here instead

  Do NOT skip this skill just because the term looks familiar. Many internal terms share names
  with generic data concepts but carry org-specific meaning. When in doubt, check.
---

# Enterprise Knowledge Base

This skill is the single source of truth for enterprise data semantics. It contains:

1. **Entities system** — entity types, their canonical attributes, identifier fields, lifecycle states
2. **AllTables system** — table types, column semantics, canvas connection patterns
3. **Internal terminology** — org-specific vocabulary, Hebrew terms, abbreviations, and concepts

---

## How to Use This Skill

When your agent encounters an unknown term, follow this lookup flow:

```
Unknown term
  │
  ├─ Sounds like an entity type, entity attribute, or identity field?
  │     → Read references/entities.md
  │
  ├─ Sounds like a table name, table type, column name, or canvas relationship?
  │     → Read references/alltables.md
  │
  ├─ Sounds like internal jargon, an abbreviation, or a Hebrew/transliterated term?
  │     → Read references/internal-terms.md
  │
  └─ Still not found?
        → Check all three files. If absent, surface the unknown term to the user
          and request clarification — do not guess.
```

**Important**: Terms often appear in more than one reference file with complementary information.
If you find a partial match in one file, check the others before forming your answer.

---

## Reference Files

| File | Contents | When to read |
|------|----------|--------------|
| `references/entities.md` | Entity types, attributes, ID fields, ontology types | Entity resolution, identity fields, lifecycle concepts |
| `references/alltables.md` | Table types, column semantics, canvas patterns, system mappings | Table selection, column connections, source system routing |
| `references/internal-terms.md` | Internal vocabulary, Hebrew terms, abbreviations, domain-specific concepts | Any unfamiliar term not clearly a table or entity concept |

---

## Keeping This Skill Current

The `scripts/ingest_docs.py` script populates and updates the reference files from a folder
of source documents (PDFs, Word docs, PowerPoint presentations).

Run it whenever new documentation is added:

```bash
python scripts/ingest_docs.py --docs-folder /path/to/your/docs --skill-dir /path/to/enterprise-knowledge
```

The script:
- Extracts text from all supported file types (PDF, DOCX, PPTX)
- Uses Claude to identify and classify terms into the three reference files
- Deduplicates against existing entries so re-runs are safe
- Preserves manually added entries

See `scripts/ingest_docs.py` for full options including `--dry-run` and `--file` for single-file ingestion.

---

## Reference File Format

Each reference file uses a consistent format for easy lookup:

```markdown
## TERM_NAME
**System**: Entities | AllTables | Cross-system
**Type**: entity_type | table | column | identifier | metric | concept | abbreviation
**Hebrew/Alias**: (if applicable)

Definition and semantic meaning.

**Fields / Attributes**: (for entities and tables)
- `field_name` — description

**Notes**: edge cases, caveats, relationships to other terms
```
