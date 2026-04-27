# RAG & Knowledge

**Purpose:** Retrieval-augmented generation on sheepsoc — how the stack works and how to use it.

| Key | Value |
|---|---|
| Interface | OpenWebUI · [192.168.50.100:8080](http://192.168.50.100:8080) |
| Embedding | `nomic-embed-text:latest` via Ollama · 768 dimensions |
| Vector store | Elasticsearch · index `open_webui_collections_d768` · HNSW/cosine · dual-use (dense + ELSER sparse, added 2026-04-23) |
| Updated | 2026-04-23 |

!!! note "Primary Interface"
    **OpenWebUI** is the only supported RAG interface on sheepsoc. All document ingestion, Knowledge base management, and AI-assisted querying is done through OpenWebUI at `:8080`. A legacy CLI RAG prototype exists at `~/repositories/sheepsoc` but is no longer maintained — see the [Legacy / Deprecated section](#legacy-cli-rag-prototype-deprecated) at the bottom of this page.

## RAG Stack Overview

Sheepsoc's RAG system answers questions from your own document collections. A query goes through four stages: the question is embedded into a vector, Elasticsearch retrieves the most relevant document chunks, the chunks are injected into the prompt as context, and an Ollama LLM generates the answer. OpenWebUI orchestrates all of this through a browser UI.

| Component | Technology | Role |
|---|---|---|
| AI interface | OpenWebUI 0.6.12 · port 8080 | Primary user-facing frontend for chat and RAG |
| LLM inference | Ollama · port 11434 · RTX 5060 Ti | Runs language models and the embedding model |
| Embedding model | `nomic-embed-text:latest` · 768d | Converts text to vectors for semantic search |
| Vector store | Elasticsearch 8.19.14 · port 9200 | Stores chunk vectors and retrieves relevant context |
| ES index | `open_webui_collections_d768` | HNSW/cosine similarity index for OpenWebUI collections · dual-use: `vector` (dense kNN) + `text_elser` (ELSER sparse, added 2026-04-23) |
| Conda env | `openwebui` · Python 3.11 | The environment that runs the OpenWebUI service |

## Uploading Documents for RAG

OpenWebUI supports two document upload modes: inline (per-conversation) and persistent Knowledge bases. Use Knowledge bases for any collection you want to query repeatedly.

### Inline Upload (Per-Conversation)

1. Open OpenWebUI at `http://192.168.50.100:8080`.
2. Start a new chat.
3. Click the **paperclip / attachment icon** in the chat input bar.
4. Upload a file (PDF, TXT, DOCX, etc.).
5. The document is embedded and used as context for that conversation only — it does not persist to a Knowledge base.

### Persistent Knowledge Bases

1. Open OpenWebUI at `http://192.168.50.100:8080`.
2. In the left sidebar, go to **Workspace → Knowledge**.
3. Click **New**, type a name (e.g., `Gardening`), and click **Create**.
4. Upload files into the Knowledge base. OpenWebUI will chunk each file, embed the chunks via `nomic-embed-text`, and store the vectors permanently in Elasticsearch.
5. The collection is now queryable in any chat via `#<KnowledgeBaseName>`.

!!! note "Bulk Ingest"
    For ingesting large pre-extracted text collections (e.g., the 63+ collections in `/mnt/ssd_working/Data/Ingest_Ready/`), use the bulk ingest script rather than the OpenWebUI UI. The dedicated script writes directly to both Elasticsearch and SQLite and is far more efficient for large datasets. See **[Bulk Ingest](bulk-ingest.md)**.

## Querying with RAG Context

Type your question in any OpenWebUI chat. If you have uploaded a document or selected a Knowledge collection, OpenWebUI automatically retrieves relevant chunks and injects them into the prompt as context. The model then answers from that context.

### Using a Knowledge Base in Chat

1. Start or open a chat.
2. In the message input box, type `#`. A dropdown of your Knowledge bases appears.
3. Select the collection you want to query against.
4. Type your question and submit. The model's response draws on the retrieved document chunks.

!!! note "Model Selection"
    Select the active language model from the dropdown at the top of any chat. All models pulled to Ollama are available. `gemma3:12b` generally gives the best answer quality; `dolphin3:latest` is faster for quick queries. See the models table below.

## Ollama Models Available

All models pulled to Ollama are available for selection in the OpenWebUI chat UI. Switch models using the model selector dropdown at the top of any chat.

| Model | Notes |
|---|---|
| `dolphin3:latest` | Fast, good general answers · good default for quick queries |
| **`gemma3:12b`** | **Best quality/speed tradeoff for RAG** |
| `mistral:7b` | Lightweight option |
| `dolphin-llama3:latest` | Alternative Dolphin variant |
| `llama2:13b` | Older but known-good fallback |
| `gemma-longctx:latest` | Extended-context variant — useful when retrieving many large chunks |
| `nemotron-mini:latest` | Small / fast experiments |
| `wizard-vicuna-uncensored:13b` | Unfiltered alternative |
| `nomic-embed-text:latest` | Embedding model only — not for chat; used internally by OpenWebUI RAG |

List what is actually on disk at any time:

```bash
pmabry@sheepsoc:~$ ollama list
```

!!! warning "Tip"
    `ollama pull <model>` downloads a new model — typically several GB. Confirm before pulling on a slow connection or during production time.

## Configuration

OpenWebUI is configured for Elasticsearch via a systemd drop-in at `/etc/systemd/system/open-webui.service.d/elasticsearch.conf`. This file injects environment variables into the service at startup, switching the default Chroma backend to Elasticsearch.

```ini
[Service]
Environment="VECTOR_DB=elasticsearch"
Environment="ELASTICSEARCH_URL=http://127.0.0.1:9200"
Environment="ELASTICSEARCH_USERNAME=elastic"
Environment="ELASTICSEARCH_PASSWORD=<elastic-password>"
Environment="RAG_EMBEDDING_ENGINE=ollama"
Environment="RAG_EMBEDDING_MODEL=nomic-embed-text"
Environment="ELASTICSEARCH_INDEX_PREFIX=open_webui_collections"
```

!!! note "Auth"
    `xpack.security` was enabled on 2026-04-21. The drop-in must include `ELASTICSEARCH_USERNAME` and `ELASTICSEARCH_PASSWORD` for OpenWebUI to authenticate. If these variables are missing, OpenWebUI will fail to connect to Elasticsearch and RAG uploads will silently fail.

| Key | Value |
|---|---|
| Index name | `open_webui_collections_d768` |
| Embedding model | `nomic-embed-text:latest` via Ollama · 768 dimensions |
| Similarity | HNSW / cosine |
| Configured | 2026-04-20 |

!!! warning "Rebuild Note"
    If the `openwebui` conda env is ever rebuilt, reinstall the Elasticsearch client manually: `pip install elasticsearch==8.19.3` (inside the activated env). It is not bundled with OpenWebUI and is easy to miss. Without it, `open-webui.service` fails to start with an `ImportError`.

## Bulk Ingest

The script at `~/repositories/sheepsoc/ingest_to_openwebui.py` bulk-ingests pre-extracted `.txt` chunk files into an OpenWebUI Knowledge base. It writes to both Elasticsearch (chunk vectors) and SQLite (`webui.db`, file and knowledge-base link records) simultaneously. After a successful run, ingested files appear natively in the OpenWebUI Knowledge UI and are queryable via `#<KnowledgeBaseName>` in chat.

The script is idempotent — files already indexed in SQLite are skipped on re-runs. Source data lives in `/mnt/ssd_working/Data/Ingest_Ready/` (63+ pre-extracted text collections covering gardening, homesteading, medicine, survival, and more).

!!! note "Dedicated Page"
    Full step-by-step SOPs, architecture notes, dataset reference table (all 63+ collections with file counts), and troubleshooting guide live on the dedicated page: **[Bulk Ingest](bulk-ingest.md)**

## ELSER Sparse Embeddings — Dual-Use Index

As of 2026-04-23, the `open_webui_collections_d768` index is dual-use. OpenWebUI's behaviour is completely unchanged — it still performs kNN search on the `vector` field as it always has. The index now also carries a second field, `text_elser`, populated by ELSER (Elastic Learned Sparse EncodeR), Elastic's built-in sparse embedding model. This field enables two additional query patterns against the same documents from outside OpenWebUI.

### How It Works

Dense embeddings (the `vector` field) represent each text chunk as a point in 768-dimensional space. Finding relevant chunks means computing cosine distance between your query vector and all stored vectors. This is excellent at catching *semantic similarity* — documents that mean the same thing even if they use different words.

ELSER works differently. Instead of a dense float array, it produces a sparse set of token-weight pairs — think of it as BM25 with learned vocabulary expansion. A chunk about strawberry cultivation might generate tokens like `strawberry`, `bees`, `pollen`, and `fertilization` even if those exact words do not all appear verbatim in the source text. This is excellent at catching *keyword intent* where dense kNN may under-perform.

Running both allows hybrid queries (using Elasticsearch's RRF — Reciprocal Rank Fusion) that combine the strengths of both approaches, which is currently considered best practice for recall in RAG pipelines.

### Configuration Added 2026-04-23

| Key | Value |
|---|---|
| New field | `text_elser` — type `sparse_vector` |
| Ingest pipeline | `elser-embed-openwebui` — set as the index's `default_pipeline` |
| ELSER model | `.elser-2-elasticsearch` — built-in; adaptive allocation (scales to zero when idle) |
| Coverage | 8,136 / 8,136 existing documents backfilled · all future uploads auto-processed at ingest |
| How-to guide | `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md` |

!!! note "OpenWebUI Unchanged"
    OpenWebUI does not know about or use `text_elser`. Its kNN search on `vector` is unaffected. The pipeline runs transparently on every document at index time — OpenWebUI writes a chunk, the pipeline adds ELSER tokens, and the document lands with both fields populated. Zero configuration change to OpenWebUI itself.

### Query Patterns

All queries below run against the same index that OpenWebUI uses. They require ES credentials (`-u elastic:<password>`). The actual password is in `~/elastic-creds.txt`.

#### Pattern 1 — Pure ELSER (`text_expansion`)

Best for keyword-intent queries. No query vector required — ELSER runs at query time.

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

#### Pattern 2 — Hybrid RRF (ELSER + kNN)

Best overall recall. Combines semantic similarity (kNN on `vector`) and keyword intent (ELSER on `text_elser`) via Reciprocal Rank Fusion.

```bash
# Step 1: compute query embedding with Ollama
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
| OpenWebUI won't start | `journalctl -u open-webui.service -n 50` — look for `ImportError` on `elasticsearch`. If the `openwebui` conda env was rebuilt, reinstall: `pip install elasticsearch==8.19.3`. |
| Uploads fail silently | Confirm Ollama is up (`systemctl status ollama`) and `nomic-embed-text` is present (`ollama list`). |
| RAG returns nothing | Check ES is up: `curl -u elastic:<password> http://localhost:9200/_cluster/health`. Confirm the Knowledge base was selected in chat (type `#`). |
| Check how many docs are in the RAG index | `curl -s -u elastic:<password> "http://localhost:9200/open_webui_collections_d768/_count" \| python3 -m json.tool` |
| Roll back to Chroma | Delete the drop-in: `sudo rm /etc/systemd/system/open-webui.service.d/elasticsearch.conf`, then `sudo systemctl daemon-reload && sudo systemctl restart open-webui`. |
| ELSER query returns zero results | ELSER may be cold-starting. Wait 30–60 seconds and retry. Check model status: `curl -u elastic:<password> http://localhost:9200/_ml/trained_models/.elser-2-elasticsearch/_stats`. |
| ES auth errors after restart | Confirm the drop-in at `/etc/systemd/system/open-webui.service.d/elasticsearch.conf` still contains `ELASTICSEARCH_USERNAME` and `ELASTICSEARCH_PASSWORD`. Security was enabled 2026-04-21. |

## Legacy: CLI RAG Prototype (Deprecated)

!!! danger "Deprecated"
    The CLI RAG application described in this section (`sheepsoc.py` / `~/repositories/sheepsoc`) is no longer maintained and is not used for production RAG on sheepsoc. It was a command-line prototype built on a MacBook during initial development. **Use OpenWebUI for all RAG interactions.** This section is preserved for historical reference only.

The legacy CLI RAG prototype was a Python command-line application that answered questions from a document collection using a hybrid KNN + BM25 search pipeline with CrossEncoder reranking. It used HuggingFace `intfloat/e5-large-v2` embeddings and ran in the `sheepsoc` conda environment (Python 3.12).

### Legacy Stack

| Component | Technology |
|---|---|
| LLM inference | Ollama (local, port 11434) |
| Vector store | Elasticsearch |
| Embeddings | HuggingFace `intfloat/e5-large-v2` (1024-dim) |
| Reranker | CrossEncoder (`sentence-transformers`) |
| Framework | LangChain |
| Language | Python 3.12 (conda env `sheepsoc`) |

### Legacy ES Index

| Key | Value |
|---|---|
| Index name | `intfloat_e5_large_v2__512__125` |
| Embedding | `intfloat/e5-large-v2` · 1024-dim |
| Chunking | 512 tokens with 125-token overlap |
| Search | Dense KNN + BM25, fused with Reciprocal Rank Fusion |

### Legacy Project Structure

```
~/repositories/sheepsoc/
├── sheepsoc/
│   ├── cli.py             # entry point — the interactive loop
│   ├── pipeline.py        # orchestrates one full query turn
│   ├── config.py          # settings loaded from .env
│   ├── prompt_builder.py  # assembles the final prompt
│   ├── fetcher.py         # document fetching + token budgeting
│   ├── utils.py           # token counting, truncation, cleaning
│   ├── logger.py          # logging setup
│   ├── clients/
│   │   ├── es_client.py   # Elasticsearch + embedding store
│   │   └── llm_client.py  # Ollama wrapper (ChatOllama)
│   └── retrieval/
│       ├── retriever.py   # vector + hybrid search
│       └── reranker.py    # CrossEncoder reranking
├── prompts/               # prompt templates
├── tests/                 # query tests + old ingestion scripts
├── ingest_to_openwebui.py # current bulk-ingest script (still used)
├── start_sheepsoc.sh      # legacy launch script (not used)
└── .env                   # legacy config (not in git)
```

!!! note
    `ingest_to_openwebui.py` lives in the same directory as the legacy CLI app but is an entirely separate, actively maintained script for bulk-ingesting data into OpenWebUI Knowledge bases. Do not confuse it with the legacy CLI prototype. See [Bulk Ingest](bulk-ingest.md).
