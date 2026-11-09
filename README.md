# Beyond Full-Schema Prompting: A Graph-based Semantic Layer for Accurate and Context-Efficient Text-to-SQL

Code and experiments for the IEEE Access paper *"Beyond Full-Schema Prompting: A
Graph-based Semantic Layer for Accurate and Context-Efficient Text-to-SQL"*
(Access-2026-45760).

The method builds a **graph-based semantic layer** over a database (schema graph +
mined joins + representative values + optional glossary) and uses **hybrid
retrieval** (vector + full-text + FK graph traversal) to select a compact,
join-closed sub-schema for the LLM, instead of dumping the full schema into the
prompt. The central result: this reaches **accuracy parity** with full-schema
prompting while cutting prompt/context tokens, and the accuracy–token trade-off can
be dialed with graded pruning.

---

## Key findings

- **Parity, stated precisely.** No significant accuracy difference vs. full-schema
  prompting by McNemar on every dataset/tier; formal TOST **equivalence**
  (±0.02 margin) on Spider (all tiers) and weak-model BIRD, with a small
  non-significant residual (≤3 pts) on stronger BIRD tiers reported openly.
- **Recall is necessary but not sufficient.** With the gold-label resolution fix,
  full-schema column recall is **1.000**; graded pruning is **monotone**
  (C4 ⊇ L1 ⊇ L2 ⊇ L3). A failure-case analysis shows 85–99% of hybrid failures
  retrieved every gold column yet still failed — residual errors are generation-side.
- **Evidence helps generation, not retrieval.** A 2×2 evidence factorial: the
  retrieval channel is ≈0 (hybrid retrieval is already recall-saturated), the
  generation channel carries the gain.
- **Token/cost efficiency without latency cost.** The schema-selection pipeline
  (graph + retrieval + pruning) runs in ≈12 ms (Spider) / ≈48 ms (BIRD) — one to two
  orders of magnitude below LLM generation (≈1–2 s). Cost scales with prompt tokens
  (up to ~65% reduction under aggressive pruning).
- **Retrieval limits justified.** A full (k_tables, k_cols) sweep places the chosen
  (8, 30) at the knee of the recall–token curve; k_tables is non-binding because
  graph expansion supplies join-relevant tables regardless of the nominal top-k.

Models evaluated across four tiers (weak → a state-of-the-art reasoning tier);
benchmarks: **Spider**, **BIRD** (mini-dev), a **BIRD-finance** pilot, and
**Spider 2.0-lite** (scale characterization).

---

## Repository layout

```
notebooks/        Jupyter notebooks (the full experiment pipeline + analyses)
  .env            your API keys + model names (not committed — see below)
data/             prepared datasets (built by 00b / 00c)
results/          tables/ and figures/ written by the notebooks
```

> The notebooks are released with outputs so the experiment record is visible on
> GitHub. Re-running is guarded (see **Reproducing**) so you never accidentally
> overwrite the master results.

---

## Setup

Requires Python 3.10+ and Jupyter.

```bash
pip install -r requirements.txt          # openai, pandas, numpy, networkx,
                                         # scikit-learn, sqlglot, matplotlib,
                                         # statsmodels, tqdm, python-dotenv, rank-bm25
```

Create `notebooks/.env`:

```ini
OPENAI_API_KEY=sk-...
LLM_MODEL_WEAK=gpt-4o-mini
LLM_MODEL_MID=gpt-5.4-mini
LLM_MODEL_STRONG=gpt-5.4
LLM_MODEL_REASON=gpt-6-luna      # optional; reasoning-tier rows are skipped if unset
EMBED_MODEL=text-embedding-3-small
```

Start Jupyter from the `notebooks/` folder (paths resolve relative to it).

---

## Notebooks

Run top to bottom for a full build; the analyses only read the master results CSV.

| Notebook | Purpose |
|----------|---------|
| `00_setup_and_config` | Config, paths, model/embedding clients, shared helpers |
| `00b_prepare_datasets` | Build Spider / BIRD / BIRD-finance; **gold-label resolution**; BIRD descriptions |
| `00c_prepare_modern_benchmarks` | Spider 2.0-lite loader + scale characterization |
| `01_build_semantic_graph` | Schema graph, mined joins, values; **monotone pruning**; factorial/ablation conditions |
| `02_run_generation` | Generation sweep (conditions × tiers) → `generation_results.csv` |
| `03_rq1_accuracy` | RQ1 — accuracy & parity (McNemar) |
| `04_rq2_token_cost` | RQ2 — token savings, accuracy–token Pareto, break-even |
| `05_rq3_join_complexity` | RQ3 — accuracy vs. join complexity |
| `06_rq4_pruning_slm` | RQ4 — pruning × model-strength (OLS interaction) |
| `07_rq5_finance_domain` | RQ5 — BIRD-finance domain pilot |
| `08_ablation` | Semantic-layer ablation |
| `09_significance_tests` | McNemar parity + TOST equivalence |
| `10_sota_baselines` | Common-setting SOTA harness (positioning) |
| `R2_1_evidence_factorial_and_ablation` | 2×2 evidence factorial + metadata leave-one-out |
| `R2_3_recall_diagnosis` | Per-question recall diagnosis (Table 1 fixes) |
| `R2_4_latency_cost` | Latency/cost from the master run |
| `R2_4b_stage_timing` | **Per-stage** latency table (self-contained) |
| `R3_failure_analysis` | Failure-case analysis (recall-sufficient failures) |
| `R3c_k_sensitivity` | **(k_tables, k_cols)** sweep + knee figure (self-contained) |

**Conditions:** `C1` full schema · `C2` vector · `C3` graph · `C4` hybrid ·
`C5-L1/L2/L3` graded pruning · evidence factorial `EV00/EV10/EV01/EV11` ·
ablations `ABL_full/nograph/nogloss/noval/nojoin`.

---

## Reproducing

The heavy notebooks detect their existing outputs and **skip by default**, so
analyses and the two self-contained experiments never re-trigger the ~65k-generation
sweep or overwrite `generation_results.csv`:

| Notebook | Skips when… | Force a redo |
|----------|-------------|--------------|
| `00b_prepare_datasets` | `data/D1/questions.json` exists | `os.environ['FORCE_PREPARE']='1'` |
| `01_build_semantic_graph` | a `*__context_meta.csv` exists | `os.environ['FORCE_BUILD_CONTEXTS']='1'` |
| `02_run_generation` | `generation_results.csv` exists | `os.environ['FORCE_GENERATION']='1'` |

`R3c_k_sensitivity` and `R2_4b_stage_timing` are **fully self-contained** (pipeline
definitions inlined; no `%run`, no imports, no generation) and run offline on cached
embeddings + local SQLite.

To reproduce from scratch: set the env keys, then run
`00 → 00b → 01 → 02` with the force flags set, followed by the analysis notebooks.

---

## Citation

```bibtex
@article{yang2026beyondfullschema,
  title   = {Beyond Full-Schema Prompting: A Graph-based Semantic Layer for
             Accurate and Context-Efficient Text-to-SQL},
  author  = {Yang, Y. and others},
  journal = {IEEE Access},
  year    = {2026},
  note    = {Manuscript Access-2026-45760}
}
```

## License

Released for research use. See `LICENSE`.
