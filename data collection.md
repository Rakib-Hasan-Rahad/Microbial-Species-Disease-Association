

# Data Collection Report  
## Microbial Species–Disease Association Text-Mining Project

### 1. Purpose
This document defines all datasets required for an end-to-end text-mining pipeline to extract **microbial species–disease associations** from published literature, along with recommended storage formats and schemas. The goal is to ensure reproducibility and enable downstream evaluation and analysis.

---

## 2. Literature Corpus (Papers)

### 2.1 Data to collect (per paper)
- PMID (required)
- Title (optional but recommended)
- Abstract text (required)
- Publication year/date (recommended)
- Journal (optional)
- MeSH headings (highly recommended; can be used as disease labels)
- Query tag (to track which search created the record)

### 2.2 Recommended source
- PubMed via NCBI E-utilities (esearch/efetch)
- Optional future extension: PubMed Central Open Access (full text)

### 2.3 Storage format
**papers.csv** (or SQLite table `papers`)

**papers.csv schema**
| column | type | example | notes |
|---|---|---|---|
| pmid | int/string | 12345678 | primary key |
| title | text | "Gut microbiome..." | optional |
| abstract | text | "We found..." | required |
| pub_year | int | 2021 | recommended |
| journal | text | "Nature" | optional |
| mesh_terms | JSON string | `["Crohn Disease","Colitis"]` | recommended |
| query_tag | text | "gut_query_v1" | reproducibility |

---

## 3. Sentence Corpus (Derived)

### 3.1 Data to collect (per sentence)
- sentence ID
- PMID
- sentence index/order
- sentence text

### 3.2 Storage format
**sentences.csv**

**sentences.csv schema**
| column | type | example |
|---|---|---|
| sentence_id | string | `12345678_05` |
| pmid | int | 12345678 |
| sent_index | int | 5 |
| sentence | text | "Faecalibacterium prausnitzii was reduced..." |

---

## 4. Microbe Dictionary (NCBI Taxonomy)

### 4.1 Data to collect
- TaxID (NCBI)
- canonical scientific name
- synonyms/aliases (including abbreviations where possible)
- rank (species/genus/etc.)
- optional lineage

### 4.2 Sources
- NCBI Taxonomy dump (names.dmp, nodes.dmp), or NCBI taxonomy API

### 4.3 Storage format (recommended: two tables)
**microbe_taxa.csv** (canonical taxa)
| column | type | example |
|---|---|---|
| taxid | int | 562 |
| canonical_name | text | "Escherichia coli" |
| rank | text | "species" |
| parent_taxid | int | 561 |
| lineage | text | "Bacteria; Proteobacteria; ..." |

**microbe_aliases.csv** (aliases → TaxID)
| column | type | example |
|---|---|---|
| alias | text | "E. coli" |
| taxid | int | 562 |
| alias_type | text | "abbrev/synonym/scientific" |

---

## 5. Disease Normalization Resource

### Option A (recommended for beginners): PubMed MeSH-based diseases
Use MeSH headings attached to each PMID as disease labels.

**paper_diseases.csv** (optional if already in papers.csv)
| pmid | disease_name | id_system | disease_id |
|---|---|---|---|
| 12345678 | Crohn Disease | MeSH | D003424 |

Optionally maintain a list of disease MeSH terms for filtering:
**mesh_disease_terms.csv**
| mesh_term | mesh_id | is_disease |
|---|---|---|
| Crohn Disease | D003424 | 1 |

### Option B (advanced): sentence-level disease NER + UMLS/MeSH mapping
If using TaggerOne/SciSpacy later, store disease mentions similarly to microbe mentions (mention text, offsets, normalized ID).

---

## 6. Relation Lexicons (Trigger/Direction/Negation)

### 6.1 Association trigger words
Examples: “associated with”, “linked to”, “correlated with”, “risk factor”, “biomarker”

**lexicon_triggers.csv**
| phrase | category | weight (optional) |
|---|---|---|
| associated with | association | 1.0 |
| correlated with | association | 1.0 |
| causes | causal | 1.5 |

### 6.2 Direction words
**lexicon_direction.csv**
| phrase | direction |
|---|---|
| increased | increase |
| enriched | increase |
| decreased | decrease |
| depleted | decrease |

### 6.3 Negation words/phrases
**lexicon_negation.csv**
| phrase | scope_hint (optional) |
|---|---|
| no association | sentence |
| not associated | local |
| no significant difference | sentence |

### 6.4 Non-claim patterns (recommended)
**lexicon_nonclaim.csv**
| phrase |
|---|
| we investigated |
| this study aimed |
| to explore whether |

---

## 7. Extracted Entity Mentions

### 7.1 Microbe mentions
**microbe_mentions.csv**
| column | example |
|---|---|
| mention_id | `12345678_05_m1` |
| sentence_id | `12345678_05` |
| pmid | 12345678 |
| mention_text | "E. coli" |
| taxid | 562 |
| canonical_name | "Escherichia coli" |
| start_char | 10 |
| end_char | 16 |
| match_method | "dictionary" |

### 7.2 Disease mentions (if sentence-level NER used)
**disease_mentions.csv** (optional; similar schema)

---

## 8. Candidate Pairs (Microbe × Disease)

**candidate_pairs.csv**
| column | example |
|---|---|
| pair_id | `12345678_05_p1` |
| sentence_id | `12345678_05` |
| pmid | 12345678 |
| taxid | 562 |
| microbe_name | "Escherichia coli" |
| disease_id | "D003424" |
| disease_name | "Crohn Disease" |
| distance_tokens (optional) | 6 |

---

## 9. Extracted Relations (Evidence-level Output)

**extracted_relations.csv**
| column | example |
|---|---|
| relation_id | R_000001 |
| pmid | 12345678 |
| sentence_id | 12345678_05 |
| sentence | "…was reduced…" |
| taxid | 562 |
| microbe_name | "Escherichia coli" |
| disease_id | "D003424" |
| disease_name | "Crohn Disease" |
| is_association | 1 |
| direction | decrease |
| trigger_phrase | "reduced" |
| negation_detected | 0 |
| confidence | 0.82 |
| extraction_method | "rule_v1" |

---

## 10. Aggregated Associations (Final Ranked Results)

**associations_aggregated.csv**
| column | example |
|---|---|
| taxid | 562 |
| microbe_name | "Escherichia coli" |
| disease_id | "D003424" |
| disease_name | "Crohn Disease" |
| n_pmids | 12 |
| n_sentences | 20 |
| n_increase | 3 |
| n_decrease | 17 |
| majority_direction | decrease |
| consistency | 0.85 |
| robustness_label | robust |
| score | 3.12 |
| example_pmid | 12345678 |
| example_sentence | "…reduced…" |

---

## 11. Validation Datasets

### 11.1 HMDAD reference
**hmdad_reference.csv** (normalized where possible)
| microbe_name | taxid (optional) | disease_name | disease_id |
|---|---|---|---|

### 11.2 Disbiome reference
**disbiome_reference.csv** (normalized where possible)

---

## 12. Manual Evaluation Sample (Small Gold Set)
**manual_eval.csv**
| column | example |
|---|---|
| relation_id | R_000201 |
| sentence | "X increased in Y" |
| predicted_association | 1 |
| predicted_direction | increase |
| human_association | 1 |
| human_direction | increase |
| notes | ok |

---

## 13. Reproducibility Files
- README.md (instructions)
- queries.json (PubMed query strings + run date + counts)
- config.yaml (thresholds, lexicon paths)
- requirements.txt / environment.yml

---

## 14. Recommended Folder Layout
```
project/
  data_raw/
  data_processed/
  lexicons/
  src/
  notebooks/
  README.md
  config.yaml
```

---
