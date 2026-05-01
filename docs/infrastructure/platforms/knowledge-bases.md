# Knowledge Bases

**Purpose:** Catalog of all OpenWebUI RAG Knowledge Base collections on sheepsoc.

| Key | Value |
|---|---|
| Interface | OpenWebUI · [192.168.50.100:8080](http://192.168.50.100:8080) · Workspace → Knowledge |
| Vector store | Elasticsearch · index `open_webui_collections_d768` |
| Status | KBs created 2026-04-24 · bulk ingestion pending · system KBs populated by nightly cron |

## Dependencies

- [OpenWebUI & RAG](openwebui-rag.md) — Knowledge Bases are created and queried through OpenWebUI; the KB catalog here maps to collections in OpenWebUI's SQLite database (`webui.db`)
- [Elasticsearch & ELSER](elasticsearch-elser.md) — all KB chunk vectors are stored in the `open_webui_collections_d768` index; a healthy ES cluster is required for KB queries to return results
- [Ollama](../services.md) — `nomic-embed-text` is used to embed every chunk at ingest time and every query at retrieval time

## Runbooks

- [OpenWebUI KB Bulk Ingest](../runbooks/openwebui-kb-bulk-ingest.md) — step-by-step SOP for populating KBs from the pre-extracted text collections in `/mnt/ssd_working/Data/Ingest_Ready/`
- [Nightly Backups](../runbooks/nightly-backups.md) — the `rag_sync` cron job that automatically manages the Sheepsoc System Docs and Sheepsoc System Config KBs

The sheepsoc RAG system organises its document collections into thematic Knowledge Bases (KBs) inside OpenWebUI. Each KB groups related source material so that chat queries can be scoped to a relevant topic domain. Content is chunked, embedded via `nomic-embed-text` (768 dimensions), and stored in Elasticsearch, where OpenWebUI retrieves it using HNSW cosine similarity search.

!!! note "Status — 2026-04-24"
    All 15 Knowledge Bases have been created in OpenWebUI. As of this date, only the two system KBs (**Sheepsoc System Docs** and **Sheepsoc System Config**) are actively populated — these are maintained by the nightly `rag_sync` cron job. Bulk ingestion of the remaining 13 thematic KBs from `/Data/Ingest_Ready/` is pending. See [OpenWebUI KB Bulk Ingest](../runbooks/openwebui-kb-bulk-ingest.md) for the procedure.

## Knowledge Base Catalog

All 15 KBs configured on sheepsoc, their internal OpenWebUI UUIDs, descriptions, and the source directories assigned to each. Source directories correspond to subdirectories under `/mnt/ssd_working/Data/Ingest_Ready/`.

| Knowledge Base | KB ID (short) | Description | Source Directories |
|---|---|---|---|
| **Medical & First Aid** | `59976efc` | Clinical nursing, emergency first aid, field medicine, and healthcare guides for when professional care is unavailable. | `nursing_text`, `medical_handbook_text`, `no_doctor_text` |
| **Herbalism & Plant Medicine** | `d630298a` | Encyclopedias and handbooks covering medicinal herbs, herbal remedies, and plant-based treatments. | `encyclopedia_herbs_text`, `handbook_herbs_text`, `herbal_medicines_text`, `herbal_remedies_text` |
| **Agriculture & Gardening** | `e690308b` | Comprehensive gardening and small-scale farming including permaculture, aquaponics, seasonal planting, homesteading, and sustainable food production. | `master_gardener_text`, `aquaponics_food_text`, `aquaponics_gardening_text`, `fall_planting_guide_text`, `fall_vegetable_text`, `victory_garden_text`, `backyard_homestead_text`, `SeppHolzer_text`, `mother_earth_news_text`, `home_and_farm_text`, `self_sufficiency_text` |
| **Livestock & Poultry** | `b4924aaf` | Raising and managing chickens, show broilers, poultry flocks, and honeybee colonies. | `chickens_text`, `poultry_qa_text`, `show_broilers_text`, `bee_keeping_manual_text` |
| **Soil & Land Management** | `448a92b4` | Soil amendments, land stewardship, rangeland management, drought resilience, and biodiversity. | `soil_ammendments_text`, `soil_amendments_2_text`, `bexar_county_soil_text`, `rangeland_drought_text`, `biodiversity_text` |
| **Texas Agriculture & Regional Resources** | `48f975c5` | Texas A&M AgriLife extension publications and Texas-specific grass and pasture management guides. | `texas_am_text`, `texas_am_grasses_text` |
| **Military Tactics & Field Craft** | `23fab10a` | U.S. military field manuals covering ranger operations, long-range surveillance, combat tactics, Seabee construction, marksmanship, and personal defense doctrine. | `ranger_handbook_text`, `tc_ranger_text`, `long_range_surveillance_text`, `seabee_combat_handbooK_1_text`, `FMFRP-12-80-Kill_or_Get_Killed_text`, `rifle_marksmanship_text`, `pistol_training_text`, `personal_defense_text` |
| **Weapons & Munitions** | `953ef1e0` | Reloading manuals, ammunition production, military explosives references, and weapons handling. | `homemade_ammo_text`, `lyman_reloading_text`, `military_explosives_text`, `james_bond_text` |
| **Security & Intelligence** | `04ba43d8` | Counter-intelligence, crowd control doctrine, physical security planning, and psychological operations. | `counter_intelligence_text`, `crowd_control_text`, `physical_security_text`, `psyops_text` |
| **Survival & Preparedness** | `ea736ffc` | Survival handbooks, food preservation, water purification, firefighting, and emergency preparedness. | `150_survival_secrets_text`, `survival_handbook_text`, `food_preservation_text`, `water_purifier_text`, `firefighting_text` |
| **Science & Reference** | `dc9e9dea` | Academic and reference texts covering biology, chemistry, earth science, meteorology, electronics, and general encyclopedic knowledge. | `Biology_OpenStax_text`, `anatomy_physiology_text`, `basic_chemistry_text`, `encyclopedia_chemistry_text`, `earth_science_text`, `meteorology_text`, `NEETS_Mod_1_text`, `encyclopedia_text`, `american_reformed_text` |
| **Construction & Engineering** | `93af47fe` | Farm construction, architectural rules, and the Household Cyclopedia covering 19th-century building trades and practical engineering. | `farm_construction_text`, `architects_rules_text`, `household_cyclopedia_text` |
| **Finance & Economics** | `a4ba1e3a` | Principles of finance, economics fundamentals, and financial literacy references. | `princiapls_of_finance_text` |
| **Sheepsoc System Docs** | `0624b7b9` | Auto-synced documentation and SOPs for the Sheepsoc server. Populated nightly by `rag_sync` cron. Do not manually ingest into this KB. | *cron-managed* |
| **Sheepsoc System Config** | `d40f321a` | Auto-synced configuration files and infrastructure settings for the Sheepsoc server. Populated nightly by `rag_sync` cron. Do not manually ingest into this KB. | *cron-managed* |

### Full UUIDs

| Knowledge Base | Full UUID |
|---|---|
| Medical & First Aid | `59976efc-1549-4c34-958c-691c28b3a358` |
| Herbalism & Plant Medicine | `d630298a-ccab-40c8-9ee2-393fecf4f756` |
| Agriculture & Gardening | `e690308b-8715-4d4d-b15b-cff521811bca` |
| Livestock & Poultry | `b4924aaf-e9e0-4473-8735-a73132707c04` |
| Soil & Land Management | `448a92b4-d3fd-473e-b491-378e73387131` |
| Texas Agriculture | `48f975c5-2f5e-4a0a-96e9-baea9a8f4e16` |
| Military Tactics | `23fab10a-ac33-4c96-a88d-17eed72363e6` |
| Weapons & Munitions | `953ef1e0-315b-4a60-b6bb-7d48f334df44` |
| Security & Intelligence | `04ba43d8-188b-4b66-8d35-225d41b0a061` |
| Survival & Preparedness | `ea736ffc-4dff-4846-a499-b8a7781209ea` |
| Science & Reference | `dc9e9dea-b145-4cd2-a366-50c3a0b21667` |
| Construction & Engineering | `93af47fe-2506-467b-a792-2f0e2e128d30` |
| Finance & Economics | `a4ba1e3a-b813-4d46-849e-363678c38105` |
| Sheepsoc System Docs | `0624b7b9-a282-45ce-8ffd-56294d8abc9f` |
| Sheepsoc System Config | `d40f321a-f081-46bd-85bc-8bad4afea88e` |

## System KBs (Auto-Managed)

Two KBs are reserved for system documentation and are managed exclusively by the nightly `rag_sync` cron job. Do not manually upload files into these KBs — the cron job performs a full sync on each run, and any manually added content may be overwritten or cause conflicts.

| KB Name | What It Contains | Managed By | Cron Schedule |
|---|---|---|---|
| **Sheepsoc System Docs** | This documentation site (HTML/MD pages), SOPs, and system guides | `rag_sync` cron | Nightly at 02:00 |
| **Sheepsoc System Config** | Infrastructure config files, service drop-ins, and settings from the config-mgmt repo | `rag_sync` cron | Nightly at 02:00 |

!!! warning "Do Not Manually Ingest"
    These two KBs are the only ones on the system with automated content management. Adding files through the OpenWebUI Workspace UI will not cause an immediate error, but the nightly sync may replace or duplicate content unpredictably. Treat them as read-only from the UI perspective.

For details on how the nightly sync works, what it backs up, and how to verify it ran successfully, see [Nightly Backups](../runbooks/nightly-backups.md) and [GitHub & RAG Sync](../backup-and-recovery/github-rag-sync.md).

## Using a KB in Chat

Any Knowledge Base can be selected as the context source for an OpenWebUI chat session. When a KB is active, OpenWebUI retrieves the most relevant document chunks from Elasticsearch and injects them into the prompt before the model generates its response.

1. Open OpenWebUI at `http://192.168.50.100:8080`.
2. Start or open a chat.
3. In the message input box, type `#`. A dropdown of your available Knowledge Bases appears.
4. Select the KB you want to query against (e.g., **Agriculture & Gardening**).
5. Type your question and submit. The model's response draws on retrieved chunks from that KB.

!!! note "Tip"
    You can reference multiple KBs in the same message by typing `#` more than once and selecting different collections. For example, combining **Soil & Land Management** and **Texas Agriculture & Regional Resources** gives regionally specific soil query context.

!!! note "Model Selection"
    Select the active language model from the dropdown at the top of any chat. `gemma3:12b` gives the best answer quality for most RAG queries. `dolphin3:latest` is faster for quick lookups. See the [models table on the RAG & Knowledge page](openwebui-rag.md#ollama-models-available) for the full list.

## Ingesting Content into a KB

Source files for the 13 thematic KBs live in pre-extracted text collections under `/mnt/ssd_working/Data/Ingest_Ready/`. Each subdirectory listed in the catalog table above corresponds to a directory at that path containing `.txt` chunk files ready for ingestion.

### Using the OpenWebUI KB Bulk Ingest Script

The `ingest_to_openwebui.py` script handles bulk ingestion. It writes chunk vectors to Elasticsearch and registers file records in OpenWebUI's SQLite database (`webui.db`) simultaneously, so ingested files appear natively in the OpenWebUI Knowledge UI after a successful run.

```bash
# Example: ingest the Agriculture & Gardening KB
pmabry@sheepsoc:~$ conda activate openwebui
(openwebui) pmabry@sheepsoc:~$ python ~/repositories/sheepsoc/ingest_to_openwebui.py
```

!!! note "Idempotent"
    The ingest tool is idempotent. Files already registered in SQLite are skipped on re-runs, so it is safe to re-run after a partial failure or to add new source directories to an existing KB.

For the full step-by-step procedure, architecture notes, complete dataset reference (all 63+ source collections with file counts), and troubleshooting, see the dedicated **[OpenWebUI KB Bulk Ingest](../runbooks/openwebui-kb-bulk-ingest.md)** page.

### Verifying KB Population in Elasticsearch

```bash
# Count total documents in the RAG index
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    "http://localhost:9200/open_webui_collections_d768/_count" \
    | python3 -m json.tool

# Browse Kibana for a visual view of the index
# http://192.168.50.100:5601
```
