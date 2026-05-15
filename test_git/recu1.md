# Cracking Recursive Inc.: A field guide for ML/LLM engineer candidates

**Bottom line:** Recursive Inc. (recursiveai.co.jp) runs a research-lab-flavored interview loop — take-home + LeetCode + a 30-minute AI paper presentation + a CEO/culture round — that fits your applied-LLM background very well, but rewards depth on **classical ML first-principles** (the most common rejection point) and on a **single research paper you can defend cold for 30 minutes of open-ended Q&A**. The role you'll actually apply to today is titled "Software Engineer" on their Breezy ATS — effectively an LLM Application Engineer — and it requires you to **already reside in Japan** (no overseas relocation right now). Engineering working language is English; Japanese (N3+) is a plus, not a gate. Compensation is undisclosed publicly but peer Tokyo AI startup data places it in the **¥7–13M total comp band** for mid-level, with stock options for eligible roles. Your Japanese-NLP, document-AI, and LangGraph agent experience is a strong fit for what Recursive actually ships (FindFlow RAG, TRUSTBANK Choice AI multi-agent system, invoice processing, kanjō-kamoku-adjacent enterprise classification). The single highest-leverage prep is: (1) drill ML fundamentals to first principles, (2) pick a paper deeply adjacent to your work and rehearse the 30-minute talk, (3) build one polished public artifact (blog post + GitHub repo) on Japanese-domain LLM work.

---

## 1. What Recursive actually is, and why it matters for your pitch

Recursive Inc. (株式会社Recursive / リカーシブ) is a **bootstrapped enterprise-AI consulting and product company** founded in August 2020 by **Tiago Ramalho** (CEO, ex-Google DeepMind Senior Research Engineer 2014–2017, ex-Cogent Labs Lead Research Scientist 2017–2020) and **Katsutoshi Yamada** (Co-founder and Chairman, ex-Cogent Labs sales director). Headquartered in **Shibuya S-6 Building 6F**, the company employs roughly **45–48 people from 22 nationalities**, with around **62% engineers and ~24% holding master's or PhDs**. The mission is explicitly **sustainability-aligned AI** — "build a fairer, more sustainable society through AI" — and the company is unusually impact-oriented relative to typical Japanese AI startups.

**Founder credentials matter for your prep.** Tiago is a Portuguese physicist (LMU Munich PhD in biophysics) whose verified DeepMind output is the *Nature* 2016 **Differentiable Neural Computer** paper (Graves et al.) and follow-on work on **few-shot learning** (Adaptive Posterior Learning with Marta Garnelo), **neural memory**, and **meta-learning**. The "AlphaGo team member (management)" claim on Recursive's bio is **not corroborated by AlphaGo author lists**, and direct authorship on Gato or AlphaFold could not be verified — treat those as **not confirmed**. His personal blog **tmramalho.github.io** is gold for interview prep: recent posts cover LLM tactics, agents, mBART fine-tuning, information retrieval, few-shot classification. **Skim it before any interview where he might appear.**

**The business is bootstrapped, not VC-backed.** Tiago stated in a May 2024 Sōgyō Techō interview that Recursive deliberately avoided VC funding "to always prioritize the mission." The single capital event is a **June 2024 strategic alliance with Rohto Pharmaceutical** (amount undisclosed; Crunchbase classifies as Seed). INITIAL.inc estimates post-money valuation at **~¥14.6B (~$95M)** as of July 2024, but this is inferred, not company-confirmed. Recognition: **Mizuho Innovation Award (July 2025)**, **OpenAI "Token of Appreciation"** for processing 10B+ tokens, and **Impact Startup Association** membership.

**Recursive's product and project portfolio is the single best signal of what you'll be hired to build.** Their three flagship platforms are:

- **FindFlow** — enterprise LLM/RAG for private corporate knowledge search (their canonical RAG product, launched ~6 months after ChatGPT's release).
- **Borealis** — physics-inspired neural networks producing high-resolution weather/environmental datasets.
- **Recursive AI Platform** — agentic AI combined with scientific simulation/optimization.

Recent named client work that you should be able to discuss intelligently in interviews: **TRUSTBANK Choice AI** (multi-agent furusato-nōzei gift recommender, featured as an OpenAI case study on openai.com); **Sumitomo Forestry × IHI** AI-physics groundwater model in Borneo; **Suntory** precipitation downscaling (paper accepted at NeurIPS 2025 "Tackling Climate Change with ML" workshop — "SpLIIF: Sparse Local Implicit Image Function for sub-km Weather Downscaling"); **COACH A** AI coaching-quality assessor; **KDDI** ad creative generation; **Kyōiku Dōjinsha** elementary math grading; **Nextorage** broadcast footage person-detection; **Fujitsu Learning Media** AI talent development; **KAIMRC Saudi Arabia** AI for tuberculosis. They're also **OpenAI Japan's first AI-native official development partner (July 2025)** and a **Google Cloud certified partner (July 2025)** — meaning they ship on OpenAI APIs and Vertex AI in production.

**Engineering culture in three lines:** working language is **English**; office model is **hybrid up to 4 days/week in Shibuya** with flex hours (core 10:00–16:00); all new hires sign a **one-year fixed-term contract that converts to permanent** on meeting performance standards. Benefits include 20 days PTO + 5 sick days, stock options for eligible roles, visa support, relocation assistance, language-learning support, and a referral bonus. Stated hiring values: **Challenger Mentality, Leadership, Trust & Cooperation, Impact**.

---

## 2. The role you'll actually apply to

Despite the historical existence of separately titled "Machine Learning Engineer," "Senior Machine Learning Engineer," and "LLM Engineer" requisitions on TokyoDev, the **only currently live engineering opening on Recursive's Breezy ATS** is titled **"Software Engineer."** This is effectively an **LLM Application Engineer** role — the scope is exactly what you've been doing at fpco-net.co.jp.

**The verbatim hard requirements:**

- Bachelor's degree **or** 10 years of industry experience.
- Experience in at least one systems language (Python, Go, Java, C++, Rust).
- Relational + NoSQL + **vector database** familiarity.
- One major cloud (GCP / AWS / Azure).
- Demonstrable experience **building and deploying at least one application powered by an LLM** — explicitly allows personal projects or open-source contributions, not just professional work.
- Benchmarking and quantitative analysis methods.
- **Must currently reside in Japan with a valid visa.** No overseas relocation in this requisition.
- **English: fluent.** **Japanese: N3+ "a plus."**

**The verbatim responsibilities** map cleanly onto your LangGraph/Gemini/RAG stack: prompt and pipeline engineering with LangChain or LlamaIndex; LLM integration and deployment (proprietary and open-source); RAG data pipelines, vector stores, embedding strategies; backend/API development; A/B testing of prompts and models for latency/cost; cloud deployment on GCP/AWS/Azure.

**Notable absences from the JD:** no required PyTorch, TensorFlow, or JAX. **This is not a foundation-model training role** — it's applied. Don't oversell training experience; do oversell production deployment, evaluation, and orchestration.

**Two adjacent open roles** (not engineering but useful to know): **Business Solutions Architect** requires **JLPT N2** Japanese and a TOEFL 90+/TOEIC 750+ — significantly stricter than engineering — and **People Operations Associate**.

**Salary is undisclosed everywhere** — Japan Dev, Breezy, Built In, Wantedly, Glassdoor, Levels.fyi all show no band. Peer-company calibration places mid-level Recursive engineers in the **¥8–13M total comp** range with stock options.

---

## 3. The interview loop, decoded

Nine Glassdoor interview reviews (difficulty 3.3/5, only ~33–40% positive experience), one TokyoDev employee profile (Xyza Rivera), Recursive's own published process descriptions, and two public interview GitHub repos (`recursiveai/review_me`, `recursiveai/recursive-interview-public`, the latter active as of Dec 2025) give a clear picture. **Average time-to-hire: 33 days overall, but ML Engineer averages 51 days vs Software Engineer ~14 days.**

### The MLE / LLM Engineer track (4–5 rounds, ~6–8 weeks)

1. **Application screen / recruiter intro.**
2. **ML take-home ("homework")** — ~4 questions on ML modeling and optimization, including **"implement basic NN components from scratch"** and "first-principles reasoning when training a very small neural network."
3. **Python LeetCode round** — 3 easy-to-medium problems. Status appears variable; one May 2025 candidate reported this round was "actively stopped" in some loops.
4. **Homework review call** — interviewer walks through your take-home, then drills fundamentals: **SGD and hyperparameters affecting training, classification vs regression loss, gradient descent, optimizers, neural-network basics**. **This is the most-reported rejection point.**
5. **30-minute AI paper presentation + culture call** — you pick a recent AI paper, present for 30 minutes, then take Q&A that **frequently strays outside the paper itself**, requiring broad ML literacy. The culture call (often with CEO/leadership) then asks: desired salary, desired location (Japan vs abroad), visa-sponsorship needs, and behavioral questions like **"Tell me about a moment in your life where you utterly failed"** and **"Why do you want to join a startup."**
6. **Final round.**

### The Software Engineer / Fullstack track (~2 weeks)

Recursive's own published process: recruiter screen → a paired **frontend + backend** technical (won't disqualify for weakness on one side) → cultural-fit. Glassdoor candidates report this in practice as a **system-design take-home**, a **frontend live coding**, a **backend code review**, and a **chat with CEO Tiago**.

**Cross-cutting facts:**
- **The founder interviews routinely** — Xyza Rivera's first round was with Tiago directly; CEO chat appears at culture-stage for MLE and early/final for SWE.
- **The 30-minute paper presentation is Recursive's signature round** — uncommon outside research labs and clearly imported from Tiago's DeepMind background.
- **Compensation discussion is embedded inside the culture round**, not handled separately post-offer. Be prepared with a number.
- **Engineering interviews are conducted in English.** No Japanese-only engineering interview track is evidenced; Japanese-language interviews are reserved for client-facing roles per OpenWork listings.
- Multiple candidates reported **friendly tone but slow scheduling** (slots often >2 weeks out) and **offer compensation lower than expected**.

---

## 4. What the technical rounds actually test — and how to prepare

### Classical ML fundamentals (where most candidates fail)

The take-home review is the single most consistent rejection point. The bar is **first-principles fluency**, not framework recitation. Prioritize:

- **Optimization**: SGD vs momentum vs Adam/AdamW; what each hyperparameter does *and why*; learning-rate schedules (cosine, warmup, linear); the bias-correction terms in Adam; when batch size affects generalization (linear scaling rule, generalization gap).
- **Bias/variance and regularization**: L1 vs L2 mathematically and geometrically; dropout's interpretation as ensemble approximation; weight decay vs L2 in Adam (decoupled weight decay in AdamW); early stopping; data augmentation as implicit regularization.
- **Losses**: cross-entropy derivation from MLE; why MSE for regression and not classification; focal loss intuition; KL divergence and its asymmetry; label smoothing.
- **Backprop from scratch**: be able to whiteboard the backward pass of a 2-layer MLP, and to derive softmax+cross-entropy gradient cleanly (the canonical `(p - y)` simplification).
- **Statistics primers**: bias of an estimator, MLE vs MAP, central limit theorem, p-values vs effect size, A/B test pitfalls (peeking, multiple testing, Simpson's paradox).
- **Evaluation**: precision/recall/F1 derivations, ROC vs PR curve and when each lies, calibration and reliability diagrams.

**Recommended materials:** Bishop's *PRML* (Ch. 1–4 only), Goodfellow *Deep Learning* (Ch. 5–8), Andrej Karpathy's "Zero to Hero" YouTube series (especially *micrograd* and *makemore* — these are exactly the "implement NN components from scratch" the take-home asks for), Chip Huyen's *Designing Machine Learning Systems* for production framing, and the Machine Learning Tokyo **EN-JP-ML-Lexicon** for Japanese term mapping.

### Deep learning and transformer internals

Expect questions on **attention** (scaled dot-product derivation, why scale by √d_k, multi-head intuition, attention complexity O(n²d) and why), **positional encodings** (sinusoidal vs learned vs RoPE vs ALiBi — RoPE is the dominant one in modern Llama/Qwen/Swallow models), **normalization** (LayerNorm vs RMSNorm, pre-norm vs post-norm stability), **KV cache** (memory math, why it dominates inference memory at long context), **tokenization** (BPE vs Unigram vs SentencePiece, why Llama-2 tokenizers fragment Japanese into ~2–3 tokens/char, and how Swallow's vocab extension fixed this for Llama-2 while Llama-3-Swallow kept the original 128k vocab unchanged).

### LLM Engineer specifics — your strongest territory

Given your LangGraph + Gemini + LoRA/QLoRA + RAG + RAGAS background, this is where you should dominate. Be ready to discuss with first-principles depth:

- **Fine-tuning**: LoRA math (why low-rank ΔW = BA works, rank/alpha tradeoffs, target modules choice — q_proj/v_proj vs all linear), QLoRA's 4-bit NF4 quantization and double-quantization, the **Unsloth optimizations** you've used (custom Triton kernels, ~2x speedup), why your **Llama-3-Swallow + LoRA** for kanjō-kamoku classification was the right base-model choice (vocab efficiency for Japanese tokens). Know SFT vs DPO vs PPO/RLHF tradeoffs and when each is appropriate.
- **RAG**: chunking strategies (fixed-size vs semantic vs structural vs late-chunking), embedding model selection (multilingual-E5, BGE-M3, OpenAI text-embedding-3, **JaColBERT for Japanese late-interaction retrieval**), hybrid retrieval (BM25 + dense), reranking (cross-encoders, Cohere Rerank, BGE-reranker), query rewriting/HyDE. Be able to discuss **failure modes you've actually hit** in production.
- **Evaluation**: RAGAS metrics (faithfulness, answer relevancy, context precision/recall) — explicitly know what each measures and where each fails; LLM-as-judge bias (position bias, verbosity bias, self-preference); ground-truth construction; offline vs online evals; **Japanese-specific eval** via llm-jp-eval, Japanese MT-Bench, and Nejumi 4 (which tests JMMLU with three patterns including "select the wrong answer" to score response consistency).
- **Agents**: LangGraph state-machine architecture vs ReAct vs plan-and-execute; tool/function calling (OpenAI vs Gemini schemas, parallel calls); multi-agent patterns (router + specialist agents, exactly the TRUSTBANK Choice AI architecture); failure-recovery and observability (LangSmith/Langfuse/Arize Phoenix).
- **Inference optimization**: quantization (GGUF/AWQ/GPTQ vs bitsandbytes), vLLM and PagedAttention (KV-cache memory management), continuous batching, speculative decoding, FlashAttention, prefix caching, and the practical cost/latency tradeoffs of GPT-4o vs Gemini Flash vs self-hosted Llama-3.1-70B.

### Japanese-language technical interview prep

Even though Recursive's engineering interviews run in English, expect mixed Japanese where a hiring manager or Japanese teammate joins, and absolutely expect Japanese-domain technical content. The vocabulary cheat sheet to memorize (most useful pairs): **過学習** (overfitting), **正則化** (regularization), **損失関数** (loss function), **勾配降下法** (gradient descent), **誤差逆伝播法** (backprop), **注意機構** (attention), **埋め込み** (embedding), **事前学習** / **ファインチューニング** (pre-training / fine-tuning), **量子化** (quantization), **蒸留** (distillation), **検索拡張生成** (RAG), **大規模言語モデル** (LLM), **基盤モデル** (foundation model), **継続事前学習** (continual pre-training), **語彙拡張** (vocabulary expansion), **ハルシネーション** (hallucination), **関数呼び出し** / **ツール利用** (function calling / tool use), **生成AI** (generative AI). In conversation, **mix katakana loanwords freely** — Japanese ML engineers naturally say アテンション機構, トランスフォーマー, ファインチューニング, KVキャッシュ rather than forcing Japanese translations.

**Japanese-domain technical content you should be able to discuss fluently:**

- **Tokenization landscape**: MeCab (IPADIC vs UniDic vs NEologd dictionaries), **Sudachi's three split modes (A/B/C)** and why A+C are combined in search indexes (A for recall, C for precision via NER-unit), Juman++'s RNN-LM advantage on ambiguous text, **fugashi** as the modern Python wrapper, **GiNZA** as the spaCy-based unified pipeline, SentencePiece for LLMs (Unigram preferred over BPE for Japanese), and **Llama tokenizer vocab-expansion** for Japanese (Swallow's design choice).
- **Preprocessing pipeline**: `unicodedata.normalize('NFKC', s)` → `neologdn.normalize()` → tokenizer. Mention this casually and you sound like an insider.
- **Japanese LLMs**: Llama-3.3-Swallow 70B (current best open Japanese model), ELYZA, Karakuri, rinna Nekomata/Bakeneko, PLaMo-100B (PFN), Sarashina (SB Intuitions, with **Sarashina2.2-OCR** for documents), Stockmark-2-100B (MIT license), CALM3-22B (CyberAgent), llm-jp-3/4, Tanuki, Fugaku-LLM, NTT tsuzumi.
- **Benchmarks**: JGLUE (MARC-ja, JSTS, JNLI, JSQuAD, JCommonsenseQA), **llm-jp-eval / jaster**, **Japanese MT-Bench**, **Nejumi LLM Leaderboard 4** (Dec 2025; adds ARC-AGI/-2, JMMLU-Pro, Humanity's Last Exam, SWE-Bench Verified, BFCL, JTruthfulQA, JBBQ), JMMLU, ELYZA-tasks-100.
- **Japanese-domain ML problems** — your sweet spot: **kanjō-kamoku (勘定科目) classification** with notation-variance (表記揺れ), per-tenant taxonomies, and few-shot LLM + fuzzy vendor matching; **invoice processing** under the new **インボイス制度** (Qualified Invoice System, Oct 2023, requires extracting T+13-digit 登録番号); 印鑑/stamp overlap; mixed 全角/半角 digits; 縦書き (vertical text); **address normalization** (Geolonia's `normalize-japanese-addresses` is the famous reference — Kōno Tarō publicly called it "the yabai problem"); **keigo handling** (avoid 二重敬語 like "お読みになられる"); **MyNumber/PII** under APPI law.

---

## 5. ML system design — likely question patterns

Recursive's product portfolio tells you exactly what to prepare. Expect prompts in the family of: **"Design a RAG system for a Japanese law firm's contract database,"** **"Design TRUSTBANK Choice AI from scratch — a multi-agent gift recommender,"** **"Design an evaluation harness for a customer-support LLM in Japanese,"** **"Scale FindFlow to handle 10M documents and 1k QPS at <2s p95,"** **"Build an invoice-processing pipeline robust to any layout."**

**What evaluators look for** (based on Recursive's stated focus on benchmarking, A/B testing, cost-effectiveness, and the take-home's "first-principles" framing):

1. **Crisp problem framing** — restate, ask about scale (users, documents, QPS), latency budget, accuracy bar, cost budget, languages.
2. **Data and ingestion** — chunking strategy *with justification for this domain* (legal documents → structural/hierarchical chunking, not fixed-size; invoices → layout-aware with key-value extraction).
3. **Retrieval architecture** — embedding choice with Japanese-specific reasoning (multilingual-E5 vs JaColBERT vs domain-tuned), hybrid BM25+dense, reranker tier-2.
4. **Generation layer** — model choice with explicit cost/latency math (Gemini Flash vs GPT-4o vs self-hosted Swallow-70B-AWQ on 2×A100), prompt structure, structured output (JSON schema with function calling).
5. **Evaluation strategy** — this is critical at Recursive given the JD's "benchmarking and performance" emphasis. Walk through: offline eval set construction (~200 hand-labeled queries), RAGAS or custom faithfulness/precision/recall, LLM-as-judge with explicit bias mitigation, online A/B with guardrail metrics, regression suite in CI.
6. **Production concerns** — observability (token usage, latency p50/p95/p99, cost per query, hallucination rate), caching (semantic cache, prefix cache), failure modes (vector DB cold start, embedding-model drift), red-team/safety, **PII redaction before prompting** (mandatory for Japanese enterprise — APPI).
7. **Iteration story** — "v1 ships in 2 weeks with X, v2 adds reranker, v3 adds fine-tuned embedding once we have ~10k user feedback signals."

**Cost/latency awareness is a Recursive-specific signal**: the JD explicitly names "performance, latency, and cost of different LLM prompts and models" as a responsibility. Always close a design with **a back-of-envelope cost-per-query** and **latency budget breakdown**.

---

## 6. Behavioral and culture-fit

Recursive's stated values — **Challenger Mentality, Leadership, Trust & Cooperation, Impact** — and its sustainability-aligned mission are **treated as a real screen**, not boilerplate. The Nov 2023 candidate quote captures it: "If you're into AI with a focus on sustainability and don't mind a longer interview process, it's worth checking out." Be ready to articulate, sincerely, why **AI for sustainability/social impact** matters to you — connect it to concrete actions (e.g., your activation-steering experiments for interpretability/alignment, your domain choice of Japanese SME accounting which democratizes back-office automation).

**Expected behavioral questions** (directly reported):
- "Tell me about a moment in your life where you utterly failed." — prepare a STAR-format story with a *clear failure*, what you learned, and what changed. Do not soft-pedal.
- "Why do you want to join a startup?" — connect to ownership, breadth, speed; avoid mercenary framing.
- Desired salary, location, visa needs — have a number ready (see §10).

**Communication-style calibration for a mixed Japanese/international team:** Recursive's English-first culture means you can be **more direct than typical Japanese business norms** — but not Silicon-Valley blunt. Lead with conclusions ("結論から申し上げますと..." pattern even in English: "The short answer is..."); flag tradeoffs explicitly; ask clarifying questions before diving in; never disagree by saying "no" — say "I'd push back on that because..." with reasoning. With a 22-nationality team, **defaulting to clear English with technical katakana loanwords** in mixed-Japanese discussion will read as professional, not foreign.

**Customer-facing skills matter.** Recursive is consulting-flavored; engineers regularly interface with client PMs. The TRUSTBANK, COACH A, Suntory, and Sumitomo Forestry case studies all involve technical engineers presenting to client stakeholders. Have one story ready about explaining a technical tradeoff to a non-technical stakeholder.

---

## 7. Near-term prep plan (next 1–3 months)

### Weeks 1–2: foundations refresh
Work through Karpathy's "Zero to Hero" series end-to-end — implementing micrograd and makemore in a clean public repo is **the single best signal** for Recursive's "implement basic NN components from scratch" take-home expectation. In parallel, drill **NeetCode 75 in Python** at 1–2 problems/day; aim for fluency on arrays, hashmaps, two pointers, sliding window, basic graphs/DFS/BFS, and dynamic-programming basics — that's the LeetCode-easy/medium bar. Read or skim **Chip Huyen's *Designing Machine Learning Systems*** for production framing.

### Weeks 3–4: LLM specialization and Japanese-NLP depth
Read in this order, taking notes you can defend:
1. *Attention Is All You Need* (Vaswani 2017) — be able to derive scaled dot-product attention.
2. *LoRA* (Hu 2021) — math of low-rank decomposition.
3. *QLoRA* (Dettmers 2023) — 4-bit NF4 + double quantization.
4. *RAG* (Lewis 2020) original paper.
5. *RAGAS* paper.
6. *Continual Pre-Training for Cross-Lingual LLM Adaptation* (Fujii et al., COLM 2024 — the **Swallow paper**, directly relevant to your Llama-3-Swallow work). Read it twice; this becomes a candidate for your paper-presentation round.
7. *The AI Scientist* (Sakana AI) or *Differentiable Neural Computer* (Graves et al., Nature 2016 — Tiago's own paper, a thoughtful choice if you can engage seriously with it).

### Weeks 5–8: portfolio polish and the paper talk
**Pick your paper presentation now.** The two strongest candidates given your background:
- **Llama-3-Swallow / Continual Pre-Training paper** (Fujii et al.) — connects directly to your kanjō-kamoku fine-tuning work and lets you compare continual pre-training vs LoRA adaptation as design choices.
- **A RAG-evaluation paper** (RAGAS or "Retrieval Augmented Generation or Long-Context LLMs?" by Li et al.) — connects directly to FindFlow.

Practice the 30-minute talk **on video, three times, with broad Q&A drills** — partner up via Tokyo AI (TAI) AMA subgroup or a Discord ML community.

**Portfolio repos to clean up** (be ruthless — quality over quantity):
- **Kanjō-kamoku classifier** — README in English with a 2-paragraph Japanese summary, eval numbers vs baselines (BM25, multilingual-E5 nearest-neighbor, Llama-3-Swallow zero-shot, your fine-tuned version), confusion matrix, cost/latency table. **This is your headline project for Recursive** — it overlaps directly with their invoice-processing work.
- **LangGraph multi-agent system** — frame as a parallel to TRUSTBANK Choice AI's router + specialist pattern; document the state-machine design explicitly.
- **Activation steering on Gemma** — frame as alignment/interpretability work; this differentiates you and aligns with Recursive's sustainability/impact framing.
- **Neural style transfer on Android** (Gatys → Johnson → AdaIN → ExecuTorch) — keep but de-emphasize; it's a strong production-ML story but not Recursive-relevant. One blog post is enough.

**Write one technical blog post**, in English with Japanese title (post on Zenn and your personal site). Recommended topic: *"Fine-tuning Llama-3-Swallow for kanjō-kamoku classification: tokenizer choices, LoRA target modules, and evaluation on a per-tenant taxonomy."* This single artifact would be the most differentiating thing you produce. Optionally post a 日本語版 to Zenn or Qiita for community visibility.

**Application logistics.** Apply via Recursive's Breezy.hr portal (`recursive.breezy.hr`). **Get a referral if possible** — search LinkedIn for "Recursive Inc" current engineers (Pablo, Head of Research Engineering; Menna, ML Engineer; Matthew Whalley, software engineer in the OpenAI/TRUSTBANK case study) and reach out with a focused, specific message about their work. Mention you've **read the OpenAI TRUSTBANK case study** and the Swallow paper, and you'd value a 15-minute coffee. Send a **short cover letter** (5–7 sentences max): one line on your fpco-net production work, one on the kanjō-kamoku project (with link), one on the activation-steering work, one on why Recursive specifically (FindFlow + sustainability angle), one closer with availability.

---

## 8. Common mistakes and red flags to avoid

The Glassdoor data and Recursive's hiring patterns reveal a clear set of failure modes. **The biggest is over-reliance on framework knowledge** — candidates who can describe LangChain abstractions but stumble on cross-entropy derivation fail the take-home review. Always be ready to **drop one level of abstraction** when challenged.

**Do not present a paper you cannot defend cold.** The Q&A "drifts to unrelated topics" — interviewers are testing whether you actually understand the field around the paper, not just the paper. If you've never trained a transformer end-to-end, don't pick a pre-training paper.

**Do not undersell your production work.** Your fpco-net experience building internal AI tools (Django + AWS/Azure + Gemini + Document Intelligence) is **directly what Recursive ships**. Frame it in business terms: users served, time saved, error rates reduced. Many ML candidates fail by sounding too academic when Recursive is actually a consulting/applied shop.

**Calibrate Japanese communication carefully.** With international engineers in the room, default to English; switch to Japanese only when a Japanese teammate signals it. **Use です・ます-form throughout** — don't strain for 尊敬語/謙譲語 unless you're already fluent (二重敬語 errors are worse than plain 丁寧語). Avoid translating technical terms into stiff Japanese (誤差逆伝播法 in speech sounds academic; バックプロパゲーション is natural).

**Watch for these red flags interviewers explicitly screen for:** name-dropping papers you haven't read (the paper round will expose this); claiming production experience without specifics on scale/latency/cost; vague evaluation answers ("we used accuracy" — they want eval-set construction details); inability to say "I don't know" cleanly; ignoring cost/latency when designing systems.

**Salary negotiation pitfalls in Japan:** Don't open with a number lower than your target (you cannot revise up in Japanese negotiation culture). Don't conflate base with total comp — Japanese companies often pay contractual 2–4 month 賞与, so "¥8M base" may be ¥10M+ total. Confirm whether the bonus is **proper 賞与** or token **寸志**. Ask whether stock options are NSO/ISO equivalents and what the recent 409A-equivalent valuation looks like (¥14.6B INITIAL estimate is your reference). For HSP visa points, base salary buys points (¥10M = 40 points); push for base, not bonus.

**Don't ask in interviews:** anything answerable from the website (lazy signal); detailed visa logistics beyond confirming sponsorship (HR handles, not engineers); WLH-style questions before you have an offer. **Do ask:** "What's the single hardest technical problem the team is working on right now?"; "How does the team handle evaluation when ground truth is ambiguous (kanjō-kamoku per-tenant taxonomies, coaching-quality assessment)?"; "What does the engineering split between MLE and SWE feel like in practice?" — these signal you've read the case studies.

---

## 9. Long-term positioning (3–12 months)

If you don't get Recursive on the first attempt, or you want to broaden your options, here's the multi-quarter play.

**The Japanese-NLP niche is your moat — own it.** Few non-Japanese engineers can speak fluently to MeCab/Sudachi/SentencePiece tradeoffs, Swallow's continual-pretraining choices, invoice OCR under the インボイス制度, address normalization, or kanjō-kamoku taxonomy variance. **Most international ML engineers in Tokyo cannot.** Double down: contribute to **llm-jp-eval** (add a benchmark task, fix a tokenizer edge case), publish a Speaker Deck slide on a Japanese tokenizer comparison, or release an open kanjō-kamoku benchmark dataset (~5k labeled examples would be a community first).

**Open-source contributions to target, in priority order:**
1. **llm-jp/llm-jp-eval** — the de-facto Japanese LLM eval framework; small PRs (new task, fix, doc) get noticed by Stockmark/PFN/ELYZA hiring managers.
2. **Swallow** (Okazaki lab) — corpus pipeline contributions or eval scripts.
3. **fugashi / SudachiPy** — tokenizer ecosystem PRs are high-signal for any JP NLP team.
4. **JaColBERT** or **multilingual-E5** Japanese fine-tunes — embedding model work is differentiated.

**Publishing and blogging strategy.** Target **one substantive technical post per month** on Zenn (Japanese tech community, high SEO and discovery) or your personal site, alternating English and Japanese. Topic ideas, in descending leverage: (1) per-tenant fine-tuning for Japanese enterprise NLP, (2) RAG evaluation under ambiguous ground truth, (3) activation steering for safer Japanese chat models, (4) ExecuTorch on-device LLM serving on Android. **Cross-post to Hacker News and Reddit r/LocalLLaMA** for international visibility.

**Tokyo AI community immersion.** Three communities are essential:
- **Tokyo AI (TAI)** at tokyoai.jp — Ilya Kulyatin's invite-only "TAIT" plus open subgroups (AAI, **AMA — Applied ML**, AHR, AIST). English-first, the single best entry point.
- **AI Tinkerers Tokyo** — demo-driven monthly meetup with Sakana, Mercari, Indeed, Woven, Shisa.AI presence.
- **LLM-jp meetings (LLM勉強会)** — NII-led, ~17 meetings through Mar 2025, 1,000+ participants; mostly Japanese-language but international researchers attend. Submit a working-group contribution.

Annual conferences worth attending or submitting to: **JSAI** (人工知能学会全国大会, ~3,000 attendees, the flagship general AI venue), **NLP2026 in Utsunomiya (Mar 2026, 32nd Annual Meeting of the ANLP)** — submitting a poster, even with limited Japanese, is a powerful network-builder because authors switch to English 1:1; **YANS 2026** (NLP若手の会, 570 attendees in 2025) for early-career networking with named awards from CyberAgent/ELYZA/Polaris.AI.

**When to apply vs keep building.** Apply now to Recursive (the live SWE/LLM-app role fits your profile and you'll learn from the loop even if you don't pass), but **also batch-apply to backup options** to create competing-offer leverage and calibrate market signal. If you reach the take-home review and stumble on fundamentals, do not panic — that's diagnostic, not terminal. Rework Karpathy's series, retry in 3 months.

**Backup employers to apply to in parallel** (most-to-least similar to Recursive):
- **Citadel AI** — closest culture analog: English-internal, no Japanese required, equity for early hires, AI testing/monitoring focus. **Apply.**
- **Sakana AI** — research-prestigious; full-time roles require **Japanese fluency** but internships/student-researcher tracks don't. Highest brand value if you can get in.
- **Stockmark** — Japanese enterprise LLMs; built Stockmark-2-100B (MIT license); Series D ¥4.5B from Polaris; **business-level Japanese typically required**.
- **ELYZA** (KDDI subsidiary, Matsuo Lab origin) — postings in Japanese on HERP Careers; **de-facto Japanese requirement** but technically world-class.
- **Preferred Networks (PFN)** — strongest pure ML lab in Japan, foundation models on MN-Core; sponsors visas; ¥12–17M for AI roles. Apply if you want training-stack depth.
- **LegalOn Technologies** — post-Series E ($200M total, $50M E round); legal AI; ¥10B+ ARR; OpenAI strategic collaboration.
- **Rinna** — Microsoft spin-off, open-source-strong (Nekomata, Bakeneko 32B with 6M+ HF downloads).
- **Karakuri**, **Lightblue**, **PKSHA**, **Cinnamon AI**, **Studio Ousia (LUKE)**, **Allganize Japan**, **Mantra**, **CyberAgent AI Lab (CALM3)**, **SB Intuitions (Sarashina, Sarashina2.2-OCR)**, **Money Forward** (fintech ML — directly relevant to your kanjō-kamoku work).

---

## 10. Salary and negotiation, in numbers

Use these benchmarks when Recursive asks for your desired comp during the culture call.

The **TokyoDev 2025 Developer Survey** (n=989, international developers in Japan) places median total comp at **¥9.5M**, with Senior Engineer median **¥10.5M** and Engineering Manager median **¥12.5M**; international-no-Japan-ops employers pay median **¥13.5M** (47% premium over Japanese-HQ). **Glassdoor Tokyo ML Engineer** (Mar 2026, n=107): average ¥8.0M, 75th percentile ¥10.0M, 90th percentile ¥13.1M. **ERI SalaryExpert Japan ML Engineer:** average ¥10.7–10.8M; entry (1–3 yrs) ¥7.5M; senior (8+ yrs) ¥13.3M with average bonus ¥492k.

**Practical bands for ML/LLM engineers in Tokyo (2025–26):**
- **Junior (1–3 yrs)**: ¥5.5–8M base.
- **Mid (3–6 yrs)**: ¥8–13M total comp.
- **Senior (6+ yrs)**: ¥12–20M at Japanese AI startups; ¥18–30M+ at FAANG/Indeed/Woven by Toyota with equity.

Peer companies for calibration: **Woven by Toyota ML Engineer median ¥16.6M (range ¥11.4M–¥22.5M+)**, **Mercari SE ¥7.4M (MG1)–¥14.9M (MG4)**, **Indeed Tokyo SE median ~¥26.7M** (outlier high), **PFN ~¥14M median per Levels.fyi**. **Sakana AI Data Scientist ~¥20M** (single data point, directional only).

**For Recursive specifically:** No salary is publicly disclosed anywhere. One Glassdoor candidate noted offer comp was "lower than I expected." Based on peer Series A/B Tokyo AI startup norms and Recursive's bootstrap funding, target **mid-band**: ask for **¥10–12M total comp** if you have 3–5 years engineering experience with strong LLM/RAG production work; ¥12–14M if you can credibly position as senior. **Push for base, not bonus** — better for HSP visa points (¥10M base = 40 HSP points), better for next-job anchoring, and more meaningful than Japanese-startup stock options.

**Equity reality check.** Recursive offers stock options "for eligible roles" but IPO timelines for bootstrapped Japanese AI startups are long, and secondary markets are weak. Treat options as **lottery upside**, not core comp. Negotiate cash.

**Negotiation framing for the culture round:** "Based on TokyoDev 2025 data and my LLM production experience at fpco-net plus my Japanese-domain NLP portfolio, I'm targeting ¥X total comp. I'm flexible on the mix between base and bonus, but I'd prioritize base given the HSP visa structure. Stock options would be a welcome addition." This signals informed, reasonable, and visa-aware — exactly the candidate profile Recursive's international-bias HR is set up to handle.

---

## Conclusion: what to do this week

Recursive is **interview-process-distinctive** (paper presentation + first-principles take-home + founder involvement) but **role-profile-conventional** (applied LLM engineer, English-first, stock-options-included). Your fpco-net production work + LangGraph + Llama-3-Swallow LoRA + RAGAS + Japanese document AI is **a strong on-paper match** for the live Software Engineer requisition, with one caveat: **the take-home review of ML fundamentals is the most consistent failure point**, and your most likely gap. The single most leveraged action this week is to commit to **30 days of Karpathy's "Zero to Hero" + a clean public micrograd/makemore repo**, paired with picking and rehearsing one paper (Llama-3-Swallow or RAGAS) for the 30-minute talk. Then apply via Breezy with a referral if possible, batch-apply to Citadel AI / PFN / Stockmark / LegalOn / Money Forward as parallel options, and use any rejection feedback as your roadmap. Your Japanese-NLP niche is the long-game moat — own it through one shipped open-source contribution to llm-jp-eval and one well-written blog post on kanjō-kamoku fine-tuning, and within 6 months you'll be a candidate Recursive's hiring manager actively wants to interview rather than the other way around.