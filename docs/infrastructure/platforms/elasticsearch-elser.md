# ELSER & OpenWebUI

**Purpose:** How ELSER sparse-vector search is layered onto the OpenWebUI RAG pipeline, what it adds, and how to query against it.

| Key | Value |
|---|---|
| Feature | Dual-use index: dense kNN (`vector`) + ELSER sparse (`text_elser`) |
| Configured | 2026-04-23 |
| Index | `open_webui_collections_d768` |
| ELSER model | `.elser-2-elasticsearch` (built-in, adaptive allocation) |
| OpenWebUI | Unchanged — continues to use kNN on `vector` field only |

!!! note "What This Page Covers"
    This page explains the ELSER architecture added to the sheepsoc RAG stack on 2026-04-23. It covers the conceptual difference between dense and sparse embeddings, what was configured, how the dual-use index works, and how to run ELSER and hybrid queries manually. OpenWebUI's behaviour is completely unchanged by this work — the changes are additive and transparent.

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
