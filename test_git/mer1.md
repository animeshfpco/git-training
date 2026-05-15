# Cracking Mercari Japan's ML interview loop

**The Mercari Tokyo ML/AI hiring loop is unusual.** Coding rounds are standard LeetCode-medium on HackerRank, but the decisive stage is an ML take-home that is later cross-examined live by two engineers who attack every modeling choice — metric selection, leakage, calibration, threshold logic, distribution shift, embedding drift — before pivoting into a free-form ML system design over a Mercari-flavored problem (search ranking, two-tower recs, fraud, AI-Listing). The bar is *defensible reasoning under hostile probing*, not pattern matching from textbooks. Most rejections come from (a) sloppy take-home submissions, (b) reciting model names without trade-offs, or (c) ignoring production realities like A/B testing, drift, and cost. Mercari's stack is GCP-native (Vertex AI Pipelines, Vector Search, BigQuery, Bigtable, GKE, Feast, Spanner), Python + PyTorch + Hugging Face for ML, Go for backend microservices, Elasticsearch for retrieval, TFLite for edge, and a heavy reliance on OpenAI (GPT-4o-mini) plus Gemini for LLM features. Reading Mercari's own `ml-system-design-pattern` repo, the "Journey to ML Re-ranking" blog, and the MerRec paper before the interview is the single highest-leverage prep step.

This report is organized in three layers: (1) the universal interview pipeline and what to do/not do in each stage, (2) a per-role technical deep dive for all nine roles, and (3) a cross-cutting "what to study cold" reference covering ML fundamentals by specialty, Mercari's actual stack, and concrete practice problems.

---

## The universal Mercari ML interview pipeline

Mercari's careers page defines the canonical stages: **résumé screen → HackerRank/GitHub skill assessment → multiple interviews → reference check → offer**. In practice, for ML roles the loop is:

1. **Recruiter screen** (skip — non-technical).
2. **HackerRank online assessment** — 2–3 LeetCode-medium DSA problems, ~60 minutes, mostly arrays / hash maps / DP / heaps / bit manipulation. Mercari has raised the bar since 2022; current samples include the "MEX problem", wildcard-`*` substring search, and graph/DP problems. **100% test-case pass is necessary but not sufficient** — code quality (naming, modularity, docstrings, simple functional decomposition) is reviewed.
3. **ML take-home assignment** (1-week deadline, scoped for ~8 hours). Two known patterns: a **small-data tabular classification** (e.g., 5-class, 700 samples, target macro-F1 ≥ 0.75, ~2.76:1 imbalance — solved in production by a hire with CatBoost + OOF target encoding + threshold optimization → 0.87 macro-F1) or a **backend-flavored task** (e.g., build a web API around an OCR engine). For Edge/CV roles, image-classification take-homes are reported.
4. **Technical deep-dive (~90 min, 2 engineers)** — defense of the take-home. This is where most candidates fail. Expect rapid-fire questions like *"Why macro-F1 over weighted-F1?"*, *"Why not SMOTE?"*, *"Prove your out-of-fold encoding is leakage-free"*, *"What if production traffic shifts so unseen hashes dominate?"*, *"Is threshold optimization equivalent to cost-sensitive learning?"*
5. **ML system design** — sometimes merged with #4, sometimes separate. Open-ended Mercari-flavored prompts: design a recommendation home feed, design search ranking, design a fraud-detection pipeline, design AI Listing Support.
6. **Engineering Manager + Director rounds** — past-project deep dive, trade-off reasoning. Out of scope for this report (behavioral).

**Total elapsed time:** 3 weeks to 2 months. **Language:** English for nearly all ML roles (B2 required); Japanese is preferred but not required.

### What TO DO across every round
Always start by asking clarifying questions before writing code or designing — Mercari's tech lead Jieqiong Yu has stated publicly that they evaluate *why* you made decisions, not just *how*. Narrate your thinking constantly. State assumptions before committing to a design. Frame every modeling choice as a trade-off with a discarded alternative ("I chose CatBoost over a neural net because with 700 rows the variance from random init dominates; I considered XGBoost but CatBoost's native categorical handling reduces leakage risk vs target encoding"). For system design, **always cover the full ML lifecycle**: data → features → model → serving → monitoring → A/B → cost. Use Mercari's own vocabulary from `ml-system-design-pattern`: prediction cache, async prediction, microservice inference, training pattern, anti-patterns. End every answer with "limitations" — what could go wrong, what you'd monitor, when you'd retrain.

### What NOT TO DO
Do not submit a take-home that just scores well on the metric; Mercari explicitly evaluates the *process* over the score, and a candidate flagged in interview reviews that "they don't care about accuracy but actually they judge based on that". Do not name-drop models without trade-offs ("I'd use BERT4Rec" with no follow-up is an instant downgrade). Do not skip data and evaluation discussion in system design — going straight to model architecture is the most common rejection pattern. Do not handwave production: if you can't talk about feature stores, training-serving skew, drift monitoring, and online A/B with guardrails, you cannot pass an ML system design at Mercari.

### What to ACTIVELY AVOID (red flags)
Over-engineering at small scale ("let me design a 100-node Spark cluster for 700 rows"). Ignoring class imbalance and proposing accuracy as the metric on a 0.1%-positive fraud problem. Proposing a cross-encoder for billion-item retrieval. Saying "I'd just fine-tune a 70B LLM" for a feature that ships to 20M users per day. Claiming RLHF/DPO experience without being able to explain the reward-model objective. Suggesting SMOTE on 700 samples without acknowledging it distorts minority manifolds. Forgetting that Mercari is **C2C** — inventory of 1, every item is unique, no SKU repeats, dual-role buyer-and-seller users, cold start everywhere. Confidently quoting offline metrics (nDCG, HR@K) as proof of success when Mercari has publicly written that those gains often do not translate to GMV — interviewers will probe this gap on purpose.

---

## Role-by-role technical deep dive

### 1. Software Engineer (Machine Learning & Search) — the "ML/Search Platform" role

**What the role actually is.** This posting (Workable `3333DC5597`) is *not* a modeling role despite its title. It is the **ML/Search platform engineer** position. The JD reads "platform engineers for machine learning/search systems develop the functions and services of the marketplace app … through the development and maintenance of infrastructure and platforms." Hard requirements are **5+ years of SWE and 3+ years of Java, Python, or Go**; preferred skills are **Kubernetes, Hadoop, Docker, TensorFlow Serving, TensorFlow Lite, ONNX, Elasticsearch, and Solr**. Office: Roppongi, Tokyo.

**Coding round.** Same HackerRank OA as all other Mercari SWE roles — 2–3 LeetCode mediums, 60 minutes, Python or Go preferred. Because this is a platform role, expect at least one problem to test data structures relevant to infra (LRU cache, rate limiter, producer-consumer queue). Code quality matters more here than for pure modeling roles — Mercari hires this person to write libraries.

**System design.** Expect classic systems design with an ML twist: "design a feature store", "design a model serving layer that handles 10k QPS at p99 < 50ms across 30 teams' models", "design CI/CD for search-index rollouts with safe rollback". The reference architecture you should be able to draw is Mercari's actual one: **Elasticsearch on ECK on GKE for retrieval, Go gRPC middleware fronting it, Redis-based dynamic feature store for online features, Vertex AI Pipelines for batch training, Vertex AI Endpoints or GKE pods for serving, BigQuery + Bigtable for offline+online feature stores, Feast as the feature framework**.

**ML fundamentals required.** Shallow but practical. Know **what** models exist (BM25, learning-to-rank, two-tower, cross-encoder) and their **serving characteristics** (latency, memory, batching potential), not the math. You must know **why Mercari chose Feast over Vertex AI Feature Store** (stream ingestion support, lower cost) and **why CPU-based HPA outperforms request-rate HPA on Elasticsearch** (force-merge interference). Read the engineering posts "Operational tips on Elasticsearch" and "ECK HPA autoscaling".

**Do.** Quote infra trade-offs in concrete numbers ("we got 40% k8s cost reduction with CPU HPA"). Discuss observability — what you'd log, what dashboards, what SLIs/SLOs.
**Don't.** Pretend to be a modeling expert. The hiring manager wants someone who keeps the ML team's experiments running, not someone competing with the modeling track.
**Avoid.** Designing systems that assume infinite compute. Mercari has explicit cost optimization stories (PCA → linear head saved 80% on Vector Search) and will probe cost.

### 2. SWE (ML & Recommendation) — junior/mid level

**What the role actually is.** Mercari's Tokyo "Software Engineer (ML & Recommendation)" posting **already requires 5+ years of end-to-end ML production experience** — there is effectively no separate junior posting in Tokyo. Junior recommendation ML hires come via the New Graduate track or internships. The JD lists **TensorFlow, PyTorch, scikit-learn, NumPy, pandas** as required and **Docker/Kubernetes, AWS/GCP/Azure, deep learning + LLMs in production, ELK** as preferred.

**Coding round.** Same HackerRank OA. For mid-level, expect to be timed more strictly and judged more harshly on code quality.

**Take-home.** Most likely a small-dataset modeling problem similar to the 5-class / 700-sample / macro-F1 task documented by the Feb 2026 hire. The expected submission style: a clean repo with README, reproducibility (seed, pinned env), unit tests, EDA notebook, a `train.py` and `predict.py`, threshold/calibration justification, a "limitations" section. Submit CatBoost or LightGBM as the baseline, then justify any deep learning lift.

**Technical deep-dive (90 min, 2 engineers).** The actual questions documented from a successful 2026 hire:
- Take-home defense: macro-F1 vs weighted-F1, why not SMOTE, why clip class weights, prove OOF encoding is leakage-free, calibration under imbalance, what happens when production traffic shifts.
- Recsys pivot: **two-tower architecture** — independent user/item encoders, ANN retrieval, embedding drift if one tower trains faster than the other, candidate generation vs ranking, cold start, **in-batch negative sampling bias and the logQ correction**, representation collapse.
- **BM25 vs dense retrieval trade-offs**, fraud → "structured numeric features prefer gradient-boosted trees like XGBoost over LLMs".
- LLM inference internals: KV cache memory growth, why decoding is memory-bandwidth bound, paged attention, vLLM block memory.
- Multimodal: CLIP vs SigLIP for regression stability, cross-attention vs concatenation fusion, head depth vs overfitting.
- Productionization: "If you had to ship this tomorrow, what changes?" — expected answers include precomputed embeddings, feature stores, embedding-drift monitoring, batch vs real-time serving.

**ML system design.** Expect "design home-feed recs for Mercari" or "design similar-items". The reference architecture from Mercari's own production:

```
[User events: views/likes/purchases]                                              
            │                                                                    
            ▼                                                                    
[BigQuery]──►[Dataflow batch]──►[implicit-CF ALS on GPU] ──┐                     
                                                          ├──►[Item factors]    
[Item text/image]──►[NN mapping text→CF factor space]─────┘  (cold-start)        
            │                                                                    
            ▼                                                                    
[Vertex AI Vector Search (ScaNN) index]──►[ANN top-K candidate retrieval]        
            │                                                                    
            ▼                                                                    
[GBDT/DNN ranker w/ position-bias correction]──►[Online endpoint]                
            │                                                                    
            ▼                                                                    
[A/B test w/ CUPED + guardrails]──►[Model monitoring (drift, calibration)]       
```

**ML fundamentals to know cold.** Implicit-feedback ALS/BPR; two-tower with logQ correction (Yi et al. 2019); SASRec / BERT4Rec / GRU4Rec / NextItNet (all benchmarked in Mercari's MerRec paper); YouTube DNN paper (Covington 2016); Wide & Deep; DeepFM; DIN/DIEN; LightGCN / PinSage; Thompson Sampling + LinUCB for cold start; position bias and IPS weighting; calibration (Platt/isotonic); the **MerRec paper (arXiv 2402.14230)** end-to-end. **The single most important Mercari-specific insight**: every item is unique inventory of 1, so naive CF breaks — Mercari trains a small NN that maps title/description embedding to the CF factor space, solving cold start while keeping the CF signal.

**Do.** Always start system design from data + evals. Show you know offline metrics (nDCG, HR@K) drift from online GMV. Discuss two-tower + reranking as a default. Discuss the **C2C cold-start asymmetry** (1M new items/day).
**Don't.** Propose pure BERT/transformer retrieval for billions of items without admitting the cost problem (Mercari chose word2vec over BERT explicitly).
**Avoid.** Confusing user-personalization with item-quality ranking. Mercari publicly worried about filter bubbles and seller-exposure fairness in a two-sided marketplace.

### 3. Senior SWE (ML & Recommendation) — what mid-level should aspire to

**Status.** There is no standalone "Senior" posting at Tokyo HQ — Mercari uses one JD and levels candidates via the interview. A separate Senior posting exists for Mercari **India (Bengaluru)** which is explicitly out of scope. For Tokyo seniors, the same JD applies with added expectations in the loop:

- **Owning architecture decisions end-to-end** for a recsys pillar (e.g., session-based ranker, or the entire two-tower retrieval stack).
- Designing **A/B experimentation infrastructure** for a two-sided marketplace (switchback tests for marketplace interference, interleaving for ranking).
- Trading off **build vs buy** explicitly (e.g., "managed Matching Engine vs self-built FAISS" — Mercari Shops chose managed because they had one ML engineer).
- Mentoring + cross-team alignment with Search, Catalog, Trust & Safety teams.
- Publications at top venues (KDD, RecSys, SIGIR) are listed as preferred, signaling that conference-grade depth is valued at senior.

For mid-level candidates: knowing the **senior bar** means you should be able to discuss not just "use two-tower" but **logQ sampling-bias correction, hard-negative mining, embedding-tower asynchrony, the switchback experiment design, and the org-level decision to use Vertex AI Vector Search rather than building ANN in-house**.

### 4. Machine Learning Platform Engineer

**What the role actually is.** Mercari does not post this title separately — the function is rolled into the SWE (ML & Search) posting (#1) plus the cross-cutting ML Platform team that supports Recommendation, Search, T&S, and AI/LLM. JD signals: **5+ years SWE, 3+ years Python/Java/Go, Kubernetes, GCP, monitoring/logging, RDBMS/SQL/Linux**.

**Coding round.** Standard HackerRank OA. Bias toward problems testing data structures used in infra (caches, queues, schedulers). Go is acceptable and probably welcomed.

**Take-home.** Likely a backend-flavored task: build a small pub/sub or batch-job API, validate Kubernetes YAML, or implement a feature-store-style point-in-time lookup service. Reproducibility + tests are non-negotiable.

**System design — the canonical Mercari ML platform.** You should be able to draw this from memory:

| Layer | Mercari's choice | What you must know |
|---|---|---|
| Orchestration | Vertex AI Pipelines (KFP v2 SDK) | Function-based components, conditional deploy, importer for inputs |
| Custom training | Vertex AI Custom Training in containers | When to GKE vs Vertex Custom Job |
| Experiment tracking | Weights & Biases (enterprise) + Vertex AI Experiments | Run grouping, sweep, artifact lineage |
| Feature store | **Feast** (online=Bigtable, offline=BigQuery) | Point-in-time joins, why Feast over Vertex AI Feature Store (stream ingestion, cost) |
| Model registry | Vertex AI Model Registry | Versioning, aliases, staging |
| Online serving | Vertex AI Endpoints OR GKE pod | Trade-off: managed vs cost/latency control |
| Batch inference | Kubernetes CronJob writing to CloudSQL → Dataflow → BigQuery | Mercari's "Customer Profiling Platform" pattern |
| Vector search | Vertex AI Vector Search (ScaNN-based, formerly Matching Engine) | Streaming index updates within seconds |
| Monitoring | Vertex AI Model Monitoring + custom Prometheus | KS test, PSI, drift, training-serving skew |
| CI/CD | GitHub Actions, Spinnaker, Terraform-CI, Argo Workflows | Blue/green model rollouts |

**ML fundamentals.** Sculley et al. 2015 "Hidden Technical Debt in ML" (Mercari cites it). Read Mercari's open-source **`mercari/ml-system-design-pattern` repo end-to-end** before interview — it catalogues serving patterns (microservice, batch, async, prediction cache), training patterns, QA patterns, and anti-patterns. Distributed training: DDP, FSDP, ZeRO 1/2/3, DeepSpeed.

**Do.** Reference Mercari's published platform posts ("Vertex Pipelines in Merpay ML Team", "Customer Profiling Platform", "How we optimized our ML training infrastructure costs" from Mercari US which describes Polyaxon + Kubeflow on GKE with PDB and Gatekeeper Assign CRDs to fix autoscaler downscaling). Frame everything as build-vs-buy with cost trade-offs.
**Don't.** Propose Airflow as the orchestrator (Mercari uses Vertex Pipelines as primary; Airflow only for specific Merpay batch jobs).
**Avoid.** Designing one-size-fits-all platforms — Mercari's actual platform is **multi-tenant on GKE with namespaces**, not per-team clusters, and they discuss this trade-off publicly.

### 5. ML Engineer – Item Understanding / Catalog

**Status.** No standalone JD; you apply via the generic "Software Engineer, Machine Learning" posting. Team members (Antony Lam ex-CV/CVPR; Bosco) own catalog enrichment, attribute extraction, image recognition for listings, AI Listing Support. Day-to-day: image classification, OCR, NLP attribute/brand extraction, multimodal embeddings, taxonomy classification at 3B-item scale.

**System design — must be ready to design these from memory:**

1. **"Re-categorize 3B items into a new taxonomy"** (Mercari literally did this in 2024). The answer they want: a two-stage approach — (a) label a few million items with **GPT-3.5-turbo via OpenAI API**, (b) train a downstream embedding-based kNN classifier (open-source embeddings + multi-GPU + vector DB, Numba JIT-accelerated nearest-neighbor scoring) to scale to all 3B. Naive full-LLM approach would have cost ~$1M and taken 1.9 years; the two-stage approach made it tractable.
2. **"AI Listing Support: photo + selected category → title, description, condition, price"** (launched Sept 10, 2024). The answer they want: **GPT-4o-mini** in production (downgraded from GPT-4 for cost/latency, supports multi-image); structured JSON output; offline/online split where offline mines best-practice listings per category and online generates suggestions.
3. **"Similar items / similar looks"**. The answer: **fine-tuned SigLIP** (ICCV 2023 sigmoid contrastive loss) on 1M Mercari image-title pairs, 512-d embeddings reduced via a trained linear layer (not PCA — PCA is lossy and Mercari explicitly chose linear head to save 80% on Vector Search cost), stored in Bigtable + Vertex AI Vector Search.

**ML fundamentals.** CLIP, ALIGN, **SigLIP** (sigmoid loss is more stable for fine-tuning on small batches), MobileNetV2 (Mercari's image-search backbone, width-multiplier 1.4, 1792-d feature, trained on 9M images × 14K classes), Japanese BERT (`cl-tohoku/bert-base-japanese` with **fugashi + unidic-lite**), BLIP/BLIP-2 for captioning, perceptual hashing for near-dupes, hierarchical/taxonomy classification (flat softmax vs cascaded vs hierarchical softmax), NER for attribute extraction.

**Do.** Discuss the **distribution shift** from professional product photos (Web data SigLIP pretrains on) to noisy user phone photos (what Mercari actually sees). Discuss **LLM-as-labeler economics**: when is GPT-3.5 cheaper than human annotation, and when do you distill to a small model.
**Don't.** Propose end-to-end fine-tuning of CLIP on 700 samples — domain gap is real but data is precious; freeze the encoder, train a head.
**Avoid.** Ignoring multilingual + Japanese specifics (mixed kanji/kana/romaji titles, katakana brand variants like ヴァイオリン vs バイオリン).

### 6. ML Engineer – Trust & Safety

**Status.** Tokyo T&S ML exists as a team (mercan articles 30964, 49488; Bosco, Henning, Calvin, Ayato, Shuji listed on `ai.mercari.com/people`) but no standalone Tokyo JD currently; you apply via the generic ML SWE posting. T&S engineering hiring has partly consolidated to Mercari India (Bengaluru — out of scope).

**Tech stack of T&S ML at Mercari** (from the Speaker Deck "MLOps in Mercari Group's Trust and Safety ML Team"):
- Backend: 6 Go services on GCP (Spanner, CloudSQL, Pub/Sub).
- ML service: Python, scikit-learn, Kubeflow, **dbt + Dataflow + BigQuery + Bigtable**, **Feast feature store with point-in-time joins**, **Vertex AI Experiments**, deployed to Vertex AI Endpoint or GKE Pod via Vertex AI Model Registry.
- Build/release: Docker, Kubernetes, Spinnaker.

**System design — must be ready to design these:**

1. **Item moderation at 1.5M items/day with 20.4M MAU**. The reference: rule-based Watchlist + ML post-listing classifier in tandem (rules give explainability and immediate response; ML catches novel patterns). Multimodal classification stack: **MobileNet V2 image embedding (1280-d) + Japanese BERT text embedding (768-d from [CLS])** fused via a custom PyTorch `AdultFusionModel` that supports both concatenation-MLP and cross-attention fusion strategies. Two independent classifiers (in-house + 3rd-party API) run in parallel through an **OR gate** to reduce single-model failure risk.
2. **Fraud / chargeback detection** (Merpay). Binary classifier retrained on a schedule via Airflow + AI Platform; uses time-series velocity features (transactions/min, distinct cards per user, account-age vs transaction graphs); served real-time at <100ms via Feast online store on Bigtable.
3. **Fraud-ring detection across thousands of accounts**. GNN on a heterogeneous user/device/card/IP graph (GraphSAGE / R-GCN / GAT); community detection (Louvain/Leiden) for ring discovery; CARE-GNN for camouflage-aware fraud.

**ML fundamentals.** Imbalanced classification (PR-AUC over ROC-AUC, class weights, focal loss; **avoid SMOTE on small data**), Isolation Forest, autoencoder reconstruction-error anomaly detection, GNNs (GraphSAGE, GAT, R-GCN, CARE-GNN), GMU multimodal fusion (Arevalo 2017 — replicated by Mercari), Neural Architecture Search (DARTS — Mercari published using NAS to automate multimodal moderation model construction, cutting new-violation model dev from months to weeks), concept drift detection (KS, PSI, ADWIN), SHAP/LIME for explainability to CS agents.

**Do.** Discuss **rules + ML tandem** explicitly. Discuss **OR-gate ensembles** of independent classifiers. Discuss adversarial drift — fraudsters adapt, so retrain cadence matters. Mention PR-AUC + feature importances + business KPIs as your evaluation triad (Mercari's published triad).
**Don't.** Propose deep learning on a 0.1%-positive structured fraud problem without first justifying it over XGBoost. Mercari's own 2026 hire was told: "Structured numeric features prefer gradient-boosted trees like XGBoost over LLMs."
**Avoid.** Forgetting that **false positives on legit users are extremely costly** in a marketplace (lost GMV, support tickets, brand damage). Always frame fraud as a precision/recall trade-off with operational cost numbers.

### 7. AI/LLM Engineer

**This is Mercari's most actively-hiring team** ("Eliza team", builders of internal "Ellie" LLM hub and the user-facing AI Listing Support / Mercari AI Assistant). Multiple posting variants exist:

| Workable ID | Title | Bar |
|---|---|---|
| `DA924B073F` | ML Engineer (AI/LLM) | Modeling-heavy, PyTorch/TF, prod ML systems |
| `DBF70CE253` | Senior ML Engineer (AI/LLM) | Same + tech leadership |
| `FACCBF9D14` | ML Research Engineer (AI/LLM) | NeurIPS/ICML/CVPR/ICLR/ACL publications preferred |
| `854CD29229` | SWE Backend (AI/LLM) | 2+ yrs backend, Go/Python microservices, GCP/AWS, integrate LLM partner APIs |
| `76EB5EB641` | SWE Frontend/Full-Stack (AI/LLM) | Product-facing GenAI UX |

**Coding round.** Standard HackerRank OA. For the Backend AI/LLM variant, expect strong emphasis on API design / concurrency questions on the take-home stage.

**System design — must be ready to design these:**

1. **"Design AI Listing Support"**. Reference answer: **multimodal VLM (GPT-4o-mini) → structured JSON output**, evals via LLM-as-judge + RAGAS + A/B on GMV/CVR; guardrails for prohibited-item leakage; cost-cascade routing (cheap model first, escalate on low confidence); **hybrid offline-online pattern** (offline GPT-4 mines best-listing examples per category; online GPT-3.5/4o-mini applies them).
2. **"Design RAG for vendor security review"** (Mercari literally built this — engineering blog Dec 2024). Components: chunking strategy, embedding model (Vertex multimodal or OpenAI text-embedding-3), vector DB, hybrid BM25 + dense + RRF, cross-encoder reranking, citation grounding, eval via faithfulness/answer-relevance/context-precision.
3. **"Build LLM evaluation infrastructure"** (Mercari's "SRE 2.0: No LLM Metrics, No Future" blog, June 2025). Reference: G-Eval / LLM-as-a-Judge with position-bias and verbosity-bias mitigation; Vertex AI metric prompt templates; DeepEval / SelfCheckGPT / QAG / Prometheus 2 / DAG frameworks.

**ML fundamentals to know cold.** Decoder-only transformers, **MHA vs MQA vs GQA** (Llama 2/3, Gemini all use GQA), RoPE, RMSNorm, SwiGLU. **FlashAttention v1/v2/v3** (IO-aware tiling). **KV cache** — why prefill is compute-bound and decode is memory-bandwidth-bound; **paged attention** (vLLM); continuous batching; speculative decoding. **Fine-tuning spectrum**: SFT → LoRA (rank 8–16 is usually enough because LLM updates lie on a low-rank manifold) → QLoRA (4-bit base + LoRA adapters) → PEFT → RLHF (PPO over reward model) → DPO/IPO/KTO (preference optimization without explicit reward model). **RAG**: chunking strategies (fixed, recursive, semantic), embedding models (BGE, E5, Cohere, Gemini text-embedding, OpenAI text-embedding-3), hybrid retrieval with RRF, reranking, HyDE, query rewriting. **Evals**: LLM-as-judge biases (position, verbosity, self-preference), RAGAS metrics (faithfulness, answer relevance, context precision/recall), MT-Bench/Arena-Hard, hallucination via FactScore. **Agents**: ReAct loop, function calling, OpenAI tools spec, LangChain/LlamaIndex limits.

**Vertex AI Gemini specifics** (Mercari is on GCP — this is mandatory): Gemini 2.5 Pro / Flash / Flash-Lite, `google-genai` SDK, **grounding** (Google Search + custom data stores), function calling, Vertex AI RAG Engine, Vertex AI Vector Search, multimodal embeddings (CoCa-based); aware that Vertex AI is rebranding to **Gemini Enterprise Agent Platform**.

**Do.** Show you know **why GPT-4 → GPT-4o-mini happened in production** (cost, latency, multi-image support). Discuss the **closed-vs-open trade-off**: Mercari uses OpenAI as primary "because they swiftly release new versions that put them back on top" but maintains Gemini as alternative for redundancy + GCP integration. Discuss **structured output / JSON mode / constrained decoding (Outlines)** for reliability.
**Don't.** Propose "just fine-tune a 70B LLM" — Mercari's pattern is *use frontier API + distill or RAG*, not self-host massive models. Mercari has only published self-hosted models at the **DistilBERT / SigLIP / Japanese BERT** scale.
**Avoid.** Hallucination as a side-note. Mercari has a dedicated AI Security Team (formed May 2025) for OWASP 2025 Top 10 GenAI risks (prompt injection, data leakage); discuss prompt-injection defenses in any RAG design.

### 8. Search Engineer (ML)

**JD title used by Mercari: "Machine Learning Engineer, Search"** (also historically as "SWE, Machine Learning & Search (Ads)" — Workable `8AD602B228`). Team: Search team led by Anri, split into Search Middleware and Search Infra; Mohit and others on ML re-ranking. JD requires **TensorFlow/PyTorch/scikit-learn/NumPy/pandas + deep IR understanding (search relevance, ranking, query understanding) + end-to-end ML to production + hands-on NLP (tokenization, language modeling, indexing, text classification)**.

**Coding round.** Standard HackerRank OA. Expect at least one string/parsing problem given the NLP-heavy nature.

**System design — must be ready to design these:**

1. **"Design search ranking for Mercari"**. Reference architecture from the "Journey to Machine-Learned Re-ranking" blog (2023): two retrieval flows ("newly posted" 新しい and "best-match" おすすめ) → first-stage Elasticsearch retrieval with **custom BM25 emphasizing freshness/recency boosting** → second-stage **LightGBM-style LTR re-ranker** trained on **human-labeled relevance** (clicks alone are too noisy and bot-contaminated in C2C) → features from Elasticsearch index + **Redis-based dynamic feature store** (CTR, impression probability, recency). Discuss **outlier issues** (¥9,999,999 listings) and **recency bias** from old items.
2. **"Design query understanding"**. Reference: **DistilBERT fine-tuned on query→category data**, hit micro-F1 0.80, doubled coverage of converted keywords vs prior ML model; rule-based fallbacks for maintainability. Mercari compared LR, SVM, XGBoost, fastText, attentional CNN, and DistilBERT before selecting.
3. **"Design image-quality ranking for search"** (Mercari published this — arXiv 2408.11349, "Image Score"). Reference: **ViT-B/16 CLIP** (Japanese-translated CC12M pretrain) + MLP head trained as **pairwise LTR (reward-model style)** with labels produced by **LLM Chain-of-Thought**; trained on a single NVIDIA T4 with AdamW (lr 1e-3, bs 128); significant web sales lift in online A/B.

**ML fundamentals to know cold.** **BM25** (k1/b parameters, length normalization) + TF-IDF. **Japanese tokenization** — MeCab/IPADIC (legacy), Kuromoji (Lucene-bundled, MeCab-style but with 2007 dictionary missing 令和 words), **SudachiPy** (UniDic, three split modes A/B/C — Mode A for precision/short units, Mode C for proper nouns/recall; has Elasticsearch plugin), SentencePiece (language-agnostic but loses morphology). **Learning-to-rank** — pointwise, pairwise (RankNet), listwise (LambdaRank, **LambdaMART / XGBoost-Ranker** is the workhorse). **Dense retrieval** — DSSM, DPR with in-batch + hard negatives, Sentence-BERT, **ColBERT** late interaction. **Cross-encoders** for top-k reranking. **Hybrid search** — BM25 + dense fusion via **Reciprocal Rank Fusion (RRF)**. Vector DBs — Vertex AI Vector Search/ScaNN, FAISS, HNSW (Vespa/Elasticsearch dense), Weaviate, Qdrant.

**Key papers**: Robertson & Zaragoza 2009 (BM25); Huang 2013 (DSSM); Reimers 2019 (SBERT); Karpukhin 2020 (DPR); Khattab 2020 (ColBERT); Chang 2020 ("Pre-training Tasks for Embedding-based Large-scale Retrieval" — ICT/BFS/WLP); Burges 2010 (RankNet → LambdaMART).

**Do.** Frame everything as **multi-stage retrieval** (recall → light ranker → heavy ranker). Discuss **clicks vs labeled relevance** explicitly (Mercari chose labels because of bot noise in C2C). Discuss **business KPI vs offline metric gap** — nDCG up but GMV flat, what happened?
**Don't.** Propose cross-encoders or full BERT re-rank over millions of candidates. Discuss "I'd use BERT for retrieval" without two-tower + ANN.
**Avoid.** Ignoring Japanese specifics. **Sudachi mode A vs C, NFKC normalization, hankaku/zenkaku, katakana variants (ヴァイオリン vs バイオリン), mojibake handling** all signal seriousness. Ignoring **freshness/recency** in a marketplace where items are inventory of 1.

### 9. Edge AI Engineer

**Status.** No active standalone JD. The Edge AI team exists and is documented (members include Chika Matsueda, Rakesh Kumar, Yusuke Shibui historically — Shibui authored Mercari's `ml-system-design-pattern` repo). You apply via the generic SWE ML posting and indicate Edge AI interest.

**Tech stack.** **TensorFlow Lite** is the primary on-device runtime (cross-platform iOS/Android chosen over CoreML for portability — note this contradicts the common assumption that CoreML is preferred for iOS-heavy Japanese mobile usage); **MediaPipe** for streaming camera pipelines; **8-bit integer quantization** for ~75% size reduction with minor accuracy loss; **XNNPACK / GPU delegates**. Models trained in TensorFlow or PyTorch then converted to `.tflite`. Orchestration via Kubeflow Pipelines; Kubeflow Metadata stores weights/configs/metrics. **JetFire** is Mercari's internal on-device validation platform that runs models on a real device farm (AWS Device Farm + local test lab), captures video-overlay visualizations of bounding-box predictions, and tracks per-device/per-OS/per-delegate latency and precision/recall in a React dashboard.

**Production features.** Real-time camera "sellability" tracker (DNN on phone camera frames, no frames sent to cloud — saves bandwidth and protects privacy); MobileNet SSD trained internally and validated through JetFire; real-time on-device price estimation (in development).

**System design — must be ready to design these:**

1. **"Build a real-time camera-based brand/category classifier for the listing flow"**. Reference: **MobileNetV2 backbone** (Mercari's actual choice, width-multiplier 1.4, 1792-d feature, 9M images × 14K classes), fine-tuned on Mercari brand data, **PTQ to INT8**, target latency <100ms on mid-tier iPhone, delivered via TFLite + MediaPipe streaming pipeline.
2. **"Deploy a 7B LLM on an iPhone"** (modern probe). Reference: **4-bit quantization (AWQ or GPTQ)**, GGUF or MLX format, memory budget ~3–4 GB, accept that the Apple Neural Engine isn't always usable for arbitrary LLM ops, fall back to Metal.
3. **"Validate model parity across 50 device + OS + delegate combinations"**. Reference: build a JetFire-style real-device farm with automated benchmarking, capture per-class accuracy (especially for fairness — TFLite quantization has known skin-tone bias issues), measure cold-start vs steady-state, watch thermal throttling.

**ML fundamentals to know cold.** **Quantization** — PTQ vs QAT, INT8/INT4, symmetric vs asymmetric, per-tensor vs per-channel vs per-group (g=128); **GPTQ** (Hessian-based one-shot), **AWQ** (activation-aware weight quant — preserves salient weights), **SmoothQuant** for activations, **LLM.int8()** for outlier handling. **Knowledge distillation** — response (KL on soft labels, Hinton 2015), feature (FitNets), relation (RKD); **DistilBERT, TinyBERT, MobileBERT**. **Pruning** — structured (channels/heads/blocks — actually speeds up on hardware) vs unstructured (just sparsity, rarely speeds up); magnitude, lottery-ticket, movement pruning. **Mobile architectures** — MobileNetV1/V2/V3 (depthwise separable conv, inverted residuals), EfficientNet, EfficientFormer, MobileViT, MobileBERT, DistilBERT. **Formats** — ONNX, CoreML (`.mlpackage` via coremltools), TFLite (LiteRT), MLIR, ExecuTorch (PyTorch's 2025 edge runtime). **iOS** — CoreML + Apple Neural Engine (ANE) + Metal Performance Shaders. **Android** — TFLite/LiteRT, NNAPI, GPU/Hexagon DSP delegates, Qualcomm QNN, MediaTek APU.

**Key papers.** Howard 2017 / Sandler 2018 (MobileNet V1/V2); Hinton 2015 (KD); Sanh 2019 (DistilBERT); Sun 2020 (MobileBERT); Frantar 2022 (GPTQ); Lin 2023 (AWQ); Xiao 2023 (SmoothQuant); Dettmers 2022 (LLM.int8()); Han 2015 (Deep Compression — pruning + quant + Huffman).

**Do.** Discuss **why TFLite over CoreML** (cross-platform iOS+Android, MediaPipe integration) — this is the **non-obvious Mercari choice**. Discuss **on-device vs server trade-off** with explicit numbers (zero per-inference cost, low latency, works offline, privacy — vs ceiling on accuracy, can't ship model instantly because of app review). Discuss **JetFire-style real-device validation** as the productionization story.
**Don't.** Assume CoreML is the default. Forget about A/B testing constraints (can't roll back a shipped on-device model instantly).
**Avoid.** Quantization without measuring per-class accuracy parity (fairness/bias). Pruning unstructured weights and claiming it speeds up inference (it doesn't on most hardware).

---

## Universal hard-skill prep — what every Mercari ML candidate must know

### Coding round preparation
Practice **HackerRank's interview prep kit** plus ~150–200 LeetCode mediums focused on the patterns Mercari actually tests:

| Category | Why Mercari tests it | Practice examples |
|---|---|---|
| Arrays / two pointers / sliding window | Most OA problems | Max subarray, longest substring |
| Hash maps / sets | All three documented 2022 OA problems used hashes | Two sum variants, group anagrams, MEX-style |
| Dynamic programming | Recurrent in OAs | LIS, coin change, edit distance |
| Heap / priority queue | Common in 2023+ OAs | Top-K elements, merge K lists |
| Graphs (BFS/DFS) | At least one in harder OAs | Number of islands, shortest path |
| Bit manipulation | Reported in 2024 Tokyo OA | XOR tricks, single number variants |
| String parsing | Especially for Search role | Wildcard match `*`, regex-lite |

**Language:** Python is universal; Go is welcomed for platform roles. **Format:** ~60 minutes for 2–3 problems on HackerRank. **Bar:** 100% test-case pass + clean code (naming, modularity, small pure functions, docstrings, basic type hints). Do not optimize for raw speed at the expense of readability.

### Mercari's tech stack — what to familiarize yourself with

**Cloud (GCP-native, almost no AWS/Azure):**
- Compute & serving: **GKE, Vertex AI Endpoints, Vertex AI Pipelines (KFP v2 SDK), Vertex AI Custom Training, Vertex AI Model Registry, Cloud Run, Cloud Functions, Kubernetes CronJobs**.
- Data: **BigQuery** (offline warehouse + feature store), **Cloud Bigtable** (online feature/embedding store), **Cloud Spanner** (Merpay), CloudSQL, GCS, **Pub/Sub, Dataflow, dbt**.
- Vector: **Vertex AI Vector Search** (ScaNN; formerly Matching Engine) with streaming index updates.
- Feature store: **Feast** (online=Bigtable, offline=BigQuery) — chosen over Vertex AI Feature Store for stream ingestion support and cost.
- Tracking: **Weights & Biases** (enterprise) + Vertex AI Experiments.
- CI/CD: GitHub Actions, Spinnaker, Terraform-CI, Argo Workflows.

**Languages:** **Go** for backend microservices (gRPC, 6 T&S services, 300+ across Mercari), **Python** for ML (PyTorch primary, scikit-learn, Hugging Face Transformers, kfp), some TypeScript/React for ML admin UIs. Java mostly legacy.

**ML frameworks:** **PyTorch + Hugging Face Transformers** is dominant (training, vision-language, edge-prep). TensorFlow exists mainly via TFLite/MediaPipe in Edge AI. scikit-learn ubiquitous for classical T&S models.

**Search infra:** **Elasticsearch on ECK on GKE** with custom BM25 + freshness boosting; Go gRPC middleware in front; Redis dynamic feature store.

**LLM infra:** Primary = **OpenAI** (GPT-4o, GPT-4o-mini, GPT-3.5-turbo); secondary = **Google Vertex AI Gemini**; tertiary = Anthropic Claude (security tools). Internal hub **"Ellie"** wraps all three providers.

**Edge:** **TensorFlow Lite + MediaPipe**, INT8 quantization, MobileNetV2 backbones, JetFire on-device validation.

### Required reading list (in priority order)
1. **`mercari/ml-system-design-pattern` GitHub repo** — the canonical vocabulary for ML system design at Mercari. Read end-to-end.
2. **"The Journey to Machine-Learned Re-ranking"** (Zagniotov & Calland, 2023) — search ranking story.
3. **"LLM-based Approach to Large-scale Item Category Classification"** (2024) — 3B-item categorization.
4. **"Improving Item Recommendation Accuracy Using CF and Vector Search"** (2023) — recsys with cold-start NN.
5. **MerRec paper** (arXiv 2402.14230) + GitHub `mercari/mercari-ml-merrec-pub-us` — C2C recommendation benchmark with SASRec/BERT4Rec/GRU4Rec/NextItNet.
6. **"Image Score: Learning and Evaluating Human Preferences for Mercari Search"** (arXiv 2408.11349) — ViT-B/16 CLIP with LLM-CoT labels.
7. **"Language Model-based Query Categorization"** (Dec 2023) — DistilBERT query understanding.
8. **"Multimodal Information Fusion for Prohibited Items Detection"** + NAS sequel — T&S multimodal classifiers.
9. **AI Listing Support announcement** + the **OpenAI customer case study on Mercari** — GPT-4 → GPT-4o-mini production story.
10. **"Vertex Pipelines in Merpay ML Team"** + **"Customer Profiling Platform"** + **"How we optimized our ML training infrastructure costs"** (Mercari US) — MLOps patterns.
11. **"SRE 2.0: No LLM Metrics, No Future"** (June 2025) — LLM eval framework.
12. **JetFire blog** (Dec 2021) — Edge AI validation.
13. **`mercari/Solr-Lucene-Analyzer-Sudachi`** — Japanese tokenization for search.

### Practice problem set — concrete examples

**Coding (HackerRank-style):**
- MEX of an array after ±X operations (actual Mercari 2022 OA).
- Wildcard substring search with `*` (actual Mercari 2022 OA).
- Minimum start value (actual Mercari 2022 OA).
- LRU cache (platform roles).
- Top-K frequent elements (recsys-flavored).
- Rate limiter design.

**Take-home (small-data ML):**
- 5-class classification, 700 samples, class imbalance ~2.7:1, target macro-F1 ≥ 0.75. Baseline = CatBoost with out-of-fold target encoding for high-cardinality categoricals + threshold optimization; document why no SMOTE, why macro-F1 not weighted, calibration choice (Platt vs isotonic), distribution-shift discussion, leakage-free OOF proof.
- Image classification on a small noisy dataset (Edge/CV roles).

**ML system design (Mercari-flavored, 45 min each):**
- "Design Mercari home-feed recommendations" (two-tower + ranker + cold-start NN + A/B with CUPED + guardrails).
- "Design Mercari search ranking" (Elasticsearch + BM25 with recency + LightGBM LTR + Redis dynamic features + offline-vs-online metric gap).
- "Design AI Listing Support" (GPT-4o-mini multimodal + structured JSON + cost cascade + LLM-as-judge eval + prompt-injection defense).
- "Design fraud detection for Merpay" (Feast feature store + XGBoost + velocity features + GNN for ring detection + concept drift monitoring).
- "Re-categorize 3B items to a new taxonomy" (LLM-as-labeler + embedding kNN distillation + Numba acceleration).
- "Deploy a real-time camera-based item classifier on mobile" (MobileNetV2 + TFLite + INT8 + JetFire validation + on-device vs server trade-off).

### Final cross-cutting do/don't/avoid checklist

**Do (every interview):**
- Open with clarifying questions; state assumptions before designing.
- Frame every decision as a trade-off vs a named alternative.
- Discuss data + evals before architecture.
- Cover the full ML lifecycle in any system design: data → features → model → serving → monitoring → A/B → cost.
- Quote Mercari-specific publications and patterns when relevant ("similar to the journey-to-ML-rerank blog…").
- End with "limitations" and "what I'd monitor".

**Don't:**
- Submit a take-home that just optimizes the metric without process documentation.
- Name-drop models without trade-offs.
- Use offline metrics as the primary success measure for system design.
- Propose end-to-end LLM self-hosting at Mercari scale.
- Treat T&S as purely classification — ignore explainability, drift, false-positive cost.
- Ignore Japanese-specific NLP and C2C-marketplace specifics.

**Actively avoid:**
- Over-engineering at small scale.
- Under-engineering at marketplace scale (forgetting two-sided effects, switchback A/B, fairness across buyers/sellers).
- Proposing cross-encoders for billion-item retrieval.
- Using accuracy on a 0.1%-positive fraud problem.
- Recommending SMOTE on small tabular data without acknowledging manifold distortion.
- Ignoring cost — Mercari has published cost-optimization stories at every layer (PCA→linear head saved 80%, GPT-4→GPT-4o-mini saved cost+latency, word2vec chosen over BERT, CPU HPA chosen over request-rate HPA).
- Forgetting about training-serving skew, drift, point-in-time correctness in feature stores.

## Conclusion: reasoning beats recall

The Mercari Tokyo ML interview rewards engineers who can **defend every choice under hostile probing** and who understand that **production ML is a cost-constrained, A/B-tested, drift-monitored craft**, not a model-zoo recital. Mercari's published work — the rerank blog, the 3B-item categorization, the Image Score paper, the MerRec dataset, the prohibited-items multimodal NAS, the AI Listing Support GPT-4-to-mini downgrade — is unusually candid about real production trade-offs, so reading it cover-to-cover is the highest-leverage prep. The single most distinctive insight a candidate can bring: **C2C marketplaces are not B2C** (inventory of 1, dual-role users, cold start everywhere, two-sided fairness, clicks too noisy to train on), and every modeling decision should reflect that. If your take-home is defensible to its last line, your system design covers data through cost, and you can quote Mercari's own choices back to the interviewer with the trade-offs that drove them, you will pass the technical bar.