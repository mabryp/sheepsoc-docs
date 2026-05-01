---

# SheepSOC-RAG-001
## Minimum Viable Experimental Protocol: Retrieval Method Comparative Study
### Version 1.0 — For Local Execution on SheepSOC Hardware

---

## 0. Protocol Header

| Field | Value |
|---|---|
| **Study ID** | SheepSOC-RAG-001 |
| **Hypothesis** | Retrieval method quality (BM25 → Dense → ELSER v2 → Hybrid) positively and monotonically correlates with generation faithfulness, and this effect is larger for Mistral 7B than Gemma 3 12B |
| **Secondary hypothesis** | Hybrid retrieval will show the largest gains specifically on conceptual query types; BM25 will be competitive on entity/exact-match query types |
| **Models** | Mistral 7B (via Ollama), Gemma 3 12B (via Ollama) |
| **Retrieval conditions** | No-RAG baseline, BM25, Dense (nomic-embed-text), ELSER v2, Hybrid (BM25 + ELSER v2, RRF) |
| **Design** | 2 models × 5 conditions × 100 questions = 1,000 model evaluations |
| **Local-only constraint** | All LLM inference, embedding, and scoring via local Ollama + Elasticsearch + HHEM |
| **Estimated runtime** | ~8–14 hours total (inference-dominated) |
| **Primary output** | Faithfulness and Precision@5 per retrieval condition, disaggregated by query type and model |

---

## 1. Corpus Construction

### 1.1 Corpus Specification

The corpus must be large enough that retrieval is non-trivial (every question cannot trivially match) but small enough to index and iterate on within hours.

| Property | Specification |
|---|---|
| **Target corpus size** | 1,000–2,000 documents |
| **Chunk size** | 400 tokens, 20% overlap (80 tokens) |
| **Expected chunks** | ~4,000–10,000 chunks after splitting |
| **Language** | English only (ELSER v2 is English-optimized) |
| **Format** | Plain text after parsing |

### 1.2 Recommended Corpus Sources (Pick One)

**Option A — Public benchmark (enables external comparison):**
Download the TREC-COVID or SciFact corpus from the BEIR benchmark suite. Both have official relevance judgments you can use to validate your golden dataset.
```bash
# Download SciFact from HuggingFace datasets
pip install datasets
python -c "from datasets import load_dataset; ds = load_dataset('BeIR/scifact', 'corpus'); ds['corpus'].to_json('corpus.jsonl')"
```

**Option B — Domain corpus you care about:**
Use any internal document set (manuals, policies, technical docs). The advantage: results are directly meaningful for your actual use case. The disadvantage: you must hand-label all relevance judgments.

**Option C — Hybrid (recommended for first run):**
Use a curated Wikipedia dump on a single domain (e.g., all Wikipedia articles on computer networking, or all articles in a scientific domain). Tools like `wikipedia-api` make this scriptable.

### 1.3 Corpus Pre-Processing Pipeline

```python
# corpus_prep.py — run once before study begins

import json
from pathlib import Path
from typing import Iterator

def chunk_text(text: str, chunk_size: int = 400, overlap: int = 80) -> list[dict]:
    """
    Token-approximate chunking with overlap.
    Uses word-count as proxy for token count (multiply by ~1.3 for actual tokens).
    For nomic-embed-text and ELSER v2, 400 words ≈ 512 tokens.
    """
    words = text.split()
    chunks = []
    step = chunk_size - overlap
    for i, start in enumerate(range(0, len(words), step)):
        chunk_words = words[start : start + chunk_size]
        if len(chunk_words) < 50:  # drop tiny trailing chunks
            break
        chunks.append({
            "chunk_id": f"chunk_{i:05d}",
            "text": " ".join(chunk_words),
            "word_count": len(chunk_words),
            "chunk_index": i,
            "start_word": start
        })
    return chunks

def process_corpus(input_jsonl: str, output_dir: str):
    output = []
    with open(input_jsonl) as f:
        for line in f:
            doc = json.loads(line)
            doc_id = doc["_id"]
            title = doc.get("title", "")
            body = doc.get("text", "")
            full_text = f"{title}. {body}".strip()
            
            for chunk in chunk_text(full_text):
                output.append({
                    "doc_id": doc_id,
                    "chunk_id": f"{doc_id}_{chunk['chunk_id']}",
                    "title": title,
                    "text": chunk["text"],
                    "word_count": chunk["word_count"],
                    "chunk_index": chunk["chunk_index"],
                    "source": input_jsonl
                })
    
    Path(output_dir).mkdir(exist_ok=True)
    with open(f"{output_dir}/chunks.jsonl", "w") as f:
        for item in output:
            f.write(json.dumps(item) + "\n")
    
    print(f"Processed {len(output)} chunks from {input_jsonl}")
```

**Controls on corpus prep:**
- Same chunk boundaries for ALL four retrieval indices (BM25, Dense, ELSER, Hybrid). Do not re-chunk for different methods.
- Log chunk count, mean word count, and vocabulary size before indexing.
- Deduplicate on `(doc_id, chunk_index)` before indexing.

---

## 2. Elasticsearch Index Configuration

### 2.1 Index Mapping (Single Index, All Methods)

Use **one index** that supports all four retrieval methods simultaneously. This guarantees identical document storage across conditions.

```json
PUT /sheepsoc-rag-001
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "english_standard": {
          "type": "english"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "doc_id":        { "type": "keyword" },
      "chunk_id":      { "type": "keyword" },
      "title":         { "type": "text", "analyzer": "english_standard" },
      "text": {
        "type": "text",
        "analyzer": "english_standard",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "word_count":    { "type": "integer" },
      "chunk_index":   { "type": "integer" },
      "dense_vector": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      },
      "elser_sparse": {
        "type": "sparse_vector"
      }
    }
  }
}
```

### 2.2 Indexing Pipeline

```python
# indexing.py

from elasticsearch import Elasticsearch
import ollama
import json

es = Elasticsearch("http://localhost:9200")
INDEX = "sheepsoc-rag-001"

def get_nomic_embedding(text: str) -> list[float]:
    """Get 768-dim embedding from nomic-embed-text via Ollama."""
    response = ollama.embeddings(model="nomic-embed-text", prompt=text)
    return response["embedding"]

def index_chunk(chunk: dict):
    dense = get_nomic_embedding(chunk["text"])
    # ELSER sparse vector is handled by Elasticsearch ML inference pipeline
    # (configure ELSER v2 inference endpoint separately)
    doc = {
        "doc_id": chunk["doc_id"],
        "chunk_id": chunk["chunk_id"],
        "title": chunk["title"],
        "text": chunk["text"],
        "word_count": chunk["word_count"],
        "chunk_index": chunk["chunk_index"],
        "dense_vector": dense
        # elser_sparse is populated via ingest pipeline — see ELSER setup docs
    }
    es.index(index=INDEX, id=chunk["chunk_id"], body=doc)

# Run indexing with progress logging
with open("corpus/chunks.jsonl") as f:
    for i, line in enumerate(f):
        chunk = json.loads(line)
        index_chunk(chunk)
        if i % 100 == 0:
            print(f"Indexed {i} chunks...")

es.indices.refresh(index=INDEX)
print("Indexing complete. Running verification...")
count = es.count(index=INDEX)["count"]
print(f"Total indexed chunks: {count}")
```

**Verification checks before running the study:**
```python
# Verify all four retrieval methods return results
test_query = "what is the mechanism of action"

# BM25
r1 = es.search(index=INDEX, body={"query": {"match": {"text": test_query}}, "size": 5})
assert r1["hits"]["total"]["value"] > 0, "BM25 failed"

# Dense
emb = get_nomic_embedding(test_query)
r2 = es.search(index=INDEX, body={"knn": {"field": "dense_vector", "query_vector": emb, "k": 5, "num_candidates": 50}})
assert r2["hits"]["total"]["value"] > 0, "Dense failed"

print("All retrieval methods verified. Proceeding to study.")
```

---

## 3. Golden Dataset Schema

### 3.1 Question Record Schema

```json
{
  "$schema": "sheepsoc-rag-001-question-v1",
  "question_id": "Q0042",
  "question_text": "What cellular mechanism allows mRNA vaccines to produce an immune response?",
  "query_type": "conceptual",
  "difficulty": "medium",
  "reference_answer": "mRNA vaccines deliver messenger RNA into cells, which ribosomes use to produce a viral antigen protein. The immune system detects this foreign protein and generates antibodies and memory cells against it, without the patient ever being exposed to the actual pathogen.",
  "reference_answer_source_chunks": ["doc_00312_chunk_00003", "doc_00312_chunk_00004"],
  "reference_doc_ids": ["doc_00312"],
  "answer_requires_multi_hop": false,
  "expected_hard_for": ["BM25"],
  "notes": "BM25 likely misses this because the query uses 'cellular mechanism' and 'immune response' while the answer chunk uses 'ribosome translation' and 'antibody production'"
}
```

### 3.2 Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `question_id` | string | ✅ | Unique ID, format Q0001–Q0100 |
| `question_text` | string | ✅ | The exact question sent to the retrieval system |
| `query_type` | enum | ✅ | See Section 3.3 |
| `difficulty` | enum | ✅ | `easy`, `medium`, `hard` — see rubric |
| `reference_answer` | string | ✅ | Gold standard answer, written from source documents |
| `reference_answer_source_chunks` | list[string] | ✅ | Exact chunk IDs that contain the answer |
| `reference_doc_ids` | list[string] | ✅ | Parent document IDs |
| `answer_requires_multi_hop` | boolean | ✅ | True if answer requires merging info from 2+ chunks |
| `expected_hard_for` | list[string] | ✅ | Which retrieval methods are expected to struggle (hypothesis annotation) |
| `notes` | string | ❌ | Author notes on why this question is interesting |

### 3.3 Query Type Taxonomy and Distribution

**Distribute 100 questions across three types:**

| Type | Count | Definition | Example | Expected Winner |
|---|---|---|---|---|
| **Exact / Entity** | 30 | Query contains a specific named entity, number, acronym, or technical term that must match literally | "What is the half-life of Compound X?" | BM25 |
| **Conceptual / Semantic** | 40 | Query uses different vocabulary than the answer document; meaning must be inferred | "How does the body fight viral infection?" (answer uses "T-cell response") | Dense / ELSER v2 |
| **Multi-hop** | 30 | Answer requires combining information from 2+ separate chunks or documents | "Given the side effects of Drug A and the contraindications of Drug B, which is safer for patients with kidney disease?" | Hybrid |

**Difficulty distribution (within each type):**
- Easy: 30% (answer is in one chunk, answer vocabulary overlaps with query)
- Medium: 50% (answer requires some inference or is in a single chunk but with vocabulary mismatch)
- Hard: 20% (answer requires multi-hop reasoning or is buried in a long document)

### 3.4 Golden Dataset Construction Procedure

This is the most time-intensive part of the study. Budget 4–6 hours.

**Step 1:** Sample candidate questions from the corpus.
```python
# Use RAGAS synthetic test set generation to bootstrap questions,
# then manually review and edit each one.

from ragas.testset.generator import TestsetGenerator
from ragas.testset.evolutions import simple, reasoning, multi_context
from langchain_community.llms import Ollama

generator_llm = Ollama(model="gemma3:12b")  # use your local model

generator = TestsetGenerator.from_langchain(
    generator_llm=generator_llm,
    critic_llm=generator_llm,
    embeddings=None  # use your nomic embeddings here
)

# Generate 200 candidate questions (you will keep 100 after review)
testset = generator.generate_with_llamaindex_docs(
    documents,
    test_size=200,
    distributions={simple: 0.40, reasoning: 0.30, multi_context: 0.30}
)
```

**Step 2:** For each generated question, manually:
1. Locate the exact source chunk(s) in Elasticsearch
2. Verify the reference answer is fully supported by those chunks
3. Assign `query_type`, `difficulty`, and `expected_hard_for`
4. Rewrite the reference answer in your own words from the source material
5. Discard any question where you cannot locate a definitive source chunk

**Step 3:** Final quality checks:
- Exactly 30 exact/entity, 40 conceptual, 30 multi-hop questions
- No two questions share the same primary source chunk
- Every reference answer is verifiable against a specific chunk ID
- At least 10 "easy" questions serve as sanity checks (all methods should pass)

---

## 4. Relevance Labeling Rubric

Applied to retrieved chunks, not to model outputs. Used to compute retrieval metrics (Precision@K, Recall@K, nDCG, MRR).

### 4.1 Graded Relevance Scale (0–3)

| Grade | Label | Definition | Example |
|---|---|---|---|
| **3** | Highly Relevant | This chunk contains the direct answer or a core factual claim needed to answer the question. The model could answer correctly from this chunk alone. | Query: "What is the boiling point of ethanol?" → Chunk: "Ethanol boils at 78.37°C at standard pressure." |
| **2** | Relevant | This chunk contains useful supporting information. It contributes to a complete answer but alone is insufficient. | Query: "What is the boiling point of ethanol?" → Chunk: "Ethanol is a volatile organic compound commonly used as a solvent." |
| **1** | Marginally Relevant | This chunk is on-topic but does not directly help answer the question. It provides background context, not the answer. | Query: "What is the boiling point of ethanol?" → Chunk: "Alcoholic beverages contain ethanol produced by fermentation." |
| **0** | Not Relevant | This chunk does not help answer the question. It may be from an unrelated document or on a different topic entirely. | Query: "What is the boiling point of ethanol?" → Chunk: "Methane is a greenhouse gas." |

### 4.2 Labeling Rules

- **Label in context:** Always read the question before reading the chunk.
- **Grade what's there, not what could be inferred:** If the chunk doesn't say it, grade it 0 or 1.
- **Multi-hop questions:** A chunk can earn grade 3 for providing one of two required facts.
- **Redundant chunks:** If two chunks from the same document have the same content, label both the same.
- **Ambiguous cases:** Default to the lower grade and add a note.

### 4.3 Computing Metrics from Graded Relevance

```python
import numpy as np

def ndcg_at_k(retrieved_grades: list[int], k: int) -> float:
    """
    retrieved_grades: list of relevance grades (0-3) in retrieval rank order
    k: cutoff rank
    """
    gains = retrieved_grades[:k]
    dcg = sum((2**g - 1) / np.log2(i + 2) for i, g in enumerate(gains))
    ideal = sorted(retrieved_grades, reverse=True)[:k]
    idcg = sum((2**g - 1) / np.log2(i + 2) for i, g in enumerate(ideal))
    return dcg / idcg if idcg > 0 else 0.0

def precision_at_k(retrieved_grades: list[int], k: int, threshold: int = 2) -> float:
    """Fraction of top-k chunks with grade >= threshold."""
    return sum(1 for g in retrieved_grades[:k] if g >= threshold) / k

def recall_at_k(retrieved_grades: list[int], k: int,
                all_relevant_count: int, threshold: int = 2) -> float:
    """Fraction of all relevant chunks found in top-k."""
    found = sum(1 for g in retrieved_grades[:k] if g >= threshold)
    return found / all_relevant_count if all_relevant_count > 0 else 0.0

def mrr(retrieved_grades: list[int], threshold: int = 2) -> float:
    """Reciprocal rank of first relevant result."""
    for i, g in enumerate(retrieved_grades):
        if g >= threshold:
            return 1.0 / (i + 1)
    return 0.0
```

---

## 5. Retrieval Query Configurations

Four exact Elasticsearch query bodies. All use `size=5` (K=5) throughout the study.

### 5.1 BM25

```python
def retrieve_bm25(question: str, k: int = 5) -> list[dict]:
    body = {
        "query": {
            "multi_match": {
                "query": question,
                "fields": ["title^2", "text"],
                "type": "best_fields",
                "analyzer": "english_standard"
            }
        },
        "size": k,
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "method": "bm25"} for h in hits]
```

### 5.2 Dense Vector (nomic-embed-text)

```python
def retrieve_dense(question: str, k: int = 5) -> list[dict]:
    query_vector = get_nomic_embedding(question)
    body = {
        "knn": {
            "field": "dense_vector",
            "query_vector": query_vector,
            "k": k,
            "num_candidates": k * 10  # search 50 candidates for top-5
        },
        "size": k,
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "method": "dense"} for h in hits]
```

### 5.3 ELSER v2

```python
def retrieve_elser(question: str, k: int = 5) -> list[dict]:
    body = {
        "query": {
            "sparse_vector": {
                "field": "elser_sparse",
                "inference_id": ".elser_model_2_linux-x86_64",
                "query": question
            }
        },
        "size": k,
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "method": "elser"} for h in hits]
```

### 5.4 Hybrid (BM25 + ELSER v2 via RRF)

```python
def retrieve_hybrid(question: str, k: int = 5) -> list[dict]:
    """
    Reciprocal Rank Fusion of BM25 + ELSER v2.
    rank_constant=60 is the standard default.
    window_size=k*4 ensures enough candidates for meaningful fusion.
    """
    body = {
        "retriever": {
            "rrf": {
                "retrievers": [
                    {
                        "standard": {
                            "query": {
                                "multi_match": {
                                    "query": question,
                                    "fields": ["title^2", "text"],
                                    "analyzer": "english_standard"
                                }
                            }
                        }
                    },
                    {
                        "standard": {
                            "query": {
                                "sparse_vector": {
                                    "field": "elser_sparse",
                                    "inference_id": ".elser_model_2_linux-x86_64",
                                    "query": question
                                }
                            }
                        }
                    }
                ],
                "rank_constant": 60,
                "window_size": k * 4
            }
        },
        "size": k,
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "method": "hybrid"} for h in hits]
```

---

## 6. Generation Configuration

### 6.1 System Prompt (Identical for All Conditions)

```python
SYSTEM_PROMPT = """You are a precise question-answering assistant.
Answer the question using ONLY the information provided in the context below.
If the context does not contain enough information to answer the question, say:
"I cannot answer this question based on the provided context."
Do not use knowledge from outside the provided context.
Be concise and factual. Cite only claims that are directly supported by the context."""

def build_prompt(question: str, retrieved_chunks: list[dict]) -> str:
    context_parts = []
    for i, chunk in enumerate(retrieved_chunks):
        context_parts.append(f"[Context {i+1}]\n{chunk['text']}")
    context = "\n\n".join(context_parts)
    
    return f"""CONTEXT:
{context}

QUESTION: {question}

ANSWER:"""
```

### 6.2 Model Configuration

```python
MODEL_CONFIGS = {
    "mistral_7b": {
        "model": "mistral:7b",
        "temperature": 0.0,   # deterministic — critical for reproducibility
        "num_predict": 300,    # max output tokens
        "top_p": 1.0,
        "seed": 42
    },
    "gemma3_12b": {
        "model": "gemma3:12b",
        "temperature": 0.0,
        "num_predict": 300,
        "top_p": 1.0,
        "seed": 42
    }
}

def generate_response(question: str, retrieved_chunks: list[dict],
                      model_key: str) -> dict:
    config = MODEL_CONFIGS[model_key]
    prompt = build_prompt(question, retrieved_chunks)
    
    response = ollama.generate(
        model=config["model"],
        prompt=prompt,
        system=SYSTEM_PROMPT,
        options={
            "temperature": config["temperature"],
            "num_predict": config["num_predict"],
            "top_p": config["top_p"],
            "seed": config["seed"]
        }
    )
    
    return {
        "response_text": response["response"],
        "model": model_key,
        "prompt_tokens": response.get("prompt_eval_count", -1),
        "response_tokens": response.get("eval_count", -1),
        "latency_ms": response.get("total_duration", 0) / 1e6
    }
```

### 6.3 No-RAG Baseline Condition

```python
def generate_no_rag(question: str, model_key: str) -> dict:
    """
    No-RAG baseline: model answers from parametric knowledge only.
    Uses the same system prompt but empty context.
    """
    config = MODEL_CONFIGS[model_key]
    prompt = f"QUESTION: {question}\n\nANSWER:"
    
    response = ollama.generate(
        model=config["model"],
        prompt=prompt,
        system="You are a precise question-answering assistant. Answer concisely and factually.",
        options={
            "temperature": config["temperature"],
            "num_predict": config["num_predict"],
            "seed": config["seed"]
        }
    )
    return {
        "response_text": response["response"],
        "model": model_key,
        "condition": "no_rag",
        "latency_ms": response.get("total_duration", 0) / 1e6
    }
```

---

## 7. Generation Scoring Rubric

### 7.1 Automated Scoring — RAGAS (Local LLM as Judge)

Configure RAGAS to use Ollama instead of OpenAI:

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall
)
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_ollama import ChatOllama, OllamaEmbeddings

# Use Gemma 3 12B as the judge LLM (your most capable local model)
judge_llm = LangchainLLMWrapper(ChatOllama(model="gemma3:12b", temperature=0))
judge_embeddings = LangchainEmbeddingsWrapper(OllamaEmbeddings(model="nomic-embed-text"))

# Assign to all metrics
faithfulness.llm = judge_llm
answer_relevancy.llm = judge_llm
answer_relevancy.embeddings = judge_embeddings
context_precision.llm = judge_llm
context_recall.llm = judge_llm

def score_with_ragas(question: str, answer: str,
                     retrieved_chunks: list[dict],
                     reference_answer: str) -> dict:
    from datasets import Dataset
    
    data = {
        "question": [question],
        "answer": [answer],
        "contexts": [[c["text"] for c in retrieved_chunks]],
        "ground_truth": [reference_answer]
    }
    dataset = Dataset.from_dict(data)
    
    result = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall]
    )
    return dict(result)
```

### 7.2 Hallucination Classifier — HHEM (Local, No LLM Required)

HHEM-2.1-Open runs on CPU and classifies each (context, answer) pair as faithful or hallucinated:

```python
from transformers import pipeline

# Load once at startup
hhem_pipe = pipeline(
    "text-classification",
    model="vectara/hallucination_evaluation_model",
    device=-1  # CPU — your Xeon cores handle this fine
)

def score_hallucination_hhem(answer: str, contexts: list[str]) -> dict:
    """
    Score faithfulness of answer against each retrieved context.
    Returns per-context faithfulness and aggregate score.
    """
    combined_context = " ".join(contexts)
    
    # HHEM expects: {"text": answer, "text_pair": context}
    input_pair = {"text": answer, "text_pair": combined_context}
    result = hhem_pipe(input_pair, truncation=True, max_length=512)[0]
    
    return {
        "hhem_label": result["label"],      # "consistent" or "hallucinated"
        "hhem_score": result["score"],       # confidence (0-1)
        "hhem_faithful": result["label"] == "consistent"
    }
```

### 7.3 Human Scoring Rubric (For Subset Validation)

Apply to a **random 20% sample** (20 questions × 8 conditions = 160 evaluations) to validate automated scores.

**Score each response on three dimensions (0–3 each):**

**Faithfulness (F) — Is the answer grounded in the retrieved context?**

| Score | Definition |
|---|---|
| **3** | Every claim in the answer is directly supported by the provided context. No information added from outside the context. |
| **2** | Most claims are supported. One minor unsupported claim that is plausibly inferable from context. |
| **1** | Core answer is present but contains at least one clearly unsupported or fabricated claim. |
| **0** | Answer contradicts the context, ignores the context, or is entirely fabricated. |

**Answer Relevance (R) — Does the answer actually address the question?**

| Score | Definition |
|---|---|
| **3** | Directly and completely answers what was asked. |
| **2** | Answers the question but misses a component or adds irrelevant information. |
| **1** | Partially answers the question. Core question not fully resolved. |
| **0** | Does not answer the question. Off-topic or "I cannot answer" when context was sufficient. |

**Correctness (C) — Is the answer factually accurate (relative to the reference answer)?**

| Score | Definition |
|---|---|
| **3** | Matches the reference answer in all key facts. |
| **2** | Correct on main point, minor detail wrong or missing. |
| **1** | Partially correct. Gets one fact right but contradicts another. |
| **0** | Factually wrong in a material way. |

**Composite human score:** `H = (F + R + C) / 9` — normalized to [0, 1]

**Labeling Rules:**
- Score F before reading R or C (prevents anchoring)
- Use the reference answer only when scoring C
- For "I cannot answer" responses: F=3, R=0, C=0 (correct behavior but unhelpful)
- Two scorers required for all 160 human-scored evaluations; resolve disagreements by discussion

---

## 8. Result Record Schema

Every evaluation produces one result record. Store all results in a single JSONL file.

```json
{
  "run_id": "run_20241215_143022",
  "question_id": "Q0042",
  "question_text": "What cellular mechanism allows mRNA vaccines to produce an immune response?",
  "query_type": "conceptual",
  "difficulty": "medium",
  "model": "mistral_7b",
  "retrieval_condition": "hybrid",

  "retrieved_chunks": [
    {
      "chunk_id": "doc_00312_chunk_00003",
      "rank": 1,
      "score": 0.923,
      "text": "...",
      "relevance_grade": 3
    }
  ],

  "retrieval_metrics": {
    "precision_at_5": 0.60,
    "recall_at_10": 0.75,
    "ndcg_at_10": 0.71,
    "mrr": 1.00
  },

  "response_text": "mRNA vaccines introduce messenger RNA into cells...",
  "latency_ms": 4230,

  "ragas_scores": {
    "faithfulness": 0.89,
    "answer_relevancy": 0.91,
    "context_precision": 0.75,
    "context_recall": 0.80
  },

  "hhem_scores": {
    "hhem_label": "consistent",
    "hhem_score": 0.94,
    "hhem_faithful": true
  },

  "human_scores": null,

  "hallucination_flag": false,
  "refused_to_answer": false,

  "metadata": {
    "chunk_count": 5,
    "context_token_estimate": 1847,
    "elasticsearch_took_ms": 23,
    "timestamp": "2024-12-15T14:30:22Z"
  }
}
```

---

## 9. Experimental Controls

These variables are **held constant** across all conditions. Any deviation must be logged.

| Control Variable | Specified Value | Rationale |
|---|---|---|
| **K (chunks retrieved)** | 5 | Standard for RAG; captures 2,500 tokens of context |
| **Chunk size** | 400 words / 20% overlap | Per literature consensus (256–512 tokens) |
| **System prompt** | Identical template (Section 6.1) | Isolates retrieval, not prompting |
| **Temperature** | 0.0 for all models | Reproducibility; eliminates stochastic variation |
| **Seed** | 42 | Reproducible Ollama inference |
| **ELSER inference endpoint** | `.elser_model_2_linux-x86_64` | Optimized for SheepSOC's Xeon CPUs |
| **RRF rank_constant** | 60 | Published standard default |
| **Judge LLM** | Gemma 3 12B (for Mistral 7B runs) | Avoid same-model self-evaluation bias |
| **Embedding model** | `nomic-embed-text` | Single model for dense; prevents confounding |
| **Index** | Single shared index | Same document storage for all methods |
| **Elasticsearch version** | 8.13+ | Required for RRF and sparse_vector |
| **Evaluation order** | Randomized across conditions | Prevent systematic Elasticsearch cache bias |
| **GPU assignment** | RTX 5060 Ti for LLM inference only | No GPU use for ELSER (runs on Xeon CPUs) |

---

## 10. Execution Order

```python
# main_experiment.py — full study runner

import json
import random
from datetime import datetime
from pathlib import Path

RETRIEVAL_METHODS = {
    "bm25": retrieve_bm25,
    "dense": retrieve_dense,
    "elser": retrieve_elser,
    "hybrid": retrieve_hybrid
}

MODELS = ["mistral_7b", "gemma3_12b"]

def run_study(questions_path: str, output_path: str):
    questions = [json.loads(l) for l in open(questions_path)]
    
    # Build full experiment matrix: all conditions × all questions
    # Randomize order to prevent cache/ordering artifacts
    experiment_jobs = []
    for q in questions:
        # No-RAG baseline for each model
        for model in MODELS:
            experiment_jobs.append((q, "no_rag", model))
        # Retrieval conditions for each model
        for method in RETRIEVAL_METHODS:
            for model in MODELS:
                experiment_jobs.append((q, method, model))
    
    random.seed(42)
    random.shuffle(experiment_jobs)
    
    results = []
    total = len(experiment_jobs)
    
    for i, (question, method, model) in enumerate(experiment_jobs):
        print(f"[{i+1}/{total}] Q={question['question_id']} method={method} model={model}")
        
        # Retrieve
        if method == "no_rag":
            chunks = []
            retrieval_metrics = {}
        else:
            chunks = RETRIEVAL_METHODS[method](question["question_text"])
            
            # Compute retrieval metrics using golden relevance grades
            grades = [
                get_relevance_grade(c["chunk_id"], question["reference_answer_source_chunks"])
                for c in chunks
            ]
            retrieval_metrics = {
                "precision_at_5": precision_at_k(grades, k=5),
                "recall_at_10": recall_at_k(grades, k=10,
                                            all_relevant_count=len(question["reference_answer_source_chunks"])),
                "ndcg_at_10": ndcg_at_k(grades, k=10),
                "mrr": mrr(grades)
            }
        
        # Generate
        if method == "no_rag":
            gen = generate_no_rag(question["question_text"], model)
        else:
            gen = generate_response(question["question_text"], chunks, model)
        
        # Score — RAGAS
        if chunks:
            ragas = score_with_ragas(
                question["question_text"], gen["response_text"],
                chunks, question["reference_answer"]
            )
        else:
            ragas = {"faithfulness": None, "answer_relevancy": None,
                     "context_precision": None, "context_recall": None}
        
        # Score — HHEM
        if chunks:
            hhem = score_hallucination_hhem(
                gen["response_text"],
                [c["text"] for c in chunks]
            )
        else:
            hhem = {"hhem_label": None, "hhem_score": None, "hhem_faithful": None}
        
        record = {
            "run_id": f"run_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            "question_id": question["question_id"],
            "question_text": question["question_text"],
            "query_type": question["query_type"],
            "difficulty": question["difficulty"],
            "model": model,
            "retrieval_condition": method,
            "retrieved_chunks": chunks,
            "retrieval_metrics": retrieval_metrics,
            "response_text": gen["response_text"],
            "latency_ms": gen["latency_ms"],
            "ragas_scores": ragas,
            "hhem_scores": hhem,
            "human_scores": None,
            "hallucination_flag": not hhem.get("hhem_faithful", True),
            "refused_to_answer": "cannot answer" in gen["response_text"].lower(),
            "metadata": {"timestamp": datetime.now().isoformat()}
        }
        results.append(record)
        
        # Write incrementally (study-safe: resume from checkpoint)
        with open(output_path, "a") as f:
            f.write(json.dumps(record) + "\n")
    
    print(f"Study complete. {len(results)} records written to {output_path}")
```

---

## 11. Statistical Analysis Plan

### 11.1 Primary Tests

**Primary research question:** Does retrieval condition significantly affect faithfulness?

```python
import pandas as pd
import scipy.stats as stats
from itertools import combinations

df = pd.read_json("results.jsonl", lines=True)
df["faithfulness"] = df["ragas_scores"].apply(lambda x: x.get("faithfulness"))

# Test 1: Kruskal-Wallis (non-parametric ANOVA) across all conditions
# Use non-parametric because faithfulness scores are unlikely to be normally distributed

for model in ["mistral_7b", "gemma3_12b"]:
    subset = df[df["model"] == model]
    groups = [
        subset[subset["retrieval_condition"] == c]["faithfulness"].dropna().values
        for c in ["no_rag", "bm25", "dense", "elser", "hybrid"]
    ]
    stat, p = stats.kruskal(*groups)
    print(f"Model: {model} | Kruskal-Wallis H={stat:.3f}, p={p:.4f}")
    
    # If significant, do pairwise Mann-Whitney with Bonferroni correction
    # 10 pairwise comparisons → alpha threshold = 0.05/10 = 0.005
    pairs = list(combinations(["no_rag", "bm25", "dense", "elser", "hybrid"], 2))
    for c1, c2 in pairs:
        g1 = subset[subset["retrieval_condition"] == c1]["faithfulness"].dropna()
        g2 = subset[subset["retrieval_condition"] == c2]["faithfulness"].dropna()
        u, p = stats.mannwhitneyu(g1, g2, alternative="two-sided")
        # Compute rank-biserial correlation as effect size
        n1, n2 = len(g1), len(g2)
        r = 1 - (2 * u) / (n1 * n2)
        sig = "***" if p < 0.005 else ("*" if p < 0.05 else "ns")
        print(f"  {c1} vs {c2}: p={p:.4f} {sig}, r={r:.3f}")
```

### 11.2 Secondary Tests

**Query type interaction:** Does retrieval condition effect depend on query type?

```python
from scipy.stats import kruskal
import statsmodels.formula.api as smf

# Two-way interaction: condition × query_type → faithfulness
# Use mixed-effects approach since same question appears in multiple conditions

for qtype in ["exact", "conceptual", "multi_hop"]:
    subset = df[df["query_type"] == qtype]
    for model in ["mistral_7b", "gemma3_12b"]:
        m_sub = subset[subset["model"] == model]
        groups = [
            m_sub[m_sub["retrieval_condition"] == c]["faithfulness"].dropna()
            for c in ["bm25", "dense", "elser", "hybrid"]
        ]
        stat, p = kruskal(*groups)
        print(f"  qtype={qtype} model={model}: H={stat:.3f}, p={p:.4f}")
```

**Hallucination rate (binary):** Use Chi-squared test between conditions.

```python
# Hallucination rate per condition
pivot = df.groupby(["model", "retrieval_condition"])["hallucination_flag"].agg(
    ["sum", "count"]
).reset_index()
pivot["hallucination_rate"] = pivot["sum"] / pivot["count"]

# Chi-squared: BM25 vs hybrid (primary comparison)
for model in ["mistral_7b", "gemma3_12b"]:
    for c1, c2 in [("bm25", "hybrid"), ("no_rag", "hybrid"), ("dense", "elser")]:
        r1 = df[(df["model"]==model) & (df["retrieval_condition"]==c1)]["hallucination_flag"]
        r2 = df[(df["model"]==model) & (df["retrieval_condition"]==c2)]["hallucination_flag"]
        ct = pd.crosstab(
            pd.concat([r1.rename("flag"), r2.rename("flag")]),
            pd.concat([pd.Series([c1]*len(r1)), pd.Series([c2]*len(r2))])
        )
        chi2, p, dof, expected = stats.chi2_contingency(ct)
        print(f"  {model} {c1} vs {c2}: chi2={chi2:.3f}, p={p:.4f}")
```

### 11.3 Minimum Detectable Effect

With n=100 questions per condition, non-parametric tests have approximately 80% power to detect:
- A difference in median faithfulness of ~0.08 (8 percentage points) at α=0.005
- A hallucination rate difference of ~15 percentage points at α=0.05

If you observe smaller differences than these, the study is **underpowered** to draw conclusions — you would need more questions.

---

## 12. Expected Result Tables

These are projections based on published literature, calibrated for a medium-difficulty general-knowledge corpus. **Your actual numbers will differ.** These are planning benchmarks, not claims.

### Table 1: Expected Retrieval Metrics (100 Questions, K=5)

| Retrieval Method | Precision@5 | Recall@10 | nDCG@10 | MRR | Latency (p50, ms) |
|---|---|---|---|---|---|
| BM25 | 0.52–0.62 | 0.55–0.65 | 0.54–0.64 | 0.55–0.65 | 5–15 |
| Dense (nomic) | 0.58–0.68 | 0.62–0.72 | 0.60–0.70 | 0.60–0.70 | 40–80 |
| ELSER v2 | 0.63–0.73 | 0.67–0.77 | 0.65–0.75 | 0.65–0.75 | 80–200 |
| Hybrid (BM25+ELSER) | 0.68–0.78 | 0.72–0.82 | 0.70–0.80 | 0.70–0.80 | 90–220 |

**Expected pattern by query type:**

| Query Type | BM25 Best For? | Dense Best For? | Hybrid Advantage? |
|---|---|---|---|
| Exact / Entity | ✅ Yes | ❌ No | Marginal |
| Conceptual | ❌ No | ✅ Yes | Moderate |
| Multi-hop | ❌ No | Moderate | ✅ Yes |

### Table 2: Expected Generation Metrics (RAGAS Faithfulness)

| Condition | Mistral 7B | Gemma 3 12B | Mistral Δ from No-RAG | Gemma Δ from No-RAG |
|---|---|---|---|---|
| No RAG | 0.40–0.55 | 0.50–0.65 | — | — |
| BM25 | 0.58–0.68 | 0.65–0.75 | +0.15–0.18 | +0.12–0.15 |
| Dense | 0.63–0.73 | 0.70–0.80 | +0.20–0.23 | +0.17–0.20 |
| ELSER v2 | 0.68–0.78 | 0.75–0.85 | +0.25–0.28 | +0.22–0.25 |
| Hybrid | 0.73–0.83 | 0.78–0.88 | +0.28–0.33 | +0.25–0.28 |

**Key prediction:** The Mistral 7B faithfulness gain from No-RAG → Hybrid should be **larger in absolute terms** than Gemma 3 12B's gain, because Mistral 7B has less parametric knowledge and thus relies more heavily on retrieved context.

### Table 3: Expected Hallucination Rates (HHEM)

| Condition | Mistral 7B | Gemma 3 12B |
|---|---|---|
| No RAG | 30–45% | 20–35% |
| BM25 | 22–32% | 15–25% |
| Dense | 18–28% | 12–22% |
| ELSER v2 | 14–24% | 10–18% |
| Hybrid | 10–18% | 8–15% |

### Table 4: Latency Budget (SheepSOC Hardware)

| Component | Mistral 7B | Gemma 3 12B | Notes |
|---|---|---|---|
| nomic-embed-text (query) | ~20ms | ~20ms | GPU-accelerated |
| BM25 retrieval | ~10ms | ~10ms | Xeon CPU, ES cache helps |
| ELSER v2 retrieval | ~120ms | ~120ms | Xeon CPU, no GPU |
| LLM inference | ~3–6s | ~6–12s | RTX 5060 Ti 16GB |
| RAGAS scoring | ~8–15s | — | Gemma 3 12B as judge |
| HHEM scoring | ~200ms | ~200ms | CPU only |
| **Total per evaluation** | **~12–22s** | **~15–28s** | Varies by retrieval method |
| **Full study (1,000 evals)** | — | — | **~8–14 hours** |

---

## 13. Required Plots

Produce all plots in a single analysis Jupyter notebook after the run completes.

### Plot 1: Retrieval Method Comparison — Box Plot Matrix
```python
import matplotlib.pyplot as plt
import seaborn as sns

fig, axes = plt.subplots(2, 2, figsize=(14, 10))
metrics = ["precision_at_5", "recall_at_10", "ndcg_at_10", "mrr"]
conditions = ["bm25", "dense", "elser", "hybrid"]

for ax, metric in zip(axes.flatten(), metrics):
    data = df[df["retrieval_condition"].isin(conditions)].copy()
    data["metric_value"] = data["retrieval_metrics"].apply(lambda x: x.get(metric))
    
    sns.boxplot(
        data=data, x="retrieval_condition", y="metric_value",
        order=conditions, ax=ax, palette="Blues"
    )
    ax.set_title(f"{metric.upper()} by Retrieval Method")
    ax.set_xlabel("Retrieval Method")
    ax.set_ylabel(metric)
    ax.set_ylim(0, 1)

plt.suptitle("SheepSOC-RAG-001: Retrieval Performance Comparison", y=1.02)
plt.tight_layout()
plt.savefig("plots/plot1_retrieval_comparison.png", dpi=150, bbox_inches="tight")
```

### Plot 2: Faithfulness by Condition and Model — Side-by-Side Box Plots
```python
fig, axes = plt.subplots(1, 2, figsize=(16, 6), sharey=True)
all_conditions = ["no_rag", "bm25", "dense", "elser", "hybrid"]
colors = ["#d62728", "#1f77b4", "#2ca02c", "#ff7f0e", "#9467bd"]

for ax, model in zip(axes, ["mistral_7b", "gemma3_12b"]):
    subset = df[df["model"] == model].copy()
    subset["faithfulness"] = subset["ragas_scores"].apply(lambda x: x.get("faithfulness"))
    
    sns.boxplot(
        data=subset, x="retrieval_condition", y="faithfulness",
        order=all_conditions, palette=colors, ax=ax
    )
    ax.set_title(f"Faithfulness — {model.replace('_', ' ').title()}")
    ax.set_xlabel("Retrieval Condition")
    ax.set_ylabel("RAGAS Faithfulness Score")
    ax.set_ylim(0, 1)
    ax.axhline(y=0.9, color="red", linestyle="--", alpha=0.5, label="Target (0.9)")

plt.suptitle("SheepSOC-RAG-001: Generation Faithfulness by Retrieval Condition", y=1.02)
plt.tight_layout()
plt.savefig("plots/plot2_faithfulness_by_condition.png", dpi=150, bbox_inches="tight")
```

### Plot 3: Query Type Interaction — Heatmap
```python
fig, axes = plt.subplots(1, 2, figsize=(16, 5))

for ax, model in zip(axes, ["mistral_7b", "gemma3_12b"]):
    subset = df[df["model"] == model].copy()
    subset["faithfulness"] = subset["ragas_scores"].apply(lambda x: x.get("faithfulness"))
    
    pivot = subset.groupby(["query_type", "retrieval_condition"])["faithfulness"].mean().unstack()
    pivot = pivot.reindex(columns=all_conditions)
    
    sns.heatmap(
        pivot, annot=True, fmt=".2f", cmap="RdYlGn",
        vmin=0.3, vmax=1.0, ax=ax,
        linewidths=0.5
    )
    ax.set_title(f"Mean Faithfulness Heatmap — {model.replace('_', ' ').title()}")
    ax.set_xlabel("Retrieval Condition")
    ax.set_ylabel("Query Type")

plt.suptitle("SheepSOC-RAG-001: Faithfulness by Query Type × Retrieval Condition")
plt.tight_layout()
plt.savefig("plots/plot3_heatmap_qtype_condition.png", dpi=150, bbox_inches="tight")
```

### Plot 4: Retrieval Quality → Faithfulness Scatter (The Key Causal Plot)
```python
fig, axes = plt.subplots(1, 2, figsize=(14, 6))
conditions_with_retrieval = ["bm25", "dense", "elser", "hybrid"]

for ax, model in zip(axes, ["mistral_7b", "gemma3_12b"]):
    subset = df[(df["model"] == model) &
                (df["retrieval_condition"].isin(conditions_with_retrieval))].copy()
    subset["faithfulness"] = subset["ragas_scores"].apply(lambda x: x.get("faithfulness"))
    subset["precision_at_5"] = subset["retrieval_metrics"].apply(lambda x: x.get("precision_at_5"))
    subset = subset.dropna(subset=["faithfulness", "precision_at_5"])
    
    condition_colors = {"bm25": "#1f77b4", "dense": "#2ca02c",
                        "elser": "#ff7f0e", "hybrid": "#9467bd"}
    
    for cond, color in condition_colors.items():
        cond_data = subset[subset["retrieval_condition"] == cond]
        ax.scatter(cond_data["precision_at_5"], cond_data["faithfulness"],
                   alpha=0.4, color=color, label=cond, s=30)
    
    # Add regression line
    from numpy.polynomial import polynomial as P
    x = subset["precision_at_5"].values
    y = subset["faithfulness"].values
    coeffs = P.polyfit(x, y, 1)
    xline = np.linspace(0, 1, 100)
    ax.plot(xline, P.polyval(xline, coeffs), "k--", alpha=0.6, label="Trend")
    
    ax.set_xlabel("Precision@5 (Retrieval Quality)")
    ax.set_ylabel("Faithfulness (Generation Quality)")
    ax.set_title(f"Retrieval Quality → Faithfulness\n{model.replace('_', ' ').title()}")
    ax.set_xlim(0, 1); ax.set_ylim(0, 1)
    ax.legend(title="Condition")

plt.suptitle("SheepSOC-RAG-001: Does Better Retrieval Cause Better Generation?")
plt.tight_layout()
plt.savefig("plots/plot4_retrieval_vs_faithfulness.png", dpi=150, bbox_inches="tight")
```

### Plot 5: Hallucination Rate Bar Chart
```python
fig, ax = plt.subplots(figsize=(12, 6))

halluc_data = df.groupby(["model", "retrieval_condition"])["hallucination_flag"].mean().reset_index()
halluc_data.columns = ["model", "condition", "hallucination_rate"]

x = np.arange(len(all_conditions))
width = 0.35
models_list = ["mistral_7b", "gemma3_12b"]
model_labels = ["Mistral 7B", "Gemma 3 12B"]
bar_colors = ["#1f77b4", "#ff7f0e"]

for i, (model, label, color) in enumerate(zip(models_list, model_labels, bar_colors)):
    rates = [halluc_data[(halluc_data["model"]==model) &
                         (halluc_data["condition"]==c)]["hallucination_rate"].values[0]
             for c in all_conditions]
    bars = ax.bar(x + i*width - width/2, rates, width, label=label, color=color, alpha=0.8)
    ax.bar_label(bars, fmt="%.0f%%", label_type="edge", padding=2, fontsize=9)

ax.set_xticks(x)
ax.set_xticklabels([c.replace("_", "\n") for c in all_conditions])
ax.set_ylabel("Hallucination Rate (HHEM)")
ax.set_ylim(0, 0.6)
ax.set_title("SheepSOC-RAG-001: Hallucination Rate by Retrieval Condition and Model")
ax.legend()
plt.tight_layout()
plt.savefig("plots/plot5_hallucination_rate.png", dpi=150, bbox_inches="tight")
```

---

## 14. Pre-Study Sanity Checks

Run these before the full experiment. Abort and fix if any check fails.

```python
# sanity_checks.py

def run_sanity_checks():
    PASS, FAIL = [], []

    # Check 1: All four retrieval methods return K=5 results
    test_q = "what is the primary function of the immune system"
    for name, fn in RETRIEVAL_METHODS.items():
        results = fn(test_q, k=5)
        if len(results) == 5:
            PASS.append(f"Retrieval {name}: returns 5 results ✅")
        else:
            FAIL.append(f"Retrieval {name}: returned {len(results)} results ❌")

    # Check 2: ELSER and Dense return DIFFERENT top results for same query
    dense_r = retrieve_dense(test_q)
    elser_r = retrieve_elser(test_q)
    dense_ids = {r["chunk_id"] for r in dense_r}
    elser_ids = {r["chunk_id"] for r in elser_r}
    overlap = len(dense_ids & elser_ids) / 5
    if overlap < 0.8:
        PASS.append(f"Dense vs ELSER overlap={overlap:.0%} < 80% ✅ (methods are distinct)")
    else:
        FAIL.append(f"Dense vs ELSER overlap={overlap:.0%} — methods may be too similar ⚠️")

    # Check 3: Both models produce non-empty output
    for model in MODELS:
        result = generate_response(test_q, dense_r, model)
        if len(result["response_text"]) > 20:
            PASS.append(f"Model {model}: generates output ✅")
        else:
            FAIL.append(f"Model {model}: output too short ❌")

    # Check 4: RAGAS scores are in [0, 1]
    ragas = score_with_ragas(test_q, "The immune system defends the body.",
                              dense_r, "The immune system protects the body from pathogens.")
    for k, v in ragas.items():
        if v is None or not (0 <= v <= 1):
            FAIL.append(f"RAGAS {k} = {v} out of bounds ❌")
        else:
            PASS.append(f"RAGAS {k} = {v:.3f} ✅")

    # Check 5: HHEM returns consistent or hallucinated
    hhem = score_hallucination_hhem("The immune system defends the body.",
                                    [r["text"] for r in dense_r])
    if hhem["hhem_label"] in ["consistent", "hallucinated"]:
        PASS.append("HHEM: returns valid label ✅")
    else:
        FAIL.append(f"HHEM: unexpected label {hhem['hhem_label']} ❌")

    # Check 6: Golden dataset has correct distribution
    questions = [json.loads(l) for l in open("golden_questions.jsonl")]
    type_counts = {t: sum(1 for q in questions if q["query_type"] == t)
                   for t in ["exact", "conceptual", "multi_hop"]}
    expected = {"exact": 30, "conceptual": 40, "multi_hop": 30}
    if type_counts == expected:
        PASS.append(f"Golden dataset distribution correct: {type_counts} ✅")
    else:
        FAIL.append(f"Golden dataset distribution wrong: {type_counts} ❌ (expected {expected})")

    print("\n=== SANITY CHECK RESULTS ===")
    for msg in PASS: print(f"  {msg}")
    for msg in FAIL: print(f"  {msg}")
    print(f"\n{'✅ ALL CHECKS PASSED — proceed with study' if not FAIL else '❌ FAILURES DETECTED — fix before running'}")
    return len(FAIL) == 0
```

---

## See Also

- [Research Agenda](../agenda.md) — the strategic research questions this protocol is designed to answer
- [RAG-001 Precision Audit](precision-audit.md) — citation audit and formal measurement methodology for the claims underlying this protocol's design
- [OpenWebUI & RAG](../../infrastructure/platforms/openwebui-rag.md) — the RAG platform providing the Elasticsearch index and Ollama inference used in this study
- [Elasticsearch & ELSER](../../infrastructure/platforms/elasticsearch-elser.md) — ELSER configuration and dual-use index that this protocol runs queries against
- [Knowledge Bases](../../infrastructure/platforms/knowledge-bases.md) — the document corpora that serve as the research corpus

## 15. Study Deliverables Checklist

At study completion, verify the following exist and are complete:

```
sheepsoc-rag-001/
├── data/
│   ├── corpus/
│   │   └── chunks.jsonl                 # all indexed chunks
│   ├── golden_questions.jsonl           # 100 verified questions
│   └── results.jsonl                    # all 1,000 result records
├── analysis/
│   ├── analysis.ipynb                   # full analysis notebook
│   ├── statistical_tests.csv            # all p-values, effect sizes
│   └── summary_table.csv               # mean metrics per condition/model
├── plots/
│   ├── plot1_retrieval_comparison.png
│   ├── plot2_faithfulness_by_condition.png
│   ├── plot3_heatmap_qtype_condition.png
│   ├── plot4_retrieval_vs_faithfulness.png
│   └── plot5_hallucination_rate.png
├── human_labels/
│   └── human_scored_sample.csv         # 160 manually scored records
└── logs/
    ├── sanity_checks.log
    ├── indexing.log
    └── study_run.log
```

**The single most important result to report:**
A table with one row per condition, showing mean faithfulness ± std with statistical significance markers against the BM25 baseline — separately for Mistral 7B and Gemma 3 12B, and then broken out by query type. That table is the core finding of SheepSOC-RAG-001.

---

---

# SheepSOC-RAG-001: Protocol Revision 1.1
## Four Targeted Fixes + Phase 1 Build Order

---

## Fix 1 — K Consistency

### The Problem

The previous protocol contained three conflicting values of K:

| Location | K Used | Problem |
|---|---|---|
| Elasticsearch `size=` | 5 | Only fetches 5 candidates |
| `recall_at_10()` call | 10 | Requires 10 retrieved results to compute |
| `ndcg_at_k(grades, k=10)` | 10 | Requires 10 grades — but only 5 retrieved |
| LLM context injection | 5 | Correct budget limit |
| `num_candidates` | `k*10 = 50` | Based on wrong K |

The result: `recall@10` and `nDCG@10` were being computed over only 5 retrieved results, which makes both metrics meaningless. `recall@10` with only 5 results is equivalent to `recall@5`, and `nDCG@10` with 5 grades pads the rest with zeros — silently deflating scores rather than raising an error.

### The Fix

**Retrieve K=10 from Elasticsearch. Pass top-5 to the LLM. Compute all metrics over K=10.**

This is standard IR evaluation practice: the retrieval system is evaluated over its full candidate set; the generation system receives a context-window-bounded subset. These are two separate decisions with separate K values.

```
K_RETRIEVE = 10    # How many chunks Elasticsearch returns
K_CONTEXT  = 5     # How many chunks are passed to the LLM
K_METRIC   = 10    # The K used in all metric computation (Precision@5, Recall@10, nDCG@10, MRR@10)
```

**Note:** Precision@5 is computed over the first 5 of the 10 retrieved results (the subset that the LLM actually sees). Recall@10, nDCG@10, and MRR are computed over all 10. This creates a deliberate split: Precision@5 measures retrieval quality *as the model experiences it*; Recall@10 measures retrieval coverage *of the full system*.

### Updated Constants

```python
# constants.py — single source of truth for all K values

K_RETRIEVE = 10    # Elasticsearch size= parameter
K_CONTEXT  = 5     # Chunks injected into LLM prompt
K_METRIC   = 10    # Denominator for nDCG, Recall, MRR
K_CANDIDATES_MULTIPLIER = 5  # num_candidates = K_RETRIEVE * this = 50
```

### Updated Retrieval Functions (All Four Methods)

Replace all four retrieval functions. The only change is `size` and `num_candidates`; query bodies are identical to the previous version.

```python
# retrieval.py

from constants import K_RETRIEVE, K_CANDIDATES_MULTIPLIER

def retrieve_bm25(question: str) -> list[dict]:
    body = {
        "query": {
            "multi_match": {
                "query": question,
                "fields": ["title^2", "text"],
                "type": "best_fields",
                "analyzer": "english_standard"
            }
        },
        "size": K_RETRIEVE,                          # ← was 5, now 10
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "rank": i + 1,
             "method": "bm25"} for i, h in enumerate(hits)]


def retrieve_dense(question: str) -> list[dict]:
    query_vector = get_nomic_embedding(question)
    body = {
        "knn": {
            "field": "dense_vector",
            "query_vector": query_vector,
            "k": K_RETRIEVE,                         # ← was 5, now 10
            "num_candidates": K_RETRIEVE * K_CANDIDATES_MULTIPLIER  # ← now 50
        },
        "size": K_RETRIEVE,
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "rank": i + 1,
             "method": "dense"} for i, h in enumerate(hits)]


def retrieve_elser(question: str) -> list[dict]:
    body = {
        "query": {
            "sparse_vector": {
                "field": "elser_sparse",
                "inference_id": ".elser_model_2_linux-x86_64",
                "query": question
            }
        },
        "size": K_RETRIEVE,                          # ← was 5, now 10
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "rank": i + 1,
             "method": "elser"} for i, h in enumerate(hits)]


def retrieve_hybrid(question: str) -> list[dict]:
    body = {
        "retriever": {
            "rrf": {
                "retrievers": [
                    {"standard": {"query": {"multi_match": {
                        "query": question,
                        "fields": ["title^2", "text"],
                        "analyzer": "english_standard"
                    }}}},
                    {"standard": {"query": {"sparse_vector": {
                        "field": "elser_sparse",
                        "inference_id": ".elser_model_2_linux-x86_64",
                        "query": question
                    }}}}
                ],
                "rank_constant": 60,
                "window_size": K_RETRIEVE * 4        # ← now 40
            }
        },
        "size": K_RETRIEVE,                          # ← was 5, now 10
        "_source": ["chunk_id", "doc_id", "text", "title"]
    }
    hits = es.search(index=INDEX, body=body)["hits"]["hits"]
    return [{"chunk_id": h["_source"]["chunk_id"],
             "text": h["_source"]["text"],
             "score": h["_score"],
             "rank": i + 1,
             "method": "hybrid"} for i, h in enumerate(hits)]
```

### Updated Metric Computation

```python
# metrics.py

from constants import K_RETRIEVE, K_CONTEXT, K_METRIC

def compute_retrieval_metrics(retrieved_chunks: list[dict],
                               qrels_for_question: dict[str, int]) -> dict:
    """
    retrieved_chunks: ordered list of K_RETRIEVE=10 chunks from Elasticsearch
    qrels_for_question: dict mapping chunk_id → relevance grade (0-3)
                        from the qrels file (see Fix 2)
    """
    grades = [qrels_for_question.get(c["chunk_id"], 0) for c in retrieved_chunks]
    
    assert len(grades) == K_RETRIEVE, (
        f"Expected {K_RETRIEVE} grades, got {len(grades)}. "
        f"Check that all retrieval methods return exactly K_RETRIEVE results."
    )

    # Count total relevant chunks in the corpus for this question (grade >= 2)
    # This comes from qrels, not from the retrieved set
    n_relevant_in_corpus = sum(1 for g in qrels_for_question.values() if g >= 2)

    return {
        # Precision@5 — over the K_CONTEXT=5 chunks the LLM sees
        "precision_at_5": precision_at_k(grades, k=5, threshold=2),

        # All others computed over the full K_RETRIEVE=10 retrieved set
        "recall_at_10":   recall_at_k(grades, k=10,
                                       n_relevant=n_relevant_in_corpus, threshold=2),
        "ndcg_at_10":     ndcg_at_k(grades, k=10),
        "mrr_at_10":      mrr(grades, threshold=2),

        # Diagnostic: what fraction of retrieved set is noise?
        "noise_rate_at_10": sum(1 for g in grades if g == 0) / K_RETRIEVE,

        # Record what was actually passed to the LLM
        "context_chunk_ids": [c["chunk_id"] for c in retrieved_chunks[:K_CONTEXT]],
        "n_relevant_in_corpus": n_relevant_in_corpus
    }
```

### Updated Generation Call

```python
def build_prompt(question: str, retrieved_chunks: list[dict]) -> str:
    # Only pass the top K_CONTEXT=5 chunks to the LLM
    context_chunks = retrieved_chunks[:K_CONTEXT]
    context_parts = [f"[Context {i+1}]\n{c['text']}"
                     for i, c in enumerate(context_chunks)]
    context = "\n\n".join(context_parts)
    return f"CONTEXT:\n{context}\n\nQUESTION: {question}\n\nANSWER:"
```

### Updated Metric Boundary Table

| Metric | K Used | Computed Over | What It Measures |
|---|---|---|---|
| Precision@5 | 5 | Ranks 1–5 only | Quality of what the LLM actually sees |
| Recall@10 | 10 | Ranks 1–10 | Coverage: did retrieval find the relevant material? |
| nDCG@10 | 10 | Ranks 1–10 | Ranking quality with position discount |
| MRR@10 | 10 | Ranks 1–10 | Position of first relevant result |
| Context passed to LLM | 5 | Ranks 1–5 | Fixed budget; does not change |

---

## Fix 2 — Separate Qrels File

### The Problem

The previous protocol embedded relevance grades inside the question record:

```json
"reference_answer_source_chunks": ["doc_00312_chunk_00003", "doc_00312_chunk_00004"]
```

This is a binary list of "relevant" chunks — it cannot express grades (0, 1, 2, 3), it treats all non-listed chunks as grade 0 by default (which may be wrong for marginally relevant chunks), it is not reusable across different retrieval systems, and it conflates "source of the reference answer" with "retrieval relevance" — these are related but not identical.

The standard solution in IR evaluation (from TREC onward) is a separate **qrels file** (query relevance judgments). This file is independent of the question text, independent of the corpus, and can be loaded by any evaluation harness.

### Qrels File Schema

**File:** `qrels.jsonl` — one record per (question, chunk) pair that has been judged.

```json
{"question_id": "Q0042", "chunk_id": "doc_00312_chunk_00003", "grade": 3, "labeler": "human_1", "label_date": "2024-12-15", "notes": "Directly states the ribosome translation mechanism"}
{"question_id": "Q0042", "chunk_id": "doc_00312_chunk_00004", "grade": 2, "labeler": "human_1", "label_date": "2024-12-15", "notes": "Supporting context on antibody production"}
{"question_id": "Q0042", "chunk_id": "doc_00198_chunk_00011", "grade": 1, "labeler": "human_1", "label_date": "2024-12-15", "notes": "Background on vaccines generally, not specific to mRNA mechanism"}
```

**Important:** Only judged chunks appear in qrels. Unjudged chunks are treated as grade 0 by the metric computation code — this is the standard TREC convention and is explicitly enforced in the code below.

### Qrels Field Definitions

| Field | Type | Required | Definition |
|---|---|---|---|
| `question_id` | string | ✅ | Must match exactly a `question_id` in `golden_questions.jsonl` |
| `chunk_id` | string | ✅ | Must match a `chunk_id` in the Elasticsearch index |
| `grade` | int (0–3) | ✅ | Graded relevance per Section 4 rubric |
| `labeler` | string | ✅ | Who labeled this record (for inter-rater analysis) |
| `label_date` | ISO date | ✅ | When it was labeled |
| `notes` | string | ❌ | Labeler rationale — required for grade ≤ 1 or when labelers disagree |

### What to Judge

You cannot judge every chunk in the corpus for every question (that would be 100 questions × 10,000 chunks = 1,000,000 labels). Use **pooling**: collect the top-10 results from all four retrieval methods for each question, union them, and judge only that pool.

```python
# build_judgment_pool.py
# Run AFTER corpus is indexed and BEFORE the main study.
# Produces a pool of candidate chunks per question for human labeling.

import json

def build_judgment_pool(questions: list[dict]) -> dict[str, set[str]]:
    """
    For each question, retrieve top-10 from all four methods,
    take the union of chunk IDs, and return the pool to be labeled.
    """
    retrieval_fns = {
        "bm25":   retrieve_bm25,
        "dense":  retrieve_dense,
        "elser":  retrieve_elser,
        "hybrid": retrieve_hybrid
    }
    pools = {}
    for q in questions:
        qid = q["question_id"]
        pool = set()
        for method, fn in retrieval_fns.items():
            results = fn(q["question_text"])
            pool.update(c["chunk_id"] for c in results)
        pools[qid] = pool
        print(f"  {qid}: {len(pool)} unique chunks to judge "
              f"(from {len(retrieval_fns)*10} retrieved, with overlap)")
    return pools

def export_pool_for_labeling(pools: dict, chunks_index: dict, output_path: str):
    """
    Write a human-readable labeling spreadsheet: one row per (question, chunk) pair.
    chunks_index: dict of chunk_id → chunk text (load from corpus/chunks.jsonl)
    """
    rows = []
    for qid, chunk_ids in pools.items():
        for cid in sorted(chunk_ids):
            rows.append({
                "question_id": qid,
                "chunk_id": cid,
                "chunk_text_preview": chunks_index.get(cid, "")[:200],
                "grade": "",           # human fills this in
                "labeler": "",
                "notes": ""
            })
    
    import csv
    with open(output_path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)
    print(f"Exported {len(rows)} candidate pairs to {output_path}")
    print(f"Expected labeling time: ~{len(rows) * 0.5 / 60:.1f} hours at 30 sec/label")
```

**Expected pool size per question:** With 4 methods × 10 results = 40 retrieved, typical overlap at K=10 is 30–50%, so expect ~20–28 unique chunks per question to label. At 30 seconds per label: 100 questions × 24 chunks × 30 sec ≈ **20 hours of labeling work**. Split across two labelers for inter-rater reliability.

### Qrels Loader and Lookup

```python
# qrels.py

import json
from collections import defaultdict
from pathlib import Path

def load_qrels(qrels_path: str) -> dict[str, dict[str, int]]:
    """
    Returns: {question_id: {chunk_id: grade, ...}, ...}
    Only includes explicitly judged pairs.
    Unjudged pairs default to grade 0 at lookup time.
    """
    qrels = defaultdict(dict)
    with open(qrels_path) as f:
        for line in f:
            record = json.loads(line.strip())
            qid = record["question_id"]
            cid = record["chunk_id"]
            grade = int(record["grade"])
            
            assert 0 <= grade <= 3, f"Grade {grade} out of range in {record}"
            
            if cid in qrels[qid]:
                # Conflict: same (question, chunk) labeled twice
                existing = qrels[qid][cid]
                if existing != grade:
                    print(f"WARNING: Conflicting grades for {qid}/{cid}: "
                          f"{existing} vs {grade}. Keeping higher grade.")
                    qrels[qid][cid] = max(existing, grade)
            else:
                qrels[qid][cid] = grade
    
    return dict(qrels)

def get_grades_for_retrieved(retrieved_chunks: list[dict],
                              qrels_for_question: dict[str, int]) -> list[int]:
    """
    Map retrieved chunks to their relevance grades.
    Unjudged chunks receive grade 0 (TREC convention).
    """
    grades = []
    for chunk in retrieved_chunks:
        grade = qrels_for_question.get(chunk["chunk_id"], 0)  # 0 for unjudged
        grades.append(grade)
    return grades

def validate_qrels(qrels: dict, questions: list[dict]) -> None:
    """Pre-run check: every question has at least one grade-3 chunk."""
    errors = []
    for q in questions:
        qid = q["question_id"]
        if qid not in qrels:
            errors.append(f"  MISSING: {qid} has no qrels entries at all")
            continue
        max_grade = max(qrels[qid].values()) if qrels[qid] else 0
        n_relevant = sum(1 for g in qrels[qid].values() if g >= 2)
        if max_grade < 3:
            errors.append(f"  WARNING: {qid} has no grade-3 chunk (max={max_grade})")
        if n_relevant == 0:
            errors.append(f"  ERROR: {qid} has no relevant chunks (grade >= 2)")
    
    if errors:
        print("Qrels validation issues:")
        for e in errors: print(e)
    else:
        print(f"Qrels validated: {len(qrels)} questions, all have at least one relevant chunk.")
```

### Updated Question Record Schema

Remove `reference_answer_source_chunks` from the question record — that information now lives in qrels. The question record becomes leaner:

```json
{
  "question_id": "Q0042",
  "question_text": "What cellular mechanism allows mRNA vaccines to produce an immune response?",
  "query_type": "conceptual",
  "difficulty": "medium",
  "reference_answer": "mRNA vaccines deliver messenger RNA into cells...",
  "answer_requires_multi_hop": false,
  "expected_hard_for": ["BM25"],
  "notes": "BM25 likely misses because query uses 'cellular mechanism' while answer chunk uses 'ribosome translation'"
}
```

The mapping from `question_id` to which chunks are relevant is now entirely in `qrels.jsonl`. Never duplicate it back into the question file.

### Updated Metric Call in Main Runner

```python
# In main_experiment.py, replace the retrieval metric computation block:

# Load qrels once before the study loop
QRELS = load_qrels("data/qrels.jsonl")

# Inside the loop:
qrels_for_q = QRELS.get(question["question_id"], {})
grades = get_grades_for_retrieved(chunks, qrels_for_q)
retrieval_metrics = compute_retrieval_metrics(chunks, qrels_for_q)
```

---

## Fix 3 — No-RAG Metric Clarification

### The Problem

The previous protocol applied RAGAS faithfulness, context precision, and context recall to the No-RAG condition. This is wrong in three distinct ways:

1. **Faithfulness** is defined as `supported_claims / total_claims` where "supported" means traceable to retrieved context. With no context, every claim is by definition unsupported. The score would be 0 for every No-RAG response — not because the model is hallucinating, but because the metric is undefined. A score of 0 would contaminate mean comparisons and make No-RAG look worse than it is on the wrong axis.

2. **Context Precision** measures whether retrieved chunks are relevant. With no retrieved chunks, the metric has no operand.

3. **Context Recall** measures whether relevant information appears in retrieved chunks. Same problem.

The No-RAG condition is measuring something genuinely different: **parametric knowledge quality** — how well the model answers from its training data alone. This requires different metrics.

### Metric Applicability Table

This is the single authoritative reference for which metrics apply to which conditions. The runner enforces it via null-setting, not by skipping the record entirely (nulls are important for the result schema to remain consistent).

| Metric | No-RAG | BM25 | Dense | ELSER v2 | Hybrid | Reason |
|---|---|---|---|---|---|---|
| **Precision@5** | `null` | ✅ | ✅ | ✅ | ✅ | Requires retrieved chunks |
| **Recall@10** | `null` | ✅ | ✅ | ✅ | ✅ | Requires retrieved chunks |
| **nDCG@10** | `null` | ✅ | ✅ | ✅ | ✅ | Requires retrieved chunks |
| **MRR@10** | `null` | ✅ | ✅ | ✅ | ✅ | Requires retrieved chunks |
| **Noise Rate@10** | `null` | ✅ | ✅ | ✅ | ✅ | Requires retrieved chunks |
| **RAGAS Faithfulness** | `null` | ✅ | ✅ | ✅ | ✅ | Undefined without context |
| **RAGAS Context Precision** | `null` | ✅ | ✅ | ✅ | ✅ | Undefined without context |
| **RAGAS Context Recall** | `null` | ✅ | ✅ | ✅ | ✅ | Undefined without context |
| **RAGAS Answer Relevance** | ✅ | ✅ | ✅ | ✅ | ✅ | Measures question-answer fit |
| **HHEM Faithfulness** | `null` | ✅ | ✅ | ✅ | ✅ | No context to check against |
| **Answer Correctness** | ✅ | ✅ | ✅ | ✅ | ✅ | Measures vs reference answer |
| **Refusal Rate** | ✅ | ✅ | ✅ | ✅ | ✅ | Model says "I cannot answer" |
| **Latency (ms)** | ✅ | ✅ | ✅ | ✅ | ✅ | Always measured |

### What to Measure for No-RAG Instead

**Answer Correctness** (RAGAS) compares the model's output directly against the reference answer using semantic similarity + factual overlap. It is the right end-to-end measure when there is no context:

```python
from ragas.metrics import answer_correctness
answer_correctness.llm = judge_llm

def score_no_rag(question: str, response_text: str, reference_answer: str) -> dict:
    """
    Scoring suite for the No-RAG condition.
    Returns only metrics that are valid without retrieved context.
    """
    from datasets import Dataset
    
    data = {
        "question": [question],
        "answer": [response_text],
        "ground_truth": [reference_answer],
        # No "contexts" key — answer_correctness does not require it
    }
    dataset = Dataset.from_dict(data)
    result = evaluate(dataset, metrics=[answer_relevancy, answer_correctness])
    
    return {
        # Valid metrics
        "answer_relevancy":   result.get("answer_relevancy"),
        "answer_correctness": result.get("answer_correctness"),
        
        # Explicitly null — not undefined, but principled nulls
        "faithfulness":       None,
        "context_precision":  None,
        "context_recall":     None,
        "hhem_faithful":      None,

        # Computed from response text alone
        "refused_to_answer": "cannot answer" in response_text.lower()
    }
```

### Explicit Analysis Guardrails

Add these assertions to the analysis notebook to prevent accidental cross-condition comparisons on invalid metrics:

```python
# analysis_guards.py

def assert_no_rag_excluded(df: pd.DataFrame, metric_col: str) -> pd.DataFrame:
    """
    Raise if a metric that is undefined for No-RAG is used in a comparison
    that includes No-RAG rows.
    """
    INVALID_FOR_NO_RAG = [
        "faithfulness", "context_precision", "context_recall",
        "precision_at_5", "recall_at_10", "ndcg_at_10", "mrr_at_10",
        "hhem_faithful"
    ]
    if metric_col in INVALID_FOR_NO_RAG:
        no_rag_rows = df[df["retrieval_condition"] == "no_rag"]
        if len(no_rag_rows) > 0:
            raise ValueError(
                f"Metric '{metric_col}' is not valid for No-RAG condition. "
                f"Filter out No-RAG rows before this comparison, or use "
                f"'answer_correctness' or 'answer_relevancy' instead."
            )
    return df

# Usage in notebook:
# When comparing faithfulness across retrieval conditions:
rag_only = assert_no_rag_excluded(df, "faithfulness")

# When comparing answer correctness (valid for all conditions):
all_conditions = df  # no guard needed
```

### What No-RAG Is Actually For

The No-RAG condition answers a specific question: **"How much does retrieval help, relative to the model's baseline parametric knowledge?"**

The comparison is not:
- No-RAG faithfulness vs. BM25 faithfulness (wrong — different metrics apply)

The comparison IS:
- No-RAG answer correctness vs. Hybrid answer correctness (valid — same metric, all conditions)
- No-RAG refusal rate vs. BM25 refusal rate (valid — shows whether retrieval helps the model know when to answer)

Separately, faithfulness comparisons are:
- BM25 faithfulness vs. Dense faithfulness vs. ELSER faithfulness vs. Hybrid faithfulness

These two comparison families answer different research questions and should be presented in separate result tables and plots.

### Updated Result Record Schema (Revised Scoring Block)

```json
{
  "question_id": "Q0042",
  "retrieval_condition": "no_rag",
  "model": "mistral_7b",

  "retrieved_chunks": [],
  "retrieval_metrics": null,

  "ragas_scores": {
    "faithfulness":       null,
    "context_precision":  null,
    "context_recall":     null,
    "answer_relevancy":   0.82,
    "answer_correctness": 0.61
  },

  "hhem_scores": {
    "hhem_label":   null,
    "hhem_score":   null,
    "hhem_faithful": null
  },

  "hallucination_flag":  null,
  "refused_to_answer":   false,
  "latency_ms":          2840
}
```

---

## Fix 4 — Revised Hypotheses

### The Problem

The original hypothesis was:

> *"Retrieval method quality (BM25 → Dense → ELSER v2 → Hybrid) positively and monotonically correlates with generation faithfulness"*

Three specific problems:

1. **"Monotonically"** is falsified by a single pair where Dense < BM25 on any metric. Published literature shows Dense underperforms BM25 out-of-domain. This is likely, not exceptional.
2. **The ordering BM25 < Dense < ELSER is not guaranteed.** ELSER v2 is 18% better than BM25 on Elastic's own BEIR subset. On SheepSOC's private corpus, the gap may be smaller, reversed for some query types, or dominated by chunking artifacts.
3. **"Correlates"** is vague. It doesn't specify a direction claim, a metric, a model, or a query type — so it can't be falsified cleanly.

### Replacement: Five Specific, Falsifiable Hypotheses

Organized as primary and secondary, with the specific metric and condition that would falsify each.

---

**H1 (Primary — Overall Winner)**

> *Hybrid retrieval (BM25 + ELSER v2, RRF) achieves the highest mean Precision@5 and mean RAGAS Faithfulness across all 100 questions for both Mistral 7B and Gemma 3 12B, and these differences are statistically significant (Mann-Whitney, α=0.005 after Bonferroni correction for 10 pairwise comparisons) against at least the BM25 and Dense conditions.*

**What falsifies it:** Hybrid does not rank first on Precision@5, or differences are not significant against BM25 or Dense.

**Why this is defensible:** Multiple independent sources (Blended RAG IBM paper, Elastic's own BEIR evaluations, practitioner production data) support hybrid as the strongest method *on average*. The hypothesis is about average performance, not every query.

---

**H2 (Primary — Query-Type Interaction)**

> *For exact/entity query types, BM25 Precision@5 is not significantly lower than Hybrid Precision@5 (Mann-Whitney, α=0.05, one-tailed). For conceptual and multi-hop query types, BM25 Precision@5 is significantly lower than Hybrid Precision@5.*

**What falsifies it:** BM25 significantly underperforms Hybrid even on exact/entity queries, OR BM25 matches Hybrid on conceptual queries.

**Why this is defensible:** BM25's exact-match strength on named entities, identifiers, and technical terms is one of the best-established findings in IR. The interaction hypothesis is the core scientific contribution of this study — it explains *when* to use which method.

---

**H3 (Secondary — Model Size Effect)**

> *The absolute faithfulness gain from No-RAG to Hybrid is larger for Mistral 7B than for Gemma 3 12B (measured as: mean_faithfulness(Hybrid, Mistral) − mean_correctness(No-RAG, Mistral) > mean_faithfulness(Hybrid, Gemma) − mean_correctness(No-RAG, Gemma)).*

**What falsifies it:** Gemma 3 12B shows equal or larger absolute gain from retrieval.

**Why this is defensible:** A smaller model has less parametric knowledge, so its No-RAG baseline is lower. A rich retrieval context should provide proportionally more lift. However, if Gemma 3 12B is also better at utilizing context (larger models are), the effect is not guaranteed — making this a genuinely interesting empirical question.

---

**H4 (Secondary — ELSER vs Dense)**

> *ELSER v2 achieves higher mean nDCG@10 than Dense (nomic-embed-text) for at least two of the three query types (exact, conceptual, multi-hop).*

**What falsifies it:** nomic-embed-text matches or beats ELSER v2 on two or more query types.

**Why this is defensible:** ELSER v2 is a learned sparse encoder specifically trained for retrieval relevance on diverse English text. nomic-embed-text is a strong general-purpose embedding model. For domain-specific corpora, the gap between them is an empirical question. Importantly, **this hypothesis is falsifiable by a single well-run experiment** — which is exactly what SheepSOC is doing.

**Note:** This is the hypothesis most likely to surface a surprising result. If nomic-embed-text beats ELSER v2, that would be a meaningful finding about general embedding models vs. specialized sparse encoders for private corpora.

---

**H5 (Null Hypothesis for Sanity Checking)**

> *Among easy questions (30 of 100), all four retrieval methods achieve Precision@5 ≥ 0.80, and the difference between the worst and best method is less than 0.15.*

**What falsifies it:** Any method fails Precision@5 ≥ 0.80 on easy questions, or the spread exceeds 0.15.

**Why include a null hypothesis:** Easy questions are designed to be retrievable by any method. If BM25 can't find the answer to an easy question, something is wrong with the index, chunking, or the question itself — not with the retrieval method comparison. This hypothesis is a quality gate, not a scientific finding.

### Hypothesis Summary Table

| ID | Type | Metric | Prediction | Falsified If |
|---|---|---|---|---|
| H1 | Primary | Precision@5, Faithfulness | Hybrid ranks first overall | Hybrid not #1 or not significant |
| H2 | Primary | Precision@5 by query type | BM25 competitive on exact, weak on conceptual | BM25 weak on exact OR strong on conceptual |
| H3 | Secondary | Faithfulness gain | Mistral 7B gains more from retrieval | Gemma gains ≥ Mistral |
| H4 | Secondary | nDCG@10 | ELSER v2 > Dense on ≥2 query types | Dense ≥ ELSER on ≥2 query types |
| H5 | Null/QA | Precision@5 | All methods ≥ 0.80 on easy questions | Any method < 0.80 on easy questions |

---

## Phase 1 Build Order

This is the dependency graph translated into an ordered task list. Each artifact depends on those above it. Do not proceed to the next artifact until the current one is validated.

```
ARTIFACT MAP — Dependencies flow downward

[A1] Elasticsearch index schema
      ↓
[A2] Corpus JSONL (raw documents)
      ↓
[A3] Chunked corpus (chunks.jsonl)
      ↓
[A4] Indexed chunks (ES index with BM25 + Dense)
      ↓
[A5] ELSER v2 inference endpoint + re-indexed sparse fields
      ↓
[A6] Retrieval function validation (all 4 methods return K=10)
      ↓
[A7] Golden questions (golden_questions.jsonl) — manual work
      ↓
[A8] Judgment pool (candidate_pairs.csv) — generated from A4+A5+A7
      ↓
[A9] Qrels (qrels.jsonl) — manual labeling of A8
      ↓
[A10] Qrels validation (validate_qrels passes)
       ↓
[A11] RAGAS + HHEM local scoring validation
       ↓
[A12] Sanity checks pass (all 6 checks green)
       ↓
[A13] Full study run
```

### Ordered Task List With Acceptance Criteria

```
┌──────────────────────────────────────────────────────────────────────────┐
│ PHASE 1 BUILD ORDER — SheepSOC-RAG-001                                  │
│ Complete each task in order. Do not proceed until acceptance criterion   │
│ passes.                                                                  │
└──────────────────────────────────────────────────────────────────────────┘

ARTIFACT A1: Elasticsearch Index Schema
  Task:    PUT /sheepsoc-rag-001 with mappings from Fix 1 (dense_vector +
           sparse_vector + text fields + english analyzer)
  Command: curl -X PUT "localhost:9200/sheepsoc-rag-001" -H "Content-Type:
           application/json" -d @index_mapping.json
  ✅ Accept: curl "localhost:9200/sheepsoc-rag-001/_mapping" returns all
             four field types (text, keyword, dense_vector, sparse_vector)
  ⏱ Estimate: 15 minutes

ARTIFACT A2: Raw Corpus JSONL
  Task:    Download corpus (BEIR/SciFact, Wikipedia domain dump, or
           internal documents). Each line: {_id, title, text}
  ✅ Accept: wc -l corpus_raw.jsonl ≥ 1000
             grep -c '"_id"' corpus_raw.jsonl = line count (no malformed JSON)
  ⏱ Estimate: 30 minutes (download) or 2–4 hours (manual curation)

ARTIFACT A3: Chunked Corpus
  Task:    Run corpus_prep.py with chunk_size=400 words, overlap=80 words
  ✅ Accept: wc -l corpus/chunks.jsonl ≥ 3000 (expect ~4–10k chunks)
             All lines valid JSON with fields: doc_id, chunk_id, text,
             word_count, chunk_index
             mean word_count between 350 and 420 (log histogram check)
  ⏱ Estimate: 20 minutes

ARTIFACT A4: Indexed Chunks — BM25 + Dense
  Task:    Run indexing.py. This generates nomic-embed-text vectors for
           every chunk via Ollama and indexes into ES.
           WARNING: ~4,000–10,000 Ollama embedding calls. Use batching.
  ✅ Accept: es.count(index="sheepsoc-rag-001")["count"] == len(chunks.jsonl)
             Test BM25 and Dense queries both return 10 hits on 3 test queries
             Embedding vector for first document is 768-dimensional (not 0-d)
  ⏱ Estimate: 1–3 hours (GPU-accelerated embedding at ~200 chunks/min)

ARTIFACT A5: ELSER v2 Endpoint + Sparse Indexing
  Task:    Download ELSER v2 model via Kibana > ML > Trained Models.
           Create inference endpoint. Add ingest pipeline that runs ELSER
           inference on the "text" field and writes to "elser_sparse".
           Re-index all documents through the pipeline.
  Prerequisite: Elastic subscription or active trial (ELSER requires it).
  ✅ Accept: ELSER query on 3 test questions returns 10 hits
             ELSER top result overlaps < 80% with Dense top result
             (confirms methods are distinct)
  ⏱ Estimate: 2–4 hours (most time is ELSER inference at 26 docs/sec on
               Xeon CPUs — expect 2–6 hours for 5,000–10,000 chunks)

ARTIFACT A6: Retrieval Validation
  Task:    Run retrieval_validation.py — calls all four methods on 10 test
           queries and asserts each returns exactly K_RETRIEVE=10 results
  Code:    (run_sanity_checks() — checks 1 and 2 from the previous protocol,
           updated to assert len(results)==10)
  ✅ Accept: All four methods return exactly 10 results on all 10 test queries
             Hybrid top-5 is not identical to either BM25 or ELSER top-5
             All chunk_ids returned by each method exist in the index
  ⏱ Estimate: 30 minutes

ARTIFACT A7: Golden Questions
  Task:    Create 100 questions following schema in Fix 2.
           Use RAGAS synthetic generation as a bootstrap, then manually
           review and edit every question.
           Verify: 30 exact, 40 conceptual, 30 multi-hop.
           Verify: every question has a written reference_answer traceable
           to a specific document in the corpus.
  ✅ Accept: python validate_questions.py passes all checks:
             - exactly 100 questions
             - exactly {exact:30, conceptual:40, multi_hop:30}
             - no duplicate question_text
             - all reference_answers non-empty (> 50 chars)
             - all question_ids in format Q0001–Q0100
  ⏱ Estimate: 4–6 hours (cannot be automated; is the critical path)

ARTIFACT A8: Judgment Pool
  Task:    Run build_judgment_pool.py. Collects top-10 from all four methods
           for each of the 100 questions, unions them, exports
           candidate_pairs.csv for human labeling.
  ✅ Accept: candidate_pairs.csv exists with columns:
             question_id, chunk_id, chunk_text_preview, grade, labeler, notes
             Row count is between 1,500 and 3,000
             (100 questions × ~20 unique chunks each)
             All chunk_ids in the file exist in the ES index
  ⏱ Estimate: 1 hour (automated, requires A5+A6+A7)

ARTIFACT A9: Qrels File
  Task:    Two labelers independently grade every row in candidate_pairs.csv
           using the 0–3 rubric from Fix 2 / Section 4.
           Resolve disagreements: accept if |grade1 - grade2| ≤ 1 (take mean,
           round down). Discuss and re-label if |grade1 - grade2| ≥ 2.
           Export final labels as qrels.jsonl (one JSON line per row).
  ✅ Accept: Inter-rater agreement: Cohen's kappa ≥ 0.60 on ordinal grades
             Every question_id in golden_questions.jsonl appears in qrels
             Every question has ≥ 1 chunk with grade = 3
             Every question has ≥ 1 chunk with grade ≥ 2 (the "relevant" threshold)
  ⏱ Estimate: 12–20 hours of labeling (split across two labelers)
               This is the largest time investment in the entire study.

ARTIFACT A10: Qrels Validation
  Task:    Run validate_qrels(load_qrels("qrels.jsonl"), questions)
           Run inter-rater statistics on the dual-labeled subset.
  ✅ Accept: validate_qrels() prints "all have at least one relevant chunk"
             Cohen's kappa ≥ 0.60
             No question has zero grade-2 or higher chunks
             n_relevant_in_corpus ≥ 1 for every question
  ⏱ Estimate: 30 minutes

ARTIFACT A11: Local Scoring Stack Validation
  Task:    Run RAGAS with Gemma 3 12B as judge on 5 test examples.
           Run HHEM on 5 test examples.
           Confirm scores are in [0, 1] and HHEM returns valid labels.
  ✅ Accept: RAGAS faithfulness ∈ [0, 1] for all 5 test cases
             RAGAS returns null (not 0) when contexts=[] (No-RAG case)
             HHEM label ∈ {"consistent", "hallucinated"} for all 5 cases
             Total scoring time < 60 seconds for 5 examples (else too slow)
  ⏱ Estimate: 1 hour (includes troubleshooting RAGAS local LLM config)

ARTIFACT A12: Full Sanity Check Suite
  Task:    Run run_sanity_checks() — all 6 checks from the previous protocol,
           updated to use K_RETRIEVE=10 and the qrels loader.
  ✅ Accept: All 6 checks print ✅. Zero ❌ or ⚠️.
             If any check fails: diagnose, fix, and re-run before A13.
  ⏱ Estimate: 15 minutes

ARTIFACT A13: Full Study Run
  Task:    Run main_experiment.py. Logs progress to study_run.log.
           Writes results incrementally to results.jsonl (resumable).
  ✅ Accept: results.jsonl has exactly 1,000 lines (5 conditions × 2 models
             × 100 questions)
             No null response_text values (all models produced output)
             RAGAS scores populated for all RAG conditions
             RAGAS scores null for faithfulness/context_precision/
             context_recall in No-RAG rows (Fix 3 compliance check)
  ⏱ Estimate: 8–14 hours (run overnight)
```

### Phase 1 Timeline Summary

| Phase | Artifacts | Automated? | Estimated Hours |
|---|---|---|---|
| Infrastructure | A1–A3 | ✅ Mostly | 1–4 |
| Indexing | A4–A6 | ✅ Fully | 3–8 |
| Human data work | A7–A9 | ❌ Manual | 17–27 |
| Validation | A10–A12 | ✅ Fully | 2 |
| Study run | A13 | ✅ Fully | 8–14 |
| **Total** | | | **31–55 hours** |

The critical path is **A7 → A8 → A9** — golden question creation and qrels labeling. No amount of automation can substitute for human judgment here. Budget two full work sessions for A7 and two to three sessions across two labelers for A9.

The most likely failure points are:
- **A5**: ELSER v2 requires an Elastic subscription — verify this before starting
- **A9**: Inter-rater kappa below 0.60 — revise the rubric and re-label the disagreements before proceeding
- **A11**: RAGAS with a local LLM judge can produce unstable scores — if variance across identical inputs exceeds 0.05, switch to temperature=0 explicitly in the judge LLM config