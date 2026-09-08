# Beyond Full-Schema Prompting

**A Graph-based Semantic Layer for Accurate and Context-Efficient Text-to-SQL**

This repository contains the code, experiment pipeline, and analysis for a study
of graph-based schema selection in Text-to-SQL. Instead of injecting an entire
relational schema into every LLM prompt, we represent the schema as a semantic
graph (Table → Column → Concept → Relationship) in Neo4j and retrieve only the
question-relevant subset via hybrid retrieval (vector + full-text + graph
traversal), optionally followed by deterministic pruning.

> **Anonymized for review.** This repository is anonymized; author and
> institution details are intentionally omitted.

---

## Key findings

Evaluated on Spider (dev, 1,034 questions) and BIRD mini-dev (500 questions)
across three model capability tiers, over 32,598 generations.

1. **Parity with full-schema prompting.** When retrieval recall is preserved,
   hybrid selection (C4) matches full-schema prompting (C1) in Execution
   Accuracy — confirmed by a TOST equivalence test within a ±2 pp margin on
   Spider (all tiers) and BIRD (strong & weak tiers).
2. **Significant token savings.** C4 reduces context tokens with high
   significance (Wilcoxon p < 1e-60); aggressive pruning (C5-L2) reaches ~60%
   savings on BIRD for a small accuracy cost, forming an accuracy–token Pareto
   frontier.
3. **The bottleneck is schema linking.** Execution Accuracy tracks retrieval
   recall, not model size. Injecting external knowledge (BIRD `evidence`) into
   the retrieval query restored recall (0.84 → 0.97) and recovered accuracy —
   direct evidence that the practical bottleneck is semantic-to-structural
   linking, not generation.
4. **Pruning is a Pareto trade-off, independent of model size.** Higher pruning
   strength monotonically lowers tokens, recall, and accuracy; the
   strength × model interaction is not statistically significant.

---

## Repository structure

```
.
├── data/                     # datasets (built by the prep notebook; not committed)
│   ├── D1/                   # Spider (questions.json, schema.json, db/*.sqlite)
│   ├── D2/                   # BIRD mini-dev
│   ├── D2_fin/               # BIRD financial subset (pilot)
│   └── _raw/                 # downloaded/extracted sources (not committed)
├── notebooks/
│   ├── 00_setup_and_config.ipynb      # paths, model tiers, grayscale plot style, cached LLM client
│   ├── 00b_prepare_datasets.ipynb     # download + convert Spider/BIRD to the unified format
│   ├── 01_build_semantic_graph.ipynb  # build graph + per-condition contexts (C1–C5)
│   ├── 02_run_generation.ipynb        # generate SQL, validate, execute, score
│   ├── 03_rq1_accuracy.ipynb          # RQ1: accuracy vs conditions
│   ├── 04_rq2_token_cost.ipynb        # RQ2: token savings, Pareto, break-even
│   ├── 05_rq3_join_complexity.ipynb   # RQ3: accuracy vs JOIN complexity
│   ├── 06_rq4_pruning_slm.ipynb       # RQ4: pruning strength × model
│   ├── 07_rq5_finance_domain.ipynb    # RQ5: finance-domain pilot + PII audit
│   ├── 08_ablation.ipynb              # retrieval-signal ablation + synthesis figure
│   └── 09_significance_tests.ipynb    # McNemar, TOST equivalence, bootstrap CIs
└── results/
    ├── figures/              # generated figures (grayscale, PNG + PDF, 600 dpi)
    └── tables/               # generated tables (CSV + LaTeX)
```

---

## Experimental design

**Conditions (context construction).**

| ID    | Name                | Description                                             |
|-------|---------------------|--------------------------------------------------------|
| C1    | Full Schema         | entire schema serialized (baseline)                    |
| C2    | Vector-only         | top-k tables/columns by embedding similarity           |
| C3    | Graph-only          | traversal from full-text anchor matches                |
| C4    | Hybrid              | vector + full-text + graph traversal (recall-oriented) |
| C5-L1 | Pruning (light)     | drop FK-isolated tables                                 |
| C5-L2 | Pruning (medium)    | join-path minimization + embedding-preserving columns  |
| C5-L3 | Pruning (aggressive)| strict token/type projection                           |

**Datasets.** D1 = Spider (small schemas), D2 = BIRD mini-dev (large schemas),
D2_fin = BIRD financial subset (small pilot).

**Models.** Three capability tiers (strong / mid / weak) configured via `.env`;
these are capability proxies, not literal parameter counts.

**Metrics.** Execution Accuracy (primary), SQL validity (SQLGlot), context tokens
and cost, and schema-linking recall/precision (strict `table.column` and relaxed
column-name matching).

---

## Setup

### Requirements

```bash
pip install openai python-dotenv pandas numpy matplotlib seaborn \
            sqlglot tqdm networkx scipy datasets huggingface_hub
```

Python 3.10+ recommended. A CUDA GPU is not required (all model calls are via API).

### Credentials

Create `notebooks/.env` with your API key and the three model tiers:

```
OPENAI_API_KEY=sk-...
LLM_MODEL_WEAK=<small model>
LLM_MODEL_MID=<mid model>
LLM_MODEL_STRONG=<large model>
```

The client tries `max_completion_tokens` first and falls back to `max_tokens`
for older models automatically. All LLM and embedding calls are cached to
`notebooks/.llm_cache/`, so re-runs are cheap and reproducible.

### Data

Spider and BIRD questions/schemas download automatically via Hugging Face in
`00b_prepare_datasets.ipynb`. The **BIRD SQLite databases are not on Hugging
Face**: download the official BIRD mini-dev package, extract it, and place the
SQLite databases so that the notebook can stage them into
`data/_raw/bird/dev_databases/<db_id>/<db_id>.sqlite`. See the notebook's
staging cell for details. Spider databases are fetched automatically.

Datasets and databases are **not committed** to this repository due to size and
licensing; run the prep notebook to reconstruct them.

---

## Reproducing the results

Run the notebooks in order:

1. `00_setup_and_config.ipynb` — configuration (run first; other notebooks call it).
2. `00b_prepare_datasets.ipynb` — download and convert Spider/BIRD.
3. `01_build_semantic_graph.ipynb` — build graph and per-condition contexts.
4. `02_run_generation.ipynb` — generate, validate, execute, and score (writes
   `results/tables/generation_results.csv`, the single input for all analysis).
5. `03`–`09` — analysis and figures (each reads the master results table).

Each notebook has a `SMOKE_TEST` switch for a small, cheap dry run before the
full experiment.

### Notes on reproducibility

- Generation is deterministic (`temperature=0`); all calls are cached by a hash
  of (model, messages, parameters).
- Gold schema-linking labels are derived by parsing each gold SQL with SQLGlot
  and qualifying columns against the schema (so `age` → `singer.age`).
- SQL execution is guarded by a wall-clock limit to prevent pathological queries
  from hanging the run.
- Figures are grayscale, caption-free, saved as both PNG and PDF at 600 dpi.

---

## License

Code is released under the MIT License (see `LICENSE`). Spider and BIRD are
distributed under their own licenses; please consult the original benchmark
releases before use.

## Citation

A citation entry will be added upon publication. (Anonymized for review.)
