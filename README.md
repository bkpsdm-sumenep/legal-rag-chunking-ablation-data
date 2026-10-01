# Data for "Context-Unit Completeness Explains the Gains of Structure-Aware Chunking"

Data accompanying the paper *Context-Unit Completeness Explains the Gains of Structure-Aware Chunking: A Controlled Ablation on Indonesian Civil-Service Regulation RAG* (Moh. Affan, Gunawan, Lukman Zaman; Institut Sains dan Teknologi Terpadu Surabaya). Indonesian title: *Kelengkapan Unit Konteks Menjelaskan Manfaat Chunking Sadar-Struktur: Ablasi Terkontrol pada RAG Regulasi Kepegawaian Indonesia*.

The package contains the test set, every answer and retrieved context for each experimental arm, and per-question scores. All numbers reported in the paper can be recomputed from these files. The system code and experiment scripts are owned by BKPSDM Kabupaten Sumenep and are not included; they are available from the corresponding author on request, subject to permission from BKPSDM Kabupaten Sumenep.

## Contents

| Path | Content |
|:---|:---|
| `data/testset.jsonl` | 109 validated test items (97 in-scope items used in the analysis, 12 out-of-scope) |
| `data/arms.json` | Description and internal run ID of every arm |
| `data/outputs/<arm>.jsonl` | One line per in-scope question: search query, retrieved contexts, generator context, answer, scores |
| `data/scores.csv` | Scores only, one row per arm and question (14 arms × 97 questions) |
| `metadata/collections_manifest.json` | Frozen Qdrant collections used by arms A0 to A4 |
| `metadata/hnsw_recall.json` | HNSW recall@5 against exact search per collection (paper §4.5, Figure 5) |
| `metadata/retrieval_latency.json` | Retrieval latency re-measured in one session (paper Table 8) |
| `results/ablation_summary.json` | Statistical results of the ablation (paired Wilcoxon, Holm correction, McNemar) as produced by the analysis |
| `results/verification.json` | Means recomputed from the files in this package and compared with the analysis output |

## Arms

| Arm | Configuration |
|:---|:---|
| `A0` | Fixed-size 1,000/100 characters, HNSW search (clean corpus) |
| `A0x` | Fixed-size 1,000/100 characters, exact search (clean corpus) |
| `A1` | Retrieve paragraph (*ayat*), send paragraph |
| `A2` | Retrieve paragraph, send parent article (*pasal*); parent-child |
| `A2r` | Repeat run of A2 (run-to-run variation) |
| `A3` | Fixed-size 5,000/500 characters (context-volume control) |
| `A4` | Retrieve article, send article |
| `B0x` | Original-run H2 baseline collection (old corpus), exact search |
| `H2-base`, `H2-treat` | Original run: fixed-size 1,000/100 with HNSW vs parent-child, both with query rewriting and hybrid search (old corpus) |
| `N0`, `N0x`, `PC-dense`, `closed-book` | Dense retrieval without query rewriting (paper Table 10): fixed-size with HNSW, fixed-size with exact search, parent-child, and no retrieval |

Arms A0 to A4, A2r, and A0x use a frozen clean-corpus snapshot. All arms except `closed-book` share the same generator and prompt; see the paper §3 for full settings.

## Fields

`data/testset.jsonl`

- `item_id`, `question` (informal civil-servant style, in Indonesian)
- `reference_answer`: answer after expert review and adjudication; `original_reference_answer`: machine-generated answer before review, when it was revised
- `validation_status` (`approved` or `revised`), `is_out_of_scope`, `in_analysis` (true for the 97 in-scope items)
- `er5_evaluable`: whether the source text could be recovered from the raw corpus text, which is required for Evidence-Retrieved@5 (93 items)
- `source_regulation`, `source_article`, `source_chunk_ids`, `source_chunk_texts`

`data/outputs/<arm>.jsonl`

- `search_query`: query sent to the retriever (after query rewriting where the arm uses it)
- `contexts`: the five retrieved units as sent to the generator; `contexts_meta`: regulation type, number, year, article, validity flag, and retrieval score
- `generator_context`: the full context string the generator received, including citation headers
- `answer`
- `scores`: `context_recall`, `context_precision`, `answer_relevance`, `faithfulness`, `factual_correctness`, all on a 0 to 1 scale (the paper reports them × 100). `faithfulness` is judged against the generator context; it is `null` where only the earlier raw-chunk instrument exists (not used in the paper).
- `evidence_retrieved_at_5`: true if every source text of the item is contained in the retrieved contexts (5-word shingle coverage ≥ 0.9); `null` when the item is not evaluable
- `refusal_detected`: automatic detection of refusal phrases such as "tidak ditemukan", "tidak tersedia", or "mohon maaf" (paper Table 10)
- `latency_s`, `input_tokens`, `output_tokens`, `generation_cost_usd`, `context_chars`

Scores were produced with RAGAS using Gemini models as judges; FactualCorrectness compares the answer with `reference_answer`.

## Reproducing the main comparisons

Paired comparisons in the paper join two arms on `item_id` and use the two-sided Wilcoxon signed-rank test on the per-question difference (× 100), with Holm correction over the eight primary tests (P1 A1 vs A0x, P2 A2 vs A1, P3 A2 vs A3, P4 A2 vs A4; metrics Context Recall and FactualCorrectness), bootstrap 95% confidence intervals (5,000 resamples, seed 42), and the exact McNemar test for Evidence-Retrieved@5. For example, with pandas and SciPy:

```python
import pandas as pd
from scipy.stats import wilcoxon

s = pd.read_csv("data/scores.csv")
w = s.pivot(index="item_id", columns="arm", values="context_recall") * 100
d = (w["A2"] - w["A3"]).dropna()
print(round(d.mean(), 2), wilcoxon(w.loc[d.index, "A2"], w.loc[d.index, "A3"]).pvalue)
```

## Privacy and sources

- The corpus consists of 71 Indonesian civil-service regulations, which are public documents. Contexts are quoted from these regulations.
- Civil-servant ID numbers (NIP) appearing in regulation text, for example in worked examples or certification blocks, are replaced with `[NIP]`. Names that appear in the published regulations are kept as published.
- No personal data of system users is included. The test set contains no reviewer identities.

## License

This dataset is licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). You may share and adapt the data for any purpose, provided you give appropriate credit by citing the paper. See `LICENSE` for the full legal code.

The regulation texts quoted in the contexts are Indonesian laws and regulations, which are not subject to copyright under Article 42 of Law No. 28 of 2014 on Copyright.

## Citation

[TODO: add the paper citation and, if deposited, the Zenodo DOI]

## Contact

Moh. Affan, b4affan@gmail.com (ORCID 0009-0009-2452-1437)
