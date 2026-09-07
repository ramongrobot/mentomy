# Corpus Entropy Benchmark

A controlled synthetic dataset for evaluating the relationship between **corpus quality** and **answer groundedness** in Retrieval-Augmented Generation (RAG) systems.

This repository accompanies the paper:

> **A Hierarchical Consistency Framework for Auditing Retrieval-Augmented Generation Systems**
> Submitted to *Arxiv*

## What this dataset is for

Most RAG benchmarks assume that adding more documents to a corpus improves answer quality. This dataset is designed to test the opposite: that beyond a certain point, additional documents introduce **noise** (intra-document degradation) and **conflict** (inter-document disagreement) that lower answer confidence even when surface retrieval similarity remains stable.

The dataset operationalises this through four controlled corpora that vary independently along two axes:

| Corpus | Documents | Intra-Document Noise (IaDNI) | Inter-Document Conflict (IeDCD) |
|---|---|---|---|
| **A** (Baseline) | 5 | Low | Low |
| **B** (Conflict) | 10 | Low | High |
| **C** (Noise) | 5 | High | Low |
| **D** (Combined) | 20 | High | High |

Each corpus covers the same five science domains — **physics, medicine, history, geography, biology** — at roughly 2,500 words per document, with all quantitative facts expressed in natural-language prose (no equations, no LaTeX) to control for tokenisation variance across embedding models.

A 25-query evaluation set, stratified into conflicted-variable, non-conflicted, and structurally-degraded queries, allows direct measurement of how Answer Confidence Score (ACS) responds to changes in Corpus Entropy.

## Repository structure

```
corpus-entropy-dataset/
├── README.md                          # this file
├── dataset_card.md                    # standard dataset documentation
├── LICENSE                            # CC-BY-4.0 for dataset, MIT for code
├── corpus_A/                          # 5 clean baseline documents
│   ├── doc_A_physics.md
│   ├── doc_A_medicine.md
│   ├── doc_A_history.md
│   ├── doc_A_geography.md
│   └── doc_A_biology.md
├── corpus_B/                          # 10 docs: 5 clean + 5 conflict-injected
│   ├── doc_A_physics.md … (copies)
│   └── doc_B_physics.md … (5 conflict-injected)
├── corpus_C/                          # 5 structurally degraded documents
│   └── doc_C_physics.md …
├── corpus_D/                          # 20 docs: 5 clean + 5 conflict + 5 structural + 5 combined
│   ├── doc_D_clean_*.md
│   ├── doc_D_conflict_*.md
│   ├── doc_D_structural_*.md
│   └── doc_D_combined_*.md
├── ground_truth_conflict_map.md       # human-readable conflict map
├── ground_truth_conflict_map.csv      # machine-readable; 40 conflicts
├── query_set.md                       # human-readable query set
└── query_set.csv                      # machine-readable; 25 queries
```

## Dataset at a glance

- **35 unique documents** across 5 domains, totalling approximately 88,000 words
- **40 controlled factual conflicts** (entity-swap strategy following Longpre et al. 2021, extended with three additional conflict types)
- **25 evaluation queries** with full metadata, stratified by query type and cross-referenced to the conflict map
- **All facts in Corpus A are real, verifiable scientific facts** sourced from canonical references (CODATA, ACC/AHA and ESC clinical guidelines, mainstream historiography, USGS, Guyton & Hall *Textbook of Medical Physiology*)
- **No equations.** All quantitative claims are expressed in prose to eliminate tokenisation noise across embedding models

## Quick start

### Loading the corpora

```python
from pathlib import Path

def load_corpus(name: str) -> dict[str, str]:
    """Load all documents in a named corpus into a {doc_id: text} dictionary."""
    return {
        p.stem: p.read_text(encoding="utf-8")
        for p in Path(f"corpus_{name}").glob("doc_*.md")
    }

corpus_a = load_corpus("A")   # 5 docs
corpus_b = load_corpus("B")   # 10 docs (5 clean + 5 conflict)
corpus_c = load_corpus("C")   # 5 docs
corpus_d = load_corpus("D")   # 20 docs
```

### Loading the query set

```python
import csv

with open("query_set.csv", encoding="utf-8") as f:
    queries = list(csv.DictReader(f))

assert len(queries) == 25
print(queries[0]["query_text"])
# → "What is the speed of light in vacuum?"
```

### Loading the conflict map

```python
import csv

with open("ground_truth_conflict_map.csv", encoding="utf-8") as f:
    conflicts = list(csv.DictReader(f))

assert len(conflicts) == 40

# Get all conflicts for a specific query
q = queries[0]
linked = [c for c in conflicts if c["conflict_id"] == q["linked_conflict_id"]]
```

## Suggested evaluation protocol

For each pipeline configuration:

1. **For each corpus** (A, B, C, D): index the documents in your retrieval system.
2. **For each query** in the query set: retrieve top-*k* passages and generate an answer.
3. **Record per query, per corpus**: the answer text, the top-1 retrieval similarity ($RS_1$), the mean and standard deviation of similarity scores ($RS_\mu$, $RS_\sigma$), and the Answer Confidence Score (ACS) of your choice.
4. **Compare**: aggregate ACS by query type (conflicted-variable, non-conflicted, structurally-degraded) and plot against the corpus's measured Corpus Entropy (CE).

The headline plot of the accompanying paper is mean ACS on the y-axis against measured CE on the x-axis, with one trace per query type. The dataset is designed so that this plot should show:

- A steep ACS drop for **conflicted-variable** queries as corpus entropy increases (B → D)
- A shallow ACS drop for **non-conflicted** queries (controls for retrieval-quality effects)
- A steep ACS drop for **structurally-degraded** queries between Corpus A and Corpus C/D, even without injected conflicts

Departures from this pattern are themselves informative — see the discussion of **conflict spillover** in `query_set.md` (specifically Q15 ↔ Q10 and Q19).

## Why we built this

Real-world enterprise document collections — the kind that small and mid-sized companies feed into private RAG systems — are typically full of duplicates, outdated documents, conflicting policies, and fragmented internal knowledge. Existing RAG benchmarks (e.g., MS MARCO, Natural Questions, HotpotQA) test retrieval and reasoning under the assumption that the corpus is high-quality. They do not isolate the effect of corpus quality on answer groundedness.

This dataset is designed to fill that gap: it provides a controlled environment in which corpus size, intra-document noise, and inter-document conflict are independently varied, with ground-truth conflicts mapped at the (entity, relation) level. It is small enough to evaluate cheaply, structured enough to support clean comparisons, and rich enough to expose the failure modes that matter in production RAG systems.

## How to cite

If you use this dataset in academic work, please cite:

```bibtex
@misc{gonzalez2026corpusentropy,
  title        = {A Hierarchical Consistency Framework for Auditing Retrieval-Augmented Generation Systems},
  author       = {Gonz{\'a}lez, Ram{\'o}n and Mentomy AI},
  year         = {2026},
  howpublished = {\url{https://github.com/ramongrobot/mentomy}},
  note         = {Companion dataset to \emph{A Hierarchical Consistency Framework for Auditing Retrieval-Augmented Generation Systems},
                  submitted to Arxiv}
}
```

## Licence

- **Dataset** (all `.md` and `.csv` files in the corpus and metadata directories): [Creative Commons Attribution 4.0 International (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/). You are free to share and adapt for any purpose, including commercial use, as long as you provide attribution.
- **Code** (any Python or shell scripts in this repository): [MIT License](https://opensource.org/license/mit).

## Issues and contributions

This is a versioned dataset. The current release is `v1.0`. Errata, suggested additional conflicts, or new query types are welcome as GitHub issues. Pull requests that would change the existing 40 conflicts or 25 queries will not be merged into `v1.0` (to preserve reproducibility for the published paper) but will be considered for a future `v2.0`.

## Contact

Ramón González Sánchez — Mentomy AI
[https://github.com/ramongrobot/mentomy](https://github.com/ramongrobot/mentomy)
