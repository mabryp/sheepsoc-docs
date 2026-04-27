---

# Precision Audit: Citations, Metrics, Hallucination, and Corpus Quality

This is a methodological deep-dive in response to the four follow-up questions. The goal is to separate what is actually proven from what is loosely cited, and to give SheepSOC a concrete, defensible measurement framework.

---

## Part I: Citation Audit — Exact Sources for Every Quantitative Claim

### Claim A: "0% vs 8% hallucination reduction"

**✅ VERIFIED — Primary Source Confirmed**

**Full Citation:**
Wada A, Tanaka Y, Nishizawa M, et al. *"Retrieval-augmented generation elevates local LLM quality in radiology contrast media consultation."* **NPJ Digital Medicine** 8:395 (July 2025).
- DOI: https://doi.org/10.1038/s41746-025-01802-z
- PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC12223273/

**What it actually measured:** In 100 synthetic iodinated contrast media consultations, RAG eliminated hallucinations (0% vs 8%; χ²₍Yates₎ = 6.38, p = 0.012) in the Llama 3.2-11B model. Qualitative assessment by a radiologist revealed that hallucinations — particularly in contrast dosing and contraindication recommendations — were present in 8% of responses from the base Llama model and were eliminated (0%) in the RAG-enhanced version.

**Is it benchmark-specific or generalizable?**
**Benchmark-specific.** Critical constraints: (1) 100 *synthetic* (not real) clinical scenarios; (2) a highly curated, expert-reviewed knowledge base of 66 chunks averaging 337 characters each; (3) hybrid search was used (semantic + keyword); (4) hallucination was defined by a *single* board-certified radiologist as "clinically incorrect or guideline-inconsistent statements that could alter patient management." The paper's own authors note: *"zero hallucinations indicates absence of clinically dangerous misinformation rather than perfect clinical responses."* The 0% figure is a ceiling of a constrained experiment — it cannot be transplanted to general-domain RAG without re-measurement. The 8% baseline, however, is consistent with other small-model hallucination rates in domain-specific medical tasks.

---

### Claim B: "3B model matching 15.5B with RAG"

**✅ VERIFIED — Primary Source Confirmed**

**Full Citation:**
Béchard P, Marquez Ayala O. *"Reducing hallucination in structured outputs via Retrieval-Augmented Generation."* **NAACL 2024 Industry Track**, 2024.naacl-industry.19.
- arXiv: https://arxiv.org/abs/2404.08189
- DOI: https://doi.org/10.18653/v1/2024.naacl-industry.19

**What it actually measured:** The smallest RAG fine-tuned model (1B) hallucinates significantly more than its larger counterparts. Among the other three variants, the 7B version gives the best trade-off, as the performance difference between 7B and 15.5B is marginal. A 3B version trained with RAG is competitive even with the 15.5B version without RAG on the Trigger EM and Bag of Steps metrics, while keeping hallucination low.

**Is it benchmark-specific or generalizable?**
**Highly benchmark-specific.** This was an enterprise workflow generation task using *StarCoderBase* variants — code-pretrained models (1B, 3B, 7B, 15.5B parameters) producing structured JSON-like workflow outputs, not free-text answers. The evaluation metrics ("Trigger Exact Match," "Bag of Steps") are proprietary task metrics, not standard NLP benchmarks. The principle — that RAG can substitute for parameter count — is directionally supported by other papers (including the RETRO architecture research, which showed a 25× parameter reduction with retrieval), but the exact multiplier "3B vs 15.5B" comes from this single domain-specific study on structured outputs. **Do not assume this ratio transfers directly to free-text QA with Mistral 7B or Llama 2 13B.**

---

### Claim C: "91% recall@10 hybrid result"

**⚠️ PARTIALLY VERIFIED — Not from a Primary Academic Paper**

The 91% recall@10 figure appears in multiple blog posts and practitioner guides but traces back to **no single peer-reviewed paper** as its origin. Here is what each source actually is:

- The supermemory.ai hybrid search guide states: "Dense-only retrieval hits 78% recall@10. Sparse-only BM25 lands at 65%. Hybrid search reaches 91% recall@10." This is a practitioner blog post, not a peer-reviewed paper.

- A Medium article by Bronckers reports "91% retrieval accuracy, up from 62% with dense-only search" — but this is a 4-stage cascade (BM25 + FAISS vectors + cross-encoder reranker) across 159 multilingual automotive documents, measured as custom "retrieval accuracy," not recall@10. The full cascade with reranking achieves 91%; hybrid without reranking achieves 79%.

**The closest academic source** is the IBM Research paper "Blended RAG" (arXiv:2404.07220, Elasticsearch Labs confirmed): Blended Retrievers achieved an NDCG@10 score of 0.87, which marks an 8.2% increment over the benchmark score of 0.804 established by the COCO-DR Large model. That is a credible, peer-reviewed number, but measured in NDCG@10 on SQuAD/BEIR-style tasks, not recall@10 in production.

The broader consensus — run BM25 and vector search concurrently, fuse with reciprocal rank fusion, and you jump from 65–78% recall to 91% recall@10 — is cited by multiple practitioners but sources like "Cormack et al. SIGIR 2009" and "Bruch et al. ACM TOIS 2023" are cited without the specific recall numbers being traceable to those papers.

**Bottom line for SheepSOC:** The *direction* (hybrid > either method alone) is academically solid. The *exact number* (91%) is a practitioner consensus figure that varies by corpus, embedding model, query type, and K. You should **measure your own recall on your own corpus** rather than inheriting this number as a design target. Also worth knowing: a hybrid search configuration with a badly tuned alpha or k parameter can underperform your dense baseline. Measure before and after. Do not assume hybrid is automatic improvement.

---

### Claim D: "18% nDCG improvement for ELSER v2"

**✅ VERIFIED — But Source is Elastic's Own Internal Benchmark**

**Full Citation:**
Elastic. *"ELSER Benchmarks."* Official Elastic Documentation, Elastic Stack 8.11+.
- URL: https://www.elastic.co/docs/explore-analyze/machine-learning/nlp/ml-nlp-elser
- Also: https://www.elastic.co/search-labs/blog/elastic-learned-sparse-encoder-elser-retrieval-performance

ELSER V2 produces significantly higher quality embeddings than ELSER V1. The table below shows the performance of ELSER V2 compared to BM25. ELSER V2 has 10 wins, 1 draw, 1 loss and an average improvement in NDCG@10 of 18%.

**Is it benchmark-specific or generalizable?**
**Internally validated — use with calibrated skepticism.** This number comes from Elastic's own BEIR subset, evaluated by the model's creators. The BEIR benchmark itself is a recognized standard (Thakur et al., NeurIPS 2021), but Elastic chose which BEIR sub-datasets to include. ELSER v2 remains in the top-10 models on MTEB for Retrieval when grouping together multiple flavors of the same competitor family among models under 250M parameters. A common question is how ELSER compares with relevance achievable with traditional keyword (BM25) retrieval or other models, including OpenAI's text-embedding-ada-002. Third-party confirmation on MTEB is more credible than self-reported BEIR numbers. For SheepSOC's specific corpus (private documents, potentially technical/domain-specific English), the 18% figure may be higher or lower — **you need to measure it on your own data.**

---

## Part II: Formally Defining Retrieval Quality

### The Four Core Metrics and What They Measure

Core retrieval metrics include precision@k (are the top-k documents relevant?), recall@k (how much of the relevant info was retrieved?), Mean Reciprocal Rank (MRR) (are correct docs ranked early?), and Normalized Discounted Cumulative Gain (nDCG) (graded relevance with position weighting).

Each metric captures a different failure mode:

| Metric | What It Measures | RAG Failure Mode It Detects |
|--------|-----------------|----------------------------|
| **Precision@K** | Signal-to-noise ratio of retrieved context | Context stuffing with irrelevant chunks → hallucination |
| **Recall@K** | Coverage: did you find all relevant material? | Knowledge gaps → incomplete/vague answers |
| **MRR** | Is the *first* relevant result ranked highly? | Burying the right answer → model ignores it |
| **nDCG@K** | Graded relevance + position weighting | Best overall ranking quality indicator |

### Which Metric Correlates Most with Hallucination?

The most direct evidence comes from the RAGCHECKER paper (NeurIPS 2024, Proceedings of the Datasets and Benchmarks Track): The quality of retrieval is crucial, as evidenced by the notable differences in overall Precision, Recall and F1 scores when comparing BM25 with E5-Mistral with the generator fixed. This improvement is agnostic to the specific choice of generator, suggesting a consistent benefit from employing a better retriever.

For hallucination *specifically*, the proximate metric is **context precision** (equivalently, Precision@K): In RAG systems, the consequences of poor precision or recall are magnified. Irrelevant context wastes limited token space and confuses the LLM, increasing the chance of hallucination. Low precision introduces noise, forcing the LLM to sift through irrelevant information, which can lead to "context-stuffing" where the model incorrectly synthesizes unrelated facts.

The causal chain is: ↓ Precision@K → ↑ irrelevant context injected → ↑ hallucination rate.

The complementary chain for *completeness* failures (different from hallucination): ↓ Recall@K → ↑ knowledge gaps → ↑ vague/generic responses.

Low precision means you're stuffing the context window with noise. The LLM has to work harder to find the signal, increasing hallucination risk. Low recall means you're missing important information. The answer might be technically accurate but incomplete.

### Should Retrieval Quality Be a Scalar or a Vector?

**A vector, definitively.** A single scalar collapses information that you need to separate to diagnose system failures. Precision and recall trade off against each other — you can engineer high recall at the cost of precision (and vice versa). The RAG Triad operationalizes this naturally: The RAG Triad is made up of 3 evaluations: context relevance, groundedness, and answer relevance. To verify groundedness, we can separate the response into individual claims and independently search for evidence that supports each within the retrieved context. By reaching satisfactory evaluations for this triad, we can make a nuanced statement about the application's correctness; our application is verified to be hallucination free up to the limit of its knowledge base.

**Recommended SheepSOC retrieval quality vector:**

| Dimension | Primary Metric | Secondary | Maps To (Generation Side) |
|-----------|---------------|-----------|---------------------------|
| Signal-to-noise | Precision@5 | Context Precision (RAGAS) | Faithfulness / hallucination rate |
| Coverage | Recall@10 | Context Recall (RAGAS) | Answer completeness |
| Ranking quality | nDCG@10 | MRR | Overall answer quality |
| Latency | P95 retrieval ms | — | System usability |

At analysis time, look for **differential effects**: if you change from BM25 to hybrid and Precision@5 improves but Recall@10 stays flat, you've reduced hallucination without improving coverage — valuable to know. A scalar would hide this.

---

## Part III: Operationalizing Hallucination

### What "Hallucination" Actually Means — Three Distinct Phenomena

The literature conflates three different things under "hallucination." SheepSOC needs to pick a definition and stick to it:

**Type 1 — Context Contradiction:** The model asserts something that directly conflicts with the retrieved context. This is measurable without a reference answer. To operationalize faithfulness, natural language inference (NLI) pipelines decompose responses into atomic claims, then classify each as entailed, neutral, or contradicted by the context using models like DeBERTa. Aggregation offers a faithfulness score as a proportion of entailed claims, with thresholds above 0.9 indicating that outputs are "grounded."

**Type 2 — Unsupported Claims:** The model asserts something not present in the retrieved context, but not necessarily false. Faithfulness Score = Number of claims in the response supported by the retrieved context / Total number of claims in the response. A score of 0.67 means 1 in 3 claims has no grounding.

**Type 3 — Factual Incorrectness:** The model's output is wrong relative to ground truth, regardless of what was retrieved. This requires a reference answer and is the hardest to measure automatically.

For SheepSOC's research purposes, **Type 1 and Type 2 are the actionable ones** — they are directly caused by retrieval quality and are measurable without expensive human annotation.

### Evaluation Stack: RAGAS + Supplements

RAGAS (Retrieval Augmented Generation Assessment) is a framework for reference-free evaluation of RAG pipelines. With RAGAS, a suite of metrics can be used to evaluate these different dimensions without having to rely on ground truth human annotations.

**Critical implementation note for SheepSOC:** RAGAS by default uses OpenAI's API for LLM-as-judge scoring. You can configure it to use a local LLM via `llm_factory`. Your **Gemma 3 12B** running through Ollama is a viable judge model for a local-only pipeline. Vectara's HHEM-2.1-Open is a classifier model (T5) that is trained to detect hallucinations from LLM generated text. This model can be used in the second step of calculating faithfulness. The model is free, small, and open-source, making it very efficient in production use cases. HHEM is an excellent complement — it runs on CPU and doesn't require an LLM call.

**Recommended evaluation stack for SheepSOC:**

| Layer | Tool | What It Measures | Notes |
|-------|------|-----------------|-------|
| **Primary automated** | RAGAS | Faithfulness, Context Precision, Context Recall, Answer Relevance | Configure with local Gemma 3 12B as judge |
| **Hallucination classifier** | HHEM-2.1-Open (Vectara) | Binary hallucination detection per claim | CPU-only, fast, open-source |
| **Retrieval metrics** | Custom Jupyter notebook | Precision@K, Recall@K, nDCG, MRR | Against your golden dataset |
| **Ground truth validation** | Human review (subset) | Factual correctness of Type 3 | 20–30 questions, not full dataset |
| **LLM-as-judge spot check** | Gemma 3 12B | Relevance scoring for ambiguous cases | Cross-validate with RAGAS |

### Handling Partial Correctness

This is a real problem. A response can be 80% correct with one fabricated claim. RAGAS handles this through the atomic claims decomposition: Faithfulness metrics catch cases like this. Claims are checked one by one: "Python was created by Guido van Rossum" → Supported ✓; "Python was created in 1991" → Supported ✓; "Python is the most popular programming language" → NOT FOUND ✗. Faithfulness = 2/3 = 0.67.

For SheepSOC's research specifically, the recommendation is: **report faithfulness as a continuous score (0–1), not binary pass/fail.** This lets you compute meaningful statistics across retrieval conditions (e.g., "mean faithfulness drops from 0.87 with hybrid to 0.71 with BM25-only for conceptual queries"). Binary pass/fail at an arbitrary threshold loses the gradient.

---

## Part IV: Defining Corpus Quality

### The Five Dimensions of Corpus Quality

A high-quality corpus for RAG is not merely "more documents." Research supports five independent axes, each of which must be defined and measured separately:

**1. Topical Purity**
Does the corpus stay on-topic for the query domain? Off-topic documents get retrieved and introduce noise. Some documents you ingest into your corpus may not be useful for your agent, either because they are irrelevant to its purpose, too old or unreliable, or because they contain problematic content. A baseline deduplication step can select the documents to keep, but you may want to select the "best" version of the duplicate using other logic such as most recently updated, publication status, or most authoritative source.

**2. Deduplication**
Deduplication: analyze the documents to identify and eliminate duplicates or near-duplicate documents. If you pull from one or more shared drives, multiple copies of the same document could exist in multiple locations. Some of those copies may have subtle modifications. If these duplicates remain in your corpus, you can end up with highly redundant chunks in your final index that can decrease the performance of your application.

**3. Chunking Quality — The Underrated Variable**
This is larger than most labs appreciate. A Vectara study published at NAACL 2025 (arXiv:2410.13070) tested 25 chunking configurations with 48 embedding models and found that chunking configuration had as much or more influence on retrieval quality as the choice of embedding model.

The recommended starting point: Two failure modes dominate. Chunks that are too small lose context. Chunks that are too large dilute relevance. The practical starting range is 256 to 512 tokens with 10 to 25% overlap. Microsoft Azure recommends 512 tokens with 25% overlap (128 tokens) as a starting point, using BERT tokens rather than character counts. Validate against your own corpus before production.

There is also an inherent tension: A common source of inaccurate or unstable answers lies in a structural conflict within the traditional "chunk-embed-retrieve" pipeline. For high-precision recall in semantic similarity search, smaller chunks (e.g., 100–256 tokens) are needed. To provide sufficiently complete and coherent context for the LLM to generate high-quality answers, larger chunks (e.g., 1024+ tokens) are needed to ensure logical completeness and sufficient background. This forces system designers into a difficult trade-off between "precise but fragmented" and "complete but vague."

**4. Recency**
Documents that were accurate in 2020 may be incorrect in 2025. For SheepSOC's research, you should tag every document with a creation/update date and test whether retrieval of outdated documents measurably increases hallucination rates for time-sensitive queries.

**5. Formatting and Metadata Richness**
A clean corpus improves both retrieval accuracy and embedding quality. Use parsing libraries to extract only the primary content from web pages or PDFs, and remove boilerplate, navigation links, and disclaimers. Each chunk should carry metadata such as topic, date, and source, so that retrieval can filter and rank results more effectively.

### How to Quantify Corpus Noise

Corpus noise is not one number — it is a property of the corpus-query pair. A document is "noise" for some queries and "signal" for others. The operational definition for SheepSOC:

**Corpus Noise Score for query Q** = (number of retrieved chunks for Q with relevance < threshold) / K

Where threshold is set empirically from your golden dataset. This is equivalent to 1 - Precision@K, but made explicit as a corpus property rather than a system property.

### Simulating Noise Systematically

The RGB Benchmark (Chen et al., AAAI 2024, arXiv:2309.01431) is the best-studied framework for structured noise testing. It evaluates LLMs in 4 fundamental abilities required for RAG: noise robustness, negative rejection, information integration, and counterfactual robustness. SheepSOC can implement all four of these as corpus injection experiments:

| Noise Type | How to Simulate | What It Tests | Expected Result |
|-----------|----------------|---------------|-----------------|
| **Irrelevant documents** | Add off-topic documents to corpus | Precision degradation, noise robustness | Faithfulness drops; small models more impacted |
| **Conflicting documents** | Insert contradictory version of a fact (e.g., two policy documents with opposite rules) | Counterfactual robustness, information integration | Models struggle; small models more confused |
| **Outdated documents** | Mix current and superseded versions of the same document | Temporal reasoning, negative rejection | Models often fail to identify which version is authoritative |
| **Redundant documents** | Add 5 near-duplicate chunks of the same fact | Context utilization efficiency | May improve coverage but wastes context window; check faithfulness |

**The key finding to watch for:** Real-world deployment will require handling heterogeneous document quality, conflicting evidence, and near-duplicate materials. Future work should incorporate diversity constraints, contradiction detection, and enhanced source filtering before conditioning the model to ensure reliable generalization across clinical contexts.

---

## Summary: What SheepSOC Actually Knows vs. Needs to Measure

| Claim | Status | Confidence Level | SheepSOC Action |
|-------|--------|-----------------|-----------------|
| "0% vs 8% hallucination with RAG" | ✅ Verified paper, specific domain | High (domain-specific) | Replicate on own domain corpus |
| "3B RAG matches 15.5B no-RAG" | ✅ Verified paper, structured outputs | Moderate (code task only) | Test on your 7B/13B range with free-text |
| "91% recall@10 hybrid" | ⚠️ Practitioner consensus, no single paper | Low for exact number | Measure yourself; expect improvement direction |
| "18% nDCG ELSER v2 over BM25" | ✅ Elastic's own BEIR benchmark | Moderate (self-reported) | Run your own BEIR-style test on SheepSOC corpus |
| Precision@K → hallucination causal link | ✅ Multiple papers (RAGCHECKER, Agnihotram 2025) | High | Use Precision@5 as your primary retrieval metric |
| Chunking affects quality as much as model | ✅ Vectara NAACL 2025 (arXiv:2410.13070) | High | Run chunking ablation as first experiment |