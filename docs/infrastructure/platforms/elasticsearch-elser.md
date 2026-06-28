# Elasticsearch, ELSER & RAG Experiments

**Purpose:** Elasticsearch deployments on sheepsoc — both Elastic Cloud 9.4.0 (GCP us-central1, used for research experiments and OTEL telemetry) and local Elasticsearch 8.19.14 (used for OpenWebUI RAG vectors). Documents cluster configs, auth, index mappings, ELSER, and the current split between deployments.

| Key | Value |
|---|---|
| Elastic Cloud cluster | Elastic Cloud 9.4.0 (GCP us-central1, 3 nodes/2 data, UUID `DaBtVAvNQNmqT0thJwt-7Q`) |
| Elastic Cloud auth | API key (`ELASTICSEARCH_API_KEY` in `~/.env`) |
| Cloud indexes | `sheepsoc_rag001_v2` / `v3` (~48k docs) — research; OTEL data streams (added 2026-06-27) |
| Local ES | Elasticsearch 8.19.14 (`elasticsearch.service`, `/mnt/elastic_data`, `127.0.0.1:9200`) |
| Local indexes | `open_webui_collections_d768` (dual dense kNN + ELSER sparse, ~1.9 GB) — **current OpenWebUI RAG backend** |
| Updated | 2026-06-28 |

!!! note "Current State (as of 2026-06-28)"
    **Two active ES deployments.** Local ES 8.19.14 (`127.0.0.1:9200`) serves [OpenWebUI](openwebui-rag.md) RAG vectors — `open_webui_collections_d768` (~1.9 GB, HNSW/cosine + ELSER) is still on local storage. **Elastic Cloud 9.4.0** serves research indexes (RAG-001 v2/v3) and [OTEL data streams](otelcol-contrib.md) (confirmed flowing 2026-06-27). Migration of OpenWebUI RAG to Elastic Cloud is planned but has **not** been done yet. The 2026-05-10 history entry "local ES decommissioned — all traffic on cloud" referred to research/ingest workloads; the OpenWebUI RAG index was never moved. Local ES is targeted for an ES 9 rebuild once the RAG migration completes.

**See [RAG Experiments](../research/rag-experiments.md)** for full pipeline, results (Hybrid 0.6809 nDCG@10), golden dataset, notebooks, and runbooks. This page focuses on the shared ES/ELSER infrastructure.

## Dependencies

- **[RAG Experiments](../research/rag-experiments.md)** **stores data in** Elastic Cloud — primary consumer of cloud indexes for research (RAG-001/002 pipelines, evaluation).
- [OpenWebUI & RAG](openwebui-rag.md) **depends on** local ES 8.19.14 — OpenWebUI RAG vectors (`open_webui_collections_d768`) live on local ES; migration to cloud is planned.
- [OpenTelemetry Collector](otelcol-contrib.md) **stores data in** Elastic Cloud — OTEL data streams (logs/metrics/traces) export directly to the cloud cluster.
- Ollama — local embedding for ingest/query vectors (cloud cannot reach localhost:11434).

## Dependencies

- [OpenWebUI & RAG](openwebui-rag.md) — the primary consumer of the `open_webui_collections_d768` index; ELSER is layered on the same index that OpenWebUI writes to
- [Ollama](../services.md) — provides the `nomic-embed-text` embedding model used for the dense kNN half of hybrid queries

## Runbooks

- [OpenWebUI KB Bulk Ingest](../runbooks/openwebui-kb-bulk-ingest.md) — ingest script that writes to the same index this page describes
- [Shutdown & Startup](../runbooks/shutdown-startup.md) — Elasticsearch must be started before OpenWebUI and the bot; see startup sequence

## See Also

- [Knowledge Bases](knowledge-bases.md) — the collections stored in this index, with UUIDs
- [Services](../services.md) — Elasticsearch service entry, port, auth details, and cluster health commands
- [Known Issues](../known-issues.md) — history of the xpack.security enablement and beats_writer configuration
- [OpenTelemetry Collector](otelcol-contrib.md) — **stores data in** Elastic Cloud 9.4.0; OTEL data streams land in the cloud cluster

## Elastic Cloud — OTEL Data Streams (Active, 2026-06-27)

The [OpenTelemetry Collector](otelcol-contrib.md) exports OTEL data directly to Elastic Cloud 9.4.0 via HTTPS. OTEL data uses Elastic's ECS-compatible data stream naming convention.

| Data Stream Pattern | Source | Status |
|---|---|---|
| `logs-open_webui.otel-*` | OpenWebUI | Confirmed flowing |
| `metrics-open_webui.otel-*` | OpenWebUI | Confirmed flowing |
| `traces-open_webui.otel-*` | OpenWebUI | Confirmed flowing |
| `logs-claude_code.otel-*` | Claude Code | Appears on next new Claude Code session |
| `metrics-claude_code.otel-*` | Claude Code | Appears on next new Claude Code session |

Verify via Kibana (Stack Management → Index Management → Data Streams, filter "otel") or via the Elastic Cloud REST API using the `ELASTICSEARCH_API_KEY`.

!!! note "Local ES Not Involved in OTEL"
    OTEL data does **not** pass through local ES 8.19.14. The collector exports directly to the cloud endpoint. Local ES 8.19.14 only serves [OpenWebUI](openwebui-rag.md) RAG vectors (`open_webui_collections_d768`).

## Background: Dense vs. Sparse Embeddings

Sheepsoc's RAG system originally used only dense embeddings to find relevant document chunks. Understanding the difference between dense and sparse is the key to understanding why ELSER was added.

### Dense Embeddings (the `vector` field)

Every text chunk is converted to a fixed-length array of floating-point numbers — in sheepsoc's case, 768 numbers produced by `nomic-embed-text`. You can think of this array as a point in 768-dimensional space. Two chunks are semantically similar if their points are close together in that space (measured by cosine distance).

Dense embeddings excel at **semantic similarity** — they find documents that *mean* the same thing even if they use completely different words. A query about "protecting crops from frost" will match a chunk about "freeze protection for strawberry fields" even though the exact words do not overlap.

The limitation of dense embeddings is that they are trained to compress all meaning into a fixed-size vector. This compression can lose keyword-level precision, particularly for highly specific terms: proper nouns, technical jargon, model numbers, exact quantities.

### Sparse Embeddings — ELSER (the `text_elser` field)

ELSER (Elastic Learned Sparse EncodeR) takes a different approach. Instead of one long array of floats, it produces a sparse set of token-weight pairs — like a weighted vocabulary. A chunk about strawberry cultivation might generate tokens such as `strawberry`, `bees`, `pollen`, `frost`, and `fertilization` with associated weights, even if some of those words do not appear verbatim in the source text. ELSER performs learned vocabulary expansion: it understands that a document about pollinating strawberries is likely relevant to queries about "fruit set" or "bee foraging".

This is conceptually similar to BM25 (term-frequency scoring), but with a neural network doing the vocabulary expansion. ELSER excels at **keyword intent** queries — finding documents when the user's query contains specific terms that should match with high precision.

### Why Both?

Running both allows **hybrid queries** using Elasticsearch's Reciprocal Rank Fusion (RRF). RRF combines two ranked lists — one from kNN on the dense field, one from `text_expansion` on the sparse field — and merges them into a single ranked result. This is currently considered best practice for RAG recall, as it covers both semantic and keyword retrieval gaps.

## What Was Configured (2026-04-23)

| Component | Detail |
|---|---|
| New index field | `text_elser` — type `sparse_vector` |
| Ingest pipeline | `elser-embed-openwebui` — runs ELSER at index time for every new document |
| Default pipeline | `elser-embed-openwebui` set as the `default_pipeline` on `open_webui_collections_d768` |
| ELSER model | `.elser-2-elasticsearch` — Elastic's built-in sparse model; adaptive allocation (scales to zero when idle) |
| Backfill | All 8,136 existing documents in the index were backfilled with ELSER tokens via `_update_by_query` |
| How-to guide | `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md` |

!!! note "OpenWebUI Is Unchanged"
    OpenWebUI does not know about `text_elser`. Its kNN search on the `vector` field operates exactly as before. The ingest pipeline runs transparently on every document at index time: OpenWebUI writes a chunk, the pipeline adds ELSER tokens, and the document lands with both fields populated. There is no configuration change to OpenWebUI itself.

## How the Ingest Pipeline Works

When OpenWebUI uploads a document chunk to Elasticsearch, the request goes through the `elser-embed-openwebui` ingest pipeline before the document is written to the index. The pipeline runs a single processor: `inference` with the `.elser-2-elasticsearch` model and `field_map: {"text": "text_elser"}`. This reads the `text` field from the incoming chunk and writes ELSER token-weight pairs into the `text_elser` field on the same document.

The pipeline runs in-cluster — Elasticsearch handles the inference call internally. No external service needs to be called, and Ollama is not involved in ELSER tokenization.

## Query Patterns

All queries below target the same index that OpenWebUI uses. They require Elasticsearch credentials. The actual password is in `~/elastic-creds.txt`.

### Pattern 1 — Pure ELSER (`text_expansion`)

Best for keyword-intent queries. No query vector is required — ELSER runs at query time, expanding the query text into a sparse token set and matching against stored document tokens.

```bash
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    -X POST "http://localhost:9200/open_webui_collections_d768/_search" \
    -H "Content-Type: application/json" \
    -d '{
  "size": 5,
  "query": {
    "text_expansion": {
      "text_elser": {
        "model_id": ".elser-2-elasticsearch",
        "model_text": "what protects strawberry plants from frost"
      }
    }
  },
  "_source": ["text", "metadata.name", "metadata.page"]
}' | python3 -m json.tool
```

!!! note "Cold Start"
    ELSER uses adaptive allocation. If it has been idle, the first query may take 30–60 seconds while the model allocates. Subsequent queries in the same session are fast.

### Pattern 2 — Hybrid RRF (ELSER + kNN)

Best overall recall. Combines semantic similarity (kNN on `vector`) and keyword intent (ELSER on `text_elser`) using Reciprocal Rank Fusion. Requires a query embedding from Ollama for the kNN half.

```bash
# Step 1: produce the query embedding
pmabry@sheepsoc:~$ QUERY_VEC=$(curl -s "http://localhost:11434/v1/embeddings" \
    -H "Content-Type: application/json" \
    -d '{"model": "nomic-embed-text", "input": "strawberry pollination"}' \
    | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['embedding'])")

# Step 2: hybrid RRF search
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    -X POST "http://localhost:9200/open_webui_collections_d768/_search" \
    -H "Content-Type: application/json" \
    -d "{
  \"size\": 5,
  \"retriever\": {
    \"rrf\": {
      \"retrievers\": [
        {\"knn\": {\"field\": \"vector\", \"query_vector\": $QUERY_VEC, \"k\": 10, \"num_candidates\": 50}},
        {\"standard\": {\"query\": {\"text_expansion\": {\"text_elser\": {\"model_id\": \".elser-2-elasticsearch\", \"model_text\": \"strawberry pollination\"}}}}}
      ],
      \"rank_window_size\": 10,
      \"rank_constant\": 60
    }
  },
  \"_source\": [\"text\", \"metadata.name\", \"metadata.page\"]
}" | python3 -m json.tool
```

## Troubleshooting

| Symptom | Check |
|---|---|
| ELSER query returns zero results | ELSER may be cold-starting. Wait 30–60 seconds and retry. |
| Check ELSER model allocation status | `curl -u elastic:<password> http://localhost:9200/_ml/trained_models/.elser-2-elasticsearch/_stats` |
| Verify ELSER pipeline exists | `curl -u elastic:<password> http://localhost:9200/_ingest/pipeline/elser-embed-openwebui` |
| Confirm `text_elser` field exists on a document | `curl -u elastic:<password> "http://localhost:9200/open_webui_collections_d768/_search?size=1" | python3 -m json.tool` — look for `text_elser` in `_source` |
| New document does not have `text_elser` | Confirm the default pipeline is set: `curl -u elastic:<password> "http://localhost:9200/open_webui_collections_d768/_settings" | python3 -m json.tool` — look for `default_pipeline: elser-embed-openwebui` |
| How-to reference | `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md` |
