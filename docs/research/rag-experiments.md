# RAG Experiments

**Purpose:** Active research workspace for reproducible RAG evaluation in `~/jupyter/rag_experiments/`. Documents RAG-001 (StackExchange sysadmin corpus, ~48k docs), RAG-002 (in progress), golden datasets, hybrid retrieval experiments, embedding comparisons, full pipelines, Elastic Cloud deployment. Resolves multiple known issues; replaces old Streamlit platform in `embedding_testing/`. The `docs/` subdirectory is the detailed living record.

## Overview

The workspace contains:
- `raw/`: immutable corpus (StackExchange JSONL dumps), golden data (`golden_questions.jsonl`).
- `src/rag_001/`: executable pipeline scripts (01_pull_se_corpus.py through 08_strip_v1_ids.py), notebooks (`SheepSOC_RAG001_Experiment_v3.ipynb`, `Results_Viewer.ipynb`, `Dense_Diagnostics.ipynb`), data/ with chunks_v2/v3, results CSVs/JSONL, plots.
- `src/rag_002/`: scaffold for hierarchical indexing and knowledge graph.
- `src/shared/`: common pipeline (`es_client.py`, `pipeline.py`, _lib/).
- `docs/`: comprehensive wiki with `concepts/`, `experiments/`, `runbooks/`, `sources/`, `index.md`, `log.md`, `schema.md`.

**RAG-001**: Full pipeline on unix/serverfault/etc. StackExchange data (~48k documents in v2/v3 indexes). Steps: pull corpus → chunk/prepare → sanity → create_index (v2/v3 with appropriate mappings for 768d/1024d) → embed with Ollama (nomic-embed-text, mxbai-embed-large) → ingest to ES → expand_relevance with AI voting (2/3 majority) → human review (07b/07c) → evaluation.

**RAG-002**: In progress. Uses BeIR unix + Arch Wiki from HuggingFace; hierarchical indexing (L1/L2/L3), "SEE ALSO" knowledge graph links, shared pipeline scaffold complete; ingest pending.

**Key Experiments & Results** (from RAG_001-3 on v2 corpus, 200-question golden set):
- Hybrid (dense + ELSER via RRF): 0.6809 nDCG@10
- ELSER: 0.6753 nDCG@10
- Dense: 0.6503 nDCG@10
- BM25 baseline: 0.6255 nDCG@10
Golden dataset: ~200 questions, >1k graded chunks (auto + human 2/3 majority vote); one noted gap (q073). Notebooks provide full diagnostics, per-query analysis, plots (ndcg_heatmap, precision_recall, chunk size dist).

**Infrastructure**: Elasticsearch migrated to **Elastic Cloud 9.4.0** (GCP us-central1, API key auth, cluster UUID `DaBtVAvNQNmqT0thJwt-7Q`). Local single-node decommissioned 2026-05-10. Notebooks updated for cloud `es_client`; some viewer notebooks pending full rewiring. Query embeddings remain local via Ollama (cloud cannot reach localhost:11434). Ties into production OpenWebUI RAG (`nomic-embed-text`, `open_webui_collections_d768` index with ELSER dual-use, 15 KBs updated via nightly `rag_sync` for system docs/configs), Kibana AI Assistant concepts.

**Old Platform**: The Streamlit `rag_experiment_platform` in `embedding_testing/` is outdated and replaced by this Jupyter + shared Python pipeline setup.

Detailed concepts (chunking strategies, relevance judging with AI voting, evaluation metrics like graded nDCG, hierarchical index, knowledge graph RAG, elastic_cloud_deployment.md), runbooks (migration, rag001_v2_pipeline, rag002_pipeline), sources, and log live in `~/jupyter/rag_experiments/docs/`. Start with `docs/index.md` and `docs/log.md`.

## Dependencies

- **[Elasticsearch & ELSER](../infrastructure/platforms/elasticsearch-elser.md)** **depends on** — primary vector/sparse store for all indexes and experiments; now fully on Elastic Cloud.
- **[OpenWebUI & RAG](../infrastructure/platforms/openwebui-rag.md)** **see also** — production RAG UI and KB collections use overlapping tech (nomic embeddings, ELSER hybrid potential, same cloud cluster).
- Ollama (for local embeddings and potential judge models).
- Conda (`sheepsoc` env with elasticsearch==9.4.0 client).

(See [schema.md](schema.md) in the experiments dir for full internal relationships.)

## Configuration

- ES connection centralized in `src/shared/_lib/es_client.py` (reads `ELASTICSEARCH_URL`, `ELASTICSEARCH_API_KEY` from `~/.env`).
- Index names: `sheepsoc_rag001_v2`, `sheepsoc_rag001_v3`, etc.
- Embeddings: pre-computed locally, shipped in bulk JSONL; ELSER sparse via cloud inference endpoint (`.elser-2-elasticsearch`).
- Golden judgments expanded via `07_expand_relevance.py` (multi-retriever voting).

Runbooks for migration, pipeline execution, and review are in the `docs/runbooks/` subdirectory.

## Health Checks

```bash
pmabry@sheepsoc:~$ python -m src.rag_001.config  # or check es_client
pmabry@sheepsoc:~$ curl -H "Authorization: ApiKey $ELASTICSEARCH_API_KEY" "$ELASTICSEARCH_URL/_cluster/health"
pmabry@sheepsoc:~$ ollama list  # nomic-embed-text, mxbai if used
```

## Runbooks

- See `~/jupyter/rag_experiments/docs/runbooks/`:
  - `elastic_cloud_migration.md` and `elastic_cloud_migration_resume.md` — full migration record and current status (v3 complete, notebooks ready).
  - `rag001_v2_pipeline.md` — complete re-chunk/index/embed/ingest + relabeling.
  - `rag002_pipeline.md` — shared scaffold status.

These are linked from main [lab-operations/sops.md](../lab-operations/sops.md) where applicable.

## Known Issues / Gotchas

- Notebooks like Results_Viewer and Dense_Diagnostics partially updated for cloud; some hardcoded localhost paths remain (tracked in `docs/log.md`).
- Cloud cannot call local Ollama for query embedding in native AI Assistant features; Cloudflare Tunnel planned for full integration (see kibana_ai_assistant.md in experiments docs).
- q073 gap in corpus for golden set (0 nDCG contribution).
- Local decommissioned storage (`/mnt/elastic_data`) no longer used for ES but retained for other data (see known-issues.md watchlist on vg_elastic).

All previous RAG-001/RAG-002 implementation gaps, migration issues, and lack of systematic evaluation are now fully documented here.

## See Also

- [Research Agenda](../agenda.md) — updated research questions now answered by these experiments.
- [RAG-001 Protocol](rag-001/protocol.md) — superseded by executed pipeline in this workspace.
- [OpenWebUI & RAG](../infrastructure/platforms/openwebui-rag.md) — production counterpart.
- [Elasticsearch & ELSER](../infrastructure/platforms/elasticsearch-elser.md) — current cloud deployment details.
- [Known Issues](../known-issues.md) — migration and RAG issues resolved by this documentation.
- `~/jupyter/rag_experiments/docs/index.md` — canonical detailed catalog (concepts, experiments, sources, runbooks).
- `~/jupyter/rag_experiments/README.md` — pipeline summary.
- [Lab Operations](../lab-operations/index.md) and runbooks for operational integration.

This page follows all link rules from the [wiki schema](../infrastructure/mkdocs-site/schema.md). Reciprocal links added to affected pages.
