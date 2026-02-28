# BVZyme RAG Pipeline

## What it does
Semantic search over 35 enzyme technical documents (TDS + REF).
Ask a question in any language → get the 3 most relevant fragments.

---

## Pipeline

```
NODE 1  → PDF Extraction           (raw_blocks)
NODE 2  → Cleaning + Normalization  (cleaned_blocks)
NODE 3  → Translation FR→EN         (translated_blocks)
NODE 4  → Chunking                  (381 chunks)
NODE 5  → Embedding                 (index: all-MiniLM-L6-v2, 384 dims)
NODE 5.5→ Synonym Expansion         (EXPANSION_DICT)
NODE 6  → Query Router              (path1 / path2 / rephrase)
NODE 7  → Path 1: Exact Lookup      (product code detected)
NODE 8  → Path 2: Semantic Search   (top-k=3, cosine similarity)
NODE 9  → Full Pipeline             (multilingual entry point)
NODE 10 → User Prompt               (input → answer)
NODE 11 → PostgreSQL Database       (optional, fill DB_CONFIG)
```

---

## Corpus

| Type | Count | Language |
|------|-------|----------|
| TDS (Technical Data Sheets) | 34 | English |
| REF (Reference doc) | 1 | French |
| **Total chunks** | **381** | — |

---

## Routing Logic

```
Query
 ├── Product code found?  → Path 1 (exact lookup)
 ├── Functional query?    → Path 2 (semantic search)
 └── Unclear?             → Ask to rephrase
```

---

## Key Features

- **Multilingual** — detects query language, translates to English for search, returns answer in original language
- **Synonym expansion** — French queries expanded with corpus vocabulary before embedding (English queries untouched)
- **Hybrid routing** — regex for product codes, NLP classifier for functional queries
- **PostgreSQL ready** — fill in `DB_CONFIG` in Node 11 when credentials available

---

## Embedding Model

| Property | Value |
|----------|-------|
| Model | `all-MiniLM-L6-v2` |
| Library | `sentence-transformers` |
| Dimension | 384 |
| Similarity | Cosine |

---

## Database (Node 11)

```sql
TABLE embeddings (
    id             SERIAL PRIMARY KEY,
    id_document    INT,
    texte_fragment TEXT,
    vecteur        VECTOR(384)
)
```
Set `DB_CONFIG = None` to skip gracefully when credentials are unavailable.

---

## Usage

```python
# Run nodes 1 → 11 in order, then:
query = input("Your question: ")
prompt(query)
```

---

## Known Limitations

- 34 TDS documents are structurally similar → semantic scores naturally lower (0.50–0.75)
- `TG MAX63` → 10 blocks, `TG MAX64` → 9 blocks (sections missing in source PDFs)
- PostgreSQL data resets on Colab session restart → re-run Node 11
