---

# 🔬 Research Report: Just How Valuable is a Vector Search Database?
## A Strategic Research Agenda for the SheepSOC AI Laboratory

---

## Preamble: State of the Research Field

Before diving into the four questions, it is worth establishing how active this space is. A bibliometric snapshot counted more than 1,200 RAG-related papers on arXiv in 2024 alone, compared with fewer than 100 the previous year, underscoring the field's rapid maturation. The questions SheepSOC wants to explore are not fringe ideas — they are live research questions that enterprise teams and academic groups are actively publishing on. However, most published work uses cloud-scale hardware or API-dependent models. A self-hosted, multi-model, full-stack lab like SheepSOC occupies a genuinely interesting niche: it can run controlled, reproducible experiments that cloud-dependent setups cannot.

---

## Research Question 1: Can We Measure How Important Quality Search Results Are to Smaller Model Performance?

### What the Research Says

This question has been partially answered but deserves systematic, controlled measurement. The short answer: **retrieval quality is arguably the single most important variable in a RAG pipeline**, and smaller models are more sensitive to it than large ones.

The choice between sparse and dense retrieval — or a hybrid of both — has a larger impact on end-to-end quality than almost any other architectural decision. The difference isn't just a performance tradeoff: sparse and dense retrieval fail in completely different ways, excel on completely different query types, and require different infrastructure to operate.

The mechanism is well-understood. In the retrieval phase, the root causes of hallucinations in RAG include data source, query, retriever, and retrieval strategy problems. In the generation phase, causes include context noise, context conflict, and the "middle curse" in long contexts, alignment problems, and model capability boundary problems. In plain terms: bad retrieval poisons the well before the model even sees the question.

For small models specifically, the research is striking. A 2024 study on local models in radiology found that RAG eliminated hallucinations (0% vs 8%; p = 0.012) and improved mean rank by 1.3 when applied to a Llama 3.2 11B model — a locally deployable scale comparable to what SheepSOC runs. Among 100 cases, 54% showed marked improvement with RAG — most notably in safety-critical content — while an additional 33% demonstrated minor gains involving non-critical factual details. These improvements were concentrated in cases that required specialized knowledge of institutional protocols or recently updated guidelines.

More pointedly, small models are **worse** at handling irrelevant context. LLMs exhibit a certain degree of noise robustness, but they still struggle significantly in terms of negative rejection, information integration, and dealing with false information. "Negative rejection" means knowing when retrieved content is *wrong or irrelevant* and refusing to use it — something small models do poorly. This creates a real risk: **bad search results can actively make a small model worse than no RAG at all.**

There is also the mechanistic finding that hallucinations in RAG occur when the Knowledge FFNs in LLMs overemphasize parametric knowledge in the residual stream, while Copying Heads fail to effectively retain or integrate external knowledge from retrieved content. This implies that when retrieval quality drops, a model literally falls back to guessing from its training data — and the smaller the model, the less training data it has to fall back on.

### Experiments SheepSOC Can Run

The **RAG Triad** is your primary measurement framework. The RAG Triad comprises three metrics — context relevance, groundedness, and answer relevance — that measure how well each step of a RAG system is performing. Common failure modes, including poor retrieval quality, hallucination and irrelevant answers, can all be traced back to the interactions between query, retrieved context, and generated output.

**Concrete test design:**
1. Build a **golden dataset** of 50–100 question/answer pairs from a domain you control (e.g., local documentation, a technical manual, a historical dataset).
2. For each question, run the same model (start with Mistral 7B) under four conditions: (a) no RAG, (b) BM25-only retrieval, (c) dense vector-only, (d) hybrid retrieval.
3. Score each response on groundedness (does the answer match the retrieved chunk?), answer relevance, and hallucination rate.
4. Use **RAGAS** (open-source, runs locally) as your evaluation harness — RAGAS is an open-source framework for evaluating retrieval and generation, with built-in synthetic data generation.
5. Repeat across all three of your chat models (Mistral 7B, Llama 2 13B, Gemma 3 12B) to isolate the model-size variable.

**Key metric to watch:** hallucination rate as a function of retrieval precision@K. If you can show a dose-response curve — "as retrieval quality drops, hallucination rate rises, and this effect is steeper for smaller models" — that is a publishable, reproducible finding specific to your hardware and model set.

---

## Research Question 2: How Much More Value Can Be Extracted From Smaller Models With Proper Search Results?

### What the Research Says

This is perhaps the most practically exciting question, and the published evidence is strong. The headline finding: **a well-retrieval-augmented small model can match or exceed a much larger model running without retrieval.**

The clearest demonstration comes from a 2024 enterprise deployment study: A 3B RAG fine-tuned model is competitive even with the 15.5B version without RAG on key metrics, while keeping hallucination low. This is a key lesson: we could deploy a 3B RAG fine-tuned model if we had more limited infrastructure.

This 5x parameter efficiency gain is not an outlier. In medical question answering, specialist retrievers enhance GPT-3.5's medical question benchmark performance to reach GPT-4 levels. That is a commercial demonstration of a smaller/cheaper model equaling a larger/more expensive one purely through retrieval quality improvements.

The underlying intuition is well-captured by Vectorize's production observations: Think of it like giving a subject matter expert clear, relevant reference materials versus giving a general expert a mountain of loosely related documents. The subject matter expert with focused materials will likely provide better answers, even though the general expert might have broader knowledge. The future of RAG isn't about using the biggest models — it's about building smarter pipelines that make efficient use of smaller, more cost-effective models.

Open-source models combined with RAG can offer competitive performance at a significantly reduced cost compared to larger models. This enables organizations with limited budgets to deploy high-performing AI solutions, making advanced technology accessible beyond large corporations.

There is a hard floor, though. The smallest RAG fine-tuned model (1B) hallucinates significantly more than its larger counterparts. Among the other three variants, the 7B version gives us the best trade-off, as the performance difference between 7B and 15.5B is marginal. For SheepSOC, this is encouraging: the 7B–13B range (Mistral 7B, Llama 2 13B, Gemma 3 12B) appears to be the sweet spot where RAG yields maximum relative gains.

Research on Gemma models specifically found that RAG integration consistently reduced latency by up to 17% while completely eliminating factual hallucinations when responding to user-specific queries. And more broadly, in fast-evolving fields, incorporating RAG models has reduced outdated responses by 15–20% compared to traditional LLMs.

### Experiments SheepSOC Can Run

1. **Model-vs-RAG tradeoff ladder:** Run the same factual question set through five configurations and plot accuracy vs. compute cost: Mistral 7B (no RAG) → Mistral 7B + BM25 → Mistral 7B + hybrid → Llama 2 13B (no RAG) → Gemma 3 12B (no RAG). The goal is to find the crossover point where the smaller model with RAG beats the larger model without it.

2. **Context sensitivity test (Needle-in-a-Haystack):** The Needle-In-A-Haystack evaluation inserts a piece of information (the "needle") into a document (the "haystack") and evaluates the model's ability to recall the inserted information. It is a great test bench for estimating the performance of a model in a RAG or memory system. Your Jupyter environment is ideal for this — vary context length and "needle" depth across all three models.

3. **Domain specificity ramp:** Start with a general knowledge corpus, then progressively add domain-specific documents. Measure how much each model's answer quality improves per document added. This tests whether smaller models gain proportionally *more* from domain context than larger ones (since they have less parametric domain knowledge to begin with).

---

## Research Question 3: Explore the Search Landscape — BM25, ELSER, ELSER2, Dense Vector, and Hybrid

### The Search Method Taxonomy

This is an area with rich published benchmarks. Here is the authoritative landscape, ordered from simplest to most sophisticated:

**BM25 (Sparse Keyword Search)**
Sparse retrieval with BM25 is fast and cheap. Inverted indexes like Elasticsearch and OpenSearch handle billions of documents on commodity hardware, support real-time updates, and return results in single-digit milliseconds. There's no GPU requirement, no embedding model to maintain, and no approximate nearest neighbor index to rebuild when documents are updated. BM25's strength is precision on exact matches. BM25 wins on precision for named entities, product SKUs, and technical jargon. Its weakness is the vocabulary mismatch problem: BM25 suffers from the "vocabulary mismatch problem" — synonyms like "Car" vs. "Automobile," polysemy like "Apple" (the fruit) vs. "Apple" (the tech giant), and typos.

**Dense Vector Search (HNSW/KNN — what SheepSOC already runs)**
Dense retrieval represents both queries and documents as fixed-dimensional embedding vectors, typically 768 or 1024 dimensions, produced by a transformer encoder. At query time, the query is encoded to a vector and the system finds the documents whose embedding vectors are most similar to the query vector. This is what SheepSOC's current nomic-embed-text + HNSW pipeline does. Dense search wins conceptual queries where paraphrasing is common, but critically: if a model is not well adapted to your specific data, it's very likely that using kNN and dense models would degrade your retrieval performance in comparison to BM25. Out-of-domain dense models can be actively harmful.

**ELSER (Elastic Learned Sparse EncodeR)**
ELSER is a retrieval model trained by Elastic that enables you to perform semantic search based on contextual meaning and user intent, rather than exact keyword matches. ELSER is an out-of-domain model which means it does not require fine-tuning on your own data, making it adaptable for various use cases out of the box. It occupies a middle ground between BM25 and dense: sparse vectors generated by models like ELSER can generalize well to new domains without requiring extensive fine-tuning. Unlike dense vector models that often need domain-specific training, sparse vectors rely on term-based representations, making them more effective for zero-shot retrieval.

**ELSER v2 — The Significant Upgrade**
The jump from ELSER v1 to v2 is substantial and directly relevant to SheepSOC's Xeon hardware. ELSER V2 produces significantly higher quality embeddings than ELSER V1. Comparing against BM25: ELSER V2 has 10 wins, 1 draw, 1 loss and an average improvement in NDCG@10 of 18%. On speed: ELSER v2 achieved between a 60% and 120% speed-up in inference compared to ELSER v1 by upgrading the libtorch backend and optimizing for x86 architecture. Throughput-wise: the optimized V2 model ingested at a max rate of 26 docs/s, compared with the ELSER V1 max rate of 14 docs/s, resulting in a 90% increase in throughput. **Critical note for SheepSOC:** The optimized model is designed to work best on newer Intel CPUs, but it works on AMD CPUs as well. It is recommended to use the new optimized Linux-x86-64 model for all new users of ELSER. Your dual Xeon E5-2680 v4 + Ubuntu 24.04 x86-64 setup is precisely the target platform for the ELSER v2 optimized build. ELSER v2 is also currently in the top-10 models on MTEB for Retrieval when grouping together multiple flavors of the same competitor family, filtering for models under 250 million parameters — impressive for an on-premises model with no API dependency.

**Hybrid Search: The Performance Champion**

The numbers here are unambiguous across multiple independent studies:

Run BM25 and vector search concurrently, fuse with reciprocal rank fusion, and you jump from 65–78% recall to 91% recall@10. Dense-only retrieval hits 78% recall@10. Sparse-only BM25 lands at 65%. Hybrid search reaches 91% recall@10. That gap is the difference between a production-ready RAG system and one that hallucinates on edge cases.

This is not a single study's result — it replicates across benchmarks. A key IBM Research paper specifically tested BM25, KNN, ELSER, and combinations using Elasticsearch and found: using three-way retrieval is the optimal option for RAG — meaning BM25 + dense vectors + sparse vectors together outperforms any two-way combination.

Blended Retrievers achieved an NDCG@10 score of 0.87, which marks an 8.2% increment over the benchmark score of 0.804 established by the COCO-DR Large model.

The overhead cost of hybrid is negligible in practice: Running two retrievers in parallel adds roughly 6ms to p50 latency versus dense-only search. In most RAG pipelines, LLM inference already dominates at 500ms to 2 seconds, so hybrid retrieval's overhead is essentially noise.

The standard fusion algorithm is **Reciprocal Rank Fusion (RRF)**. Running a hybrid search query boils down to sending a mix of at least one lexical search query and one semantic search query and then merging the results. Luckily, there are several existing methods: Convex Combination (CC) and Reciprocal Rank Fusion (RRF). Elasticsearch already supports this natively: if you are already on Elasticsearch, RRF works out of the box since v8.9.

### Important Caveat: Dense Models Out-of-Domain

One finding that deserves emphasis: dense models that outperform BM25 on in-domain datasets are usually *worse* out-of-domain. If a model is not well adapted to your specific data, it's very likely that using kNN and dense models would degrade your retrieval performance in comparison to BM25. This is a trap for labs that benchmark on publicly available datasets and then expect the same results on their own document collections. SheepSOC should test all search methods on *its own* corpus, not just published benchmarks.

### Experiments SheepSOC Can Run

SheepSOC has every component needed to run the definitive comparison across all four methods on the same Elasticsearch instance:

1. **Build four parallel indices** for the same document corpus: BM25 only, dense vector only (nomic-embed-text), ELSER v2, and hybrid (BM25 + ELSER v2 via RRF).
2. **Run the same 100-question golden dataset** against all four indices and score using precision@5, recall@10, and nDCG@10.
3. **Feed top-K retrieved chunks** from each method into Mistral 7B and measure final answer quality (RAGAS groundedness + answer relevance scores).
4. **Vary K** (top-3, top-5, top-10 results) to understand diminishing returns from more context.
5. **Segment by query type:** exact-match queries (entity names, codes) vs. conceptual queries ("what is the policy on X"). BM25 should win the first category; dense should win the second; hybrid should win overall.

This is a publishable experiment design that maps directly to the Blended RAG paper methodology.

---

## Research Question 4: Is a Large Vector Dataset Helpful in Expanding the Basic Knowledge of Smaller Models?

### What the Research Says

This is the foundational value proposition of RAG, and the evidence is strong: **yes, a vector database acts as a runtime knowledge extension for any model, including small ones, and larger/more complete knowledge bases produce measurably better outputs.**

The core mechanism: RAG enhances LLMs by incorporating an information-retrieval mechanism that allows models to access and utilize additional data beyond their original training set. When new information becomes available, rather than having to retrain the model, all that's needed is to augment the model's external knowledge base with the updated information. Typically, the data to be referenced is converted into LLM embeddings, numerical representations in the form of a large vector space.

By redesigning the language model with the retriever in mind, a 25-time smaller network can get comparable perplexity as its much larger counterparts. That is the extreme end of what retrieval-grounded models can achieve — a 25x parameter reduction while maintaining output quality — though this requires training from scratch (the RETRO architecture), which is beyond SheepSOC's scope. But the principle applies to inference-time RAG: the vector store is acting as an external memory that compensates for the model's limited internal knowledge.

Vector DBs act as external knowledge bases or external memory of LLMs. They enable low-latency similarity search using approximate nearest neighbor search algorithms over the vector space, providing the retrieved knowledge efficiently to the LLMs for high-quality generation.

There is an important nuance here about *what kind* of knowledge matters. RAG is not simply "more documents = better." The key finding from enterprise deployments: one of RAG's biggest advantages is that it integrates the most current data into the LLM's decision-making process. By connecting directly to updated databases, RAG ensures that the information being used is the latest available, which is vital for applications where timeliness is critical. The value is not in volume alone but in domain specificity and recency — exactly what a private vector corpus provides.

Research has also identified what kind of corpus benefits small models most. RAG was shown to improve the factual accuracy of LLMs for domain-specific and time-sensitive queries related to private knowledge bases. The research also revealed the limitations of fine-tuning LLMs with small-scale and skewed datasets. In other words, large, balanced, domain-specific corpora are the gold standard.

For the question of *corpus size specifically*, the RETRO model research provides the clearest signal: retrieval models scale with corpus size in a way that parametric models scale with parameter count. You effectively get more "knowledge" per dollar by growing your vector store than by upgrading your model — at least in knowledge-intensive domains.

### Important Tension: Noise vs. Coverage

Bigger corpora can introduce noise. RAG was initially primarily regarded as a means to fill knowledge gaps left from the pre-training stage, but subsequent research has shown that it has broader value: correcting internal model errors. However, RAG systems may retrieve factually correct but misleading sources, leading to errors in interpretation. A large corpus with poor chunking strategy or low-quality documents can actually hurt performance.

RAG systems may retrieve factually correct but misleading sources, leading to errors in interpretation. When faced with conflicting information, RAG models may struggle to determine which source is accurate. This is more pronounced in small models than large ones, since larger models are better at detecting when retrieved context conflicts with their parametric knowledge.

### Experiments SheepSOC Can Run

1. **Corpus size scaling experiment:** Start with a 100-document corpus on a specific domain. Measure answer accuracy. Double it to 200, then 500, then 1,000, then 5,000 documents. Plot answer quality vs. corpus size. You expect a positive but diminishing returns curve — understanding where returns plateau is valuable.

2. **Corpus quality vs. quantity tradeoff:** Split a large corpus into a "curated" high-quality subset (clean, relevant, well-chunked) vs. the raw full corpus. Compare retrieval quality and final answer quality. This tests whether cleaning 20% of documents beats having 5x more raw documents.

3. **Domain transfer experiment:** Index a large general knowledge corpus (e.g., Wikipedia dumps) and measure whether it improves performance on domain-specific queries compared to a small, targeted domain corpus. Small models should benefit more from domain-specific corpora than general ones.

4. **Chunking strategy impact:** nomic-embed-text operates at 768 dimensions with Elasticsearch's HNSW. Test different chunk sizes (128 tokens, 256 tokens, 512 tokens) and overlap percentages against the same golden question set. The literature suggests 256-token chunks with 10–20% overlap as a starting baseline.

---

## Cross-Cutting Recommendation: How to Run All Four Questions Together

SheepSOC's setup is ideally suited for a **factorial experiment design** that answers all four questions simultaneously with a single unified test harness:

| Variable | Levels to Test |
|---|---|
| **Model** | Mistral 7B, Llama 2 13B, Gemma 3 12B |
| **Retrieval method** | None, BM25, Dense (nomic), ELSER v2, Hybrid |
| **Corpus size** | Small (100 docs), Medium (1K docs), Large (10K docs) |
| **Query type** | Exact/entity, Conceptual/paraphrase, Multi-hop reasoning |

Your **251 GB of RAM** means you can hold all indices in memory simultaneously. Your **56 logical cores** enable running multiple retrieval paths in parallel. Your **RTX 5060 Ti** handles the inference. Your **Jupyter environment** is the analysis layer. This is, frankly, a better-instrumented testbed than many published papers used.

---

## Confidence & Gaps

**High confidence findings:**
- Hybrid search (BM25 + dense or sparse) consistently outperforms either method alone by 15–30% on recall, across multiple independent studies
- ELSER v2 delivers an average 18% improvement in nDCG@10 over BM25 and is the practical best choice for English zero-shot retrieval on Elasticsearch
- A small model with good retrieval can match or beat a model 3–5× its size without retrieval
- Hallucination rates are substantially reduced by RAG in small models, particularly for domain-specific knowledge gaps

**Moderate confidence:**
- The exact performance advantage of large vs. small corpora is underexplored at SheepSOC's scale range (thousands, not millions of documents). Most studies go very small or very large, skipping the mid-range.
- Whether out-of-domain dense vector search is consistently worse than BM25 depends heavily on the embedding model and corpus. nomic-embed-text specifically has not been benchmarked on this question in published work against ELSER v2.

**Gaps and open questions:**
- There is surprisingly little published work on the *exact* dose-response relationship between retrieval quality (measured by nDCG) and final answer quality (measured by hallucination rate) *specifically for 7B–13B models* — the range SheepSOC runs. Most papers study either very small (<3B) or very large (>70B) models.
- The "negative rejection" capability (knowing when retrieved context is bad) in small models is understudied. The RGB benchmark from 2023 raised this issue but it hasn't been systematically quantified across the Ollama model library.
- ELSER requires an Elastic subscription for full production use. SheepSOC should verify whether the trial/developer license covers the intended research workload — this is an operational gap, not a research gap, but it matters for the experiment plan.