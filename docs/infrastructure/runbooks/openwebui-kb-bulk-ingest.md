# OpenWebUI KB Bulk Ingest

**Purpose:** Step-by-step SOP for bulk-ingesting pre-extracted text collections into OpenWebUI Knowledge bases.

| Key | Value |
|---|---|
| Script | `~/repositories/sheepsoc/ingest_to_openwebui.py` |
| Conda env | `openwebui` · Python 3.11 |
| Targets | Elasticsearch `open_webui_collections_d768` · SQLite `webui.db` |
| Source data | `/mnt/ssd_working/Data/Ingest_Ready/` — pre-extracted `.txt` chunk files |

## Overview

`ingest_to_openwebui.py` bulk-ingests pre-extracted `.txt` chunk files into an OpenWebUI Knowledge base. It writes to **two backends simultaneously**: Elasticsearch (chunk text and embedding vectors) and the OpenWebUI SQLite database (`webui.db`, file records and knowledge-base links). After a successful run, ingested files appear natively in the OpenWebUI Knowledge UI and are immediately queryable in chat via `#<KnowledgeBaseName>`.

### The ES + SQLite Split-Brain Architecture

OpenWebUI separates its storage responsibilities across two systems. Understanding the boundary is essential before adapting the script for a new collection.

```
  Source .txt files
         |
         v
  ingest_to_openwebui.py
    |               |
    v               v
  Elasticsearch   SQLite (webui.db)
  open_webui_     knowledge  (KB name, UUID)
  collections_    file       (one row per source file)
  d768            knowledge_file (file <-> KB join)
  · chunk text
  · 768-dim vector
  · collection = KB UUID
```

| Store | What It Holds |
|---|---|
| Elasticsearch (`open_webui_collections_d768`) | Chunk text, 768-dim embedding vectors, and a `collection` field that links each chunk to its knowledge base by UUID |
| SQLite (`/home/pmabry/infrastructure/open-webui/webui.db`) | Knowledge base names and UUIDs (`knowledge` table), one row per ingested source file (`file` table), and the `knowledge_file` join table that links files to knowledge bases |

!!! danger "Critical"
    The ES `collection` field must be set to the **bare knowledge base UUID** from the `knowledge` table in `webui.db`. Do **not** use the `file-{uuid}` format — that namespace is reserved for standalone inline file uploads only. Using it for a Knowledge base will cause OpenWebUI to silently fail to retrieve any chunks at query time.

### What the Script Does, Step by Step

1. **Pre-flight:** verifies Ollama is up and `nomic-embed-text` is loaded, confirms Elasticsearch is reachable, checks `webui.db` exists, and validates that the target knowledge base UUID is present in SQLite.
2. **Idempotency check:** derives a stable `uuid5`-based file ID from each source filename. If that ID already exists in the SQLite `file` table, the file is skipped — safe to re-run after an interrupted ingest.
3. **Chunking:** splits each source file with a 1,500-character sliding window and 150-character overlap, matching OpenWebUI's defaults.
4. **Embedding:** calls Ollama `/api/embed` (batch endpoint) to produce a 768-dim vector for every chunk in a single request per file.
5. **ES bulk indexing:** writes chunks in batches of 50 via the Elasticsearch `_bulk` API.
6. **SQLite write:** inserts one row into `file` and one into `knowledge_file` only after ES indexing succeeds with zero errors. An ES failure leaves the file unrecorded so the next run retries it cleanly.

## Prerequisites

### Conda Environment

Run this script in the `openwebui` conda environment (Python 3.11). Do **not** use the `sheepsoc` env — `openwebui` is the environment that has the matching library versions and accesses the correct path to `webui.db`.

Think of conda environments as isolated Python installations. Activating `openwebui` switches your shell to use that environment's Python and packages, completely separate from anything else on the machine.

```bash
pmabry@sheepsoc:~$ conda activate openwebui
# Your prompt will change to show the active env:
(openwebui) pmabry@sheepsoc:~$
```

!!! note "If Conda Is Not Initialized"
    If you open a new shell and `conda` is not recognized, run `source ~/infrastructure/miniconda3/etc/profile.d/conda.sh` before the activate command.

### Dependencies

| Package | Source | Notes |
|---|---|---|
| `sqlite3` | Python standard library | No install needed |
| `requests` | Already in `openwebui` env | No install needed |

No additional pip installs are required before running the script.

### Services That Must Be Running

| Service | Port | Why It Is Needed |
|---|---|---|
| Ollama (`ollama.service`) | 11434 | Provides the `nomic-embed-text` embedding model. Every chunk is sent here to generate a 768-dim vector. |
| Elasticsearch (`elasticsearch.service`) | 9200 | The vector store. Chunks and their embeddings are written here under the knowledge base UUID. |
| OpenWebUI (`open-webui.service`) | 8080 | Must be running so that `webui.db` is in a consistent state. The script opens the SQLite file directly — it does not call the OpenWebUI API. |

### Source Data Format

Source files live in `/mnt/ssd_working/Data/Ingest_Ready/`. The directory tree is organized as one subdirectory per collection, each containing:

- `*.txt` files — pre-extracted, pre-chunked text. Each file is typically ~150 words (one logical passage or page extract).
- `*.meta.json` files — sidecar provenance metadata (original filename, source URL, etc.). The ingest script ignores these; they exist for human reference.

!!! warning "Important"
    Only ingest from `Ingest_Ready/`. Do **not** point the script at `SPLIT_PDFS_1/`, `SPLIT_PDFS_2/`, or `Raw/` — those directories contain source PDFs and extraction artifacts, not the clean text files.

## SOP: Ingesting a New Dataset

Follow these steps in order when adding a brand-new collection to a new or existing knowledge base.

### Step 1 — Verify Services Are Running

```bash
# Check all three required services
pmabry@sheepsoc:~$ systemctl is-active ollama elasticsearch open-webui
active
active
active

# Confirm nomic-embed-text is loaded in Ollama
pmabry@sheepsoc:~$ ollama list | grep nomic
nomic-embed-text:latest    ...

# Confirm Elasticsearch is healthy
pmabry@sheepsoc:~$ curl -s http://localhost:9200/_cluster/health | python3 -m json.tool | grep status
    "status": "green",
```

If any service is down, start it before proceeding: `sudo systemctl start <service-name>`.

### Step 2 — Identify the Source Directory and Confirm Files Are Present

```bash
# List the available collections
pmabry@sheepsoc:~$ ls /mnt/ssd_working/Data/Ingest_Ready/

# Confirm the target collection has .txt files
pmabry@sheepsoc:~$ ls /mnt/ssd_working/Data/Ingest_Ready/victory_garden_text/*.txt | wc -l
7
```

See the [dataset reference table](#dataset-reference-table) below for file counts and suggested KB assignments for all available collections.

### Step 3 — Create or Identify the Target Knowledge Base in OpenWebUI

1. Open OpenWebUI at `http://192.168.50.100:8080` in a browser.
2. In the left sidebar, go to **Workspace → Knowledge**.
3. If creating a new Knowledge base: click **New**, type the exact name you want (e.g., `Gardening`), and click **Create**.
4. Note the name exactly as typed — OpenWebUI uses it case-sensitively in the chat `#` selector.
5. If adding to an existing KB, confirm its name in the Knowledge list.

### Step 4 — Look Up the Knowledge Base UUID from webui.db

Elasticsearch does not know KB names — it works only with UUIDs. You must retrieve the UUID that OpenWebUI assigned to the knowledge base when you created it.

```bash
pmabry@sheepsoc:~$ sqlite3 /home/pmabry/infrastructure/open-webui/webui.db \
>   "SELECT id, name FROM knowledge ORDER BY created_at DESC LIMIT 10;"
d200f010-32b0-422c-8767-9bfaa89a1ae5|Gardening
...
```

The value in the `id` column (the UUID on the left) is what you will put in the `KNOWLEDGE_ID` variable in the script. Copy it exactly.

### Step 5 — Edit the Script Configuration

Open the script in vim and update the configuration block near the top. There are three variables to change for a new collection:

```bash
pmabry@sheepsoc:~$ vim ~/repositories/sheepsoc/ingest_to_openwebui.py
```

Find and update these variables at the top of the file:

```python
# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------
SOURCE_DIRS = [
    Path("/mnt/ssd_working/Data/Ingest_Ready/victory_garden_text"),
    # Add more directories here if combining multiple collections into one KB
    # Path("/mnt/ssd_working/Data/Ingest_Ready/another_collection_text"),
]

# Must match the existing Knowledge base UUID from webui.db (step 4 above)
KNOWLEDGE_ID   = "d200f010-32b0-422c-8767-9bfaa89a1ae5"
USER_ID        = "ebbf6ecc-4181-48d8-8dc9-d78c7eaeefa8"  # pmabry's UUID in webui.db
```

!!! note "User ID"
    `USER_ID` is Phillip's account UUID inside OpenWebUI's database. It does not change between runs — you only need to update `SOURCE_DIRS` and `KNOWLEDGE_ID` when targeting a different collection or knowledge base.

Do not change `SQLITE_PATH`, `ES_URL`, `ES_INDEX`, `OLLAMA_URL`, `EMBED_MODEL`, `CHUNK_SIZE`, `CHUNK_OVERLAP`, or `ES_BATCH_SIZE` unless you have a specific reason to — these are tuned to match OpenWebUI's defaults and the existing ES index configuration.

### Step 6 — Activate the Conda Environment

```bash
pmabry@sheepsoc:~$ conda activate openwebui
# Prompt changes to confirm the active env:
(openwebui) pmabry@sheepsoc:~$
```

### Step 7 — Run the Script

```bash
(openwebui) pmabry@sheepsoc:~$ python ~/repositories/sheepsoc/ingest_to_openwebui.py
```

The script will print a pre-flight check summary, then process each file and print progress inline. On completion it prints a summary:

```
=== Pre-flight checks ===
  Ollama OK — nomic-embed-text available
  Elasticsearch OK — cluster 'sheepsoc', status green
  SQLite OK — /home/pmabry/infrastructure/open-webui/webui.db
  Source dir OK — /mnt/ssd_working/Data/Ingest_Ready/victory_garden_text  (7 .txt files)
  Knowledge base OK — 'Victory Garden' (a1b2c3d4-...)

Found 7 source .txt files across 1 directories.

[  1/  7]   [chapter_1.txt]  18,432 chars  →  13 chunks  embed 2.4s  index 0.6s
...
============================================================
INGEST COMPLETE
  Files processed : 7
  Chunks indexed  : 91
  Total time      : 67.3s (1.1 min)
  Errors          : 0

  ES total docs (collection=a1b2c3d4-...): 91
```

### Step 8 — Verify in Elasticsearch

```bash
# Total chunks across all OpenWebUI collections
pmabry@sheepsoc:~$ curl -s "http://localhost:9200/open_webui_collections_d768/_count" \
>   | python3 -m json.tool

# Chunks for your specific knowledge base (substitute the actual KB UUID)
pmabry@sheepsoc:~$ curl -s -X GET "http://localhost:9200/open_webui_collections_d768/_count" \
>   -H "Content-Type: application/json" \
>   -d '{"query": {"term": {"collection": "a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx"}}}'
{"count":91,...}
```

The count should match the "Chunks indexed" number from the script's completion summary.

### Step 9 — Verify in OpenWebUI

1. Open OpenWebUI at `http://192.168.50.100:8080`.
2. Go to **Workspace → Knowledge**.
3. Click the knowledge base you just populated.
4. Confirm that the file list shows the expected number of files. Each source `.txt` file appears as one entry.

### Step 10 — Test Retrieval in Chat

1. Start a new chat in OpenWebUI.
2. In the message input box, type `#` — a dropdown of your Knowledge bases will appear.
3. Select the knowledge base you just ingested.
4. Ask a question related to the content you ingested.
5. The model's response should draw on the retrieved text. You can inspect retrieved chunks in the source citations if OpenWebUI's citation display is enabled.

## SOP: Adding More Collections to an Existing KB

Use this path when you want to add additional source directories to a knowledge base that already has content — for example, adding a new AgriLife extension collection to the existing Gardening KB.

1. Confirm the target KB already exists in OpenWebUI (Workspace → Knowledge). Do not create a new one — you will reuse the existing UUID.
2. Identify the new source directory in `Ingest_Ready/` and confirm `.txt` files are present (Step 2 of the main SOP above).
3. Look up the existing KB UUID from `webui.db` (Step 4 above). This is already set in the script if you are adding to the current Gardening KB.
4. Edit the script and add the new directory to `SOURCE_DIRS`. Existing directories can remain in the list — already-indexed files will be skipped automatically.
5. Activate the conda env and run the script (Steps 6 and 7 above).
6. Verify in ES and in the OpenWebUI Knowledge UI (Steps 8 and 9 above) — the file count should have increased.

!!! note "Idempotent"
    The script is safe to re-run at any time. Files already in the SQLite `file` table are skipped based on a stable, filename-derived UUID (`uuid5`). Re-running after an interruption picks up exactly where it left off, and adding new source directories to an existing run is entirely safe.

## Dataset Reference Table

All pre-extracted text collections available for ingest live under `/mnt/ssd_working/Data/Ingest_Ready/`. The table below lists every collection, its approximate file count, and a suggested knowledge base assignment. Tier 1 collections are recommended to ingest first; they are topically coherent with the existing Gardening KB or cover foundational self-sufficiency topics.

| Collection Directory | Approx Files | Suggested KB | Notes |
|---|---|---|---|
| `texas_am_text` | 123 | Gardening | **ingested** — already in Gardening KB |
| `texas_am_grasses_text` | 11 | Gardening | **ingested** — already in Gardening KB |
| `master_gardener_text` | 40 | Gardening | tier 1 — ingest first |
| `victory_garden_text` | 7 | Gardening | tier 1 — ingest first |
| `bexar_county_soil_text` | 28 | Gardening | tier 1 — ingest first |
| `encyclopedia_herbs_text` | 34 | Gardening | tier 1 — ingest first |
| `herbal_medicines_text` | 73 | Gardening | tier 1 — ingest first |
| `herbal_remedies_text` | 30 | Gardening | tier 1 — ingest first |
| `handbook_herbs_text` | 90 | Gardening | tier 1 — ingest first |
| `fall_planting_guide_text` | 4 | Gardening | tier 1 — ingest first |
| `fall_vegetable_text` | 1 | Gardening | tier 1 — ingest first |
| `rangeland_drought_text` | 1 | Gardening | tier 1 — ingest first |
| `soil_ammendments_text` | 1 | Gardening | tier 1 — ingest first |
| `soil_amendments_2_text` | 1 | Gardening | tier 1 — ingest first |
| `SeppHolzer_text` | 26 | Gardening | Permaculture / food forest |
| `aquaponics_gardening_text` | 30 | Gardening or new KB | — |
| `aquaponics_food_text` | 62 | Gardening or new KB | — |
| `biodiversity_text` | 40 | Gardening or new KB | — |
| `bee_keeping_manual_text` | 29 | Gardening or new KB | — |
| `poultry_qa_text` | 1 | Homestead | — |
| `show_broilers_text` | 1 | Homestead | — |
| `chickens_text` | 35 | Homestead | — |
| `backyard_homestead_text` | 88 | Homestead | — |
| `self_sufficiency_text` | 26 | Homestead | — |
| `home_and_farm_text` | 32 | Homestead | — |
| `farm_construction_text` | 355 | Homestead | Large collection — long run time expected |
| `food_preservation_text` | 109 | Homestead | — |
| `household_cyclopedia_text` | 82 | Homestead | — |
| `encyclopedia_text` | 75 | Reference | General encyclopedia |
| `encyclopedia_chemistry_text` | 185 | Reference | — |
| `basic_chemistry_text` | 140 | Reference | — |
| `Biology_OpenStax_text` | 140 | Reference | — |
| `earth_science_text` | 101 | Reference | — |
| `anatomy_physiology_text` | 127 | Reference | — |
| `meteorology_text` | 192 | Reference | — |
| `princiapls_of_finance_text` | 64 | Reference | Note: directory name has a typo (`princiapls`) |
| `american_reformed_text` | 38 | Reference | — |
| `medical_handbook_text` | 62 | Medical / Emergency | — |
| `nursing_text` | 192 | Medical / Emergency | — |
| `no_doctor_text` | 51 | Medical / Emergency | — |
| `survival_handbook_text` | 68 | Survival / Preparedness | — |
| `150_survival_secrets_text` | 44 | Survival / Preparedness | — |
| `water_purifier_text` | 1 | Survival / Preparedness | — |
| `ranger_handbook_text` | 37 | Survival / Preparedness | — |
| `tc_ranger_text` | 1 | Survival / Preparedness | — |
| `seabee_combat_handbooK_1_text` | 49 | Survival / Preparedness | — |
| `NEETS_Mod_1_text` | 29 | Technical | Navy electronics training material |
| `architects_rules_text` | 50 | Technical | — |
| `physical_security_text` | 32 | Security / Tactics | — |
| `personal_defense_text` | 6 | Security / Tactics | — |
| `pistol_training_text` | 10 | Security / Tactics | — |
| `rifle_marksmanship_text` | 12 | Security / Tactics | — |
| `crowd_control_text` | 14 | Security / Tactics | — |
| `long_range_surveillance_text` | 39 | Security / Tactics | — |
| `FMFRP-12-80-Kill_or_Get_Killed_text` | 44 | Security / Tactics | — |
| `counter_intelligence_text` | 23 | Security / Tactics | — |
| `psyops_text` | 44 | Security / Tactics | — |
| `military_explosives_text` | 21 | Security / Tactics | — |
| `firefighting_text` | 3 | Emergency | — |
| `lyman_reloading_text` | 43 | Misc | — |
| `homemade_ammo_text` | 5 | Misc | — |
| `james_bond_text` | 48 | Misc | — |
| `mother_earth_news_text` | 19,688 | Future work | Too large for current script — see note below |

!!! warning "Mother Earth News"
    The `mother_earth_news_text` collection is 19,688 files (~38 GB of extracted text). It is too large to run through this script without significant modification — it requires a dedicated bulk script and should land in a separate Elasticsearch index. This is flagged as future work. Do not ingest with this script.

## See Also

- [OpenWebUI & RAG](../platforms/openwebui-rag.md) — service page for the primary RAG interface this runbook feeds data into
- [Elasticsearch & ELSER](../platforms/elasticsearch-elser.md) — the index (`open_webui_collections_d768`) this script writes chunk vectors to
- [Knowledge Bases](../platforms/knowledge-bases.md) — catalog of all KB UUIDs and their source directory assignments
- [Conda Guide](../platforms/conda.md) — the `openwebui` conda environment required to run this script
- [SOPs: Add Documents to Elasticsearch](../../lab-operations/sops.md#4-add-documents-to-elasticsearch) — quick reference entry in the SOPs page

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| KB shows 0 files in OpenWebUI after ingest | UUID mismatch: `KNOWLEDGE_ID` does not match, or ES `collection` field was written as `file-{uuid}` instead of the bare UUID | Run `sqlite3 webui.db "SELECT id, name FROM knowledge;"` to confirm the UUID. Check the script's `KNOWLEDGE_ID` matches exactly. Verify ES docs have `collection == KNOWLEDGE_ID` (not `file-...`). |
| Script prints `SKIP (already indexed)` for every file on re-run | Idempotency is working correctly: the files were already indexed in a previous run | This is expected. To add new content, add new source directories to `SOURCE_DIRS` or use a different source directory with new files. |
| Pre-flight exits with embedding error / Ollama unreachable | Ollama is down or `nomic-embed-text` is not loaded | `systemctl status ollama`; if down, `sudo systemctl start ollama`. Then: `ollama list | grep nomic`. If missing: `ollama pull nomic-embed-text`. |
| Pre-flight exits with ES error | Elasticsearch is not running or not reachable on `localhost:9200` | `systemctl status elasticsearch`; if down, `sudo systemctl start elasticsearch`. Then: `curl http://localhost:9200/_cluster/health`. |
| Pre-flight exits: `SQLite DB not found` | `webui.db` is not at the expected path, or open-webui.service has never run | Confirm: `ls /home/pmabry/infrastructure/open-webui/webui.db`. If missing, start open-webui so it creates the database: `sudo systemctl start open-webui`. |
| SQLite write error / database is locked | Another process has an exclusive write lock on `webui.db` | The script is designed to run while `open-webui.service` is active (concurrent reads are safe with WAL mode). If you see lock errors with the service running, wait a moment and re-run. |
| RAG returns nothing in chat after ingest | The `collection` field in ES was written incorrectly, or the KB is not selected in the chat | Type `#` in the chat input and confirm the KB appears. Verify ES has matching docs: `curl -s -X GET "http://localhost:9200/open_webui_collections_d768/_count" -H "Content-Type: application/json" -d '{"query": {"term": {"collection": "<KB-UUID>"}}}'`. If count is 0, the collection field is wrong. |
