# CV/ML Engineering in Japan: A nine-company field guide

This report is the result of deep research across job postings, engineering blogs, GitHub repos, employee accounts, Glassdoor/Levels.fyi/TokyoDev/Japan-Dev, and press releases on nine Tokyo-area CV/ML employers. **The bottom line: for a Python/PyTorch SWE with LoRA, RAG/LangGraph, Gemini, Azure Document Intelligence, on-device CV (ExecuTorch Android style transfer) and LLM activation-steering work, the highest-conviction near-term targets are PayPay, Mercari (Edge AI / AI-LLM team), Citadel AI, and Recursive.** MUJIN, Tier IV, and Woven by Toyota require 12–18 months of focused 3D-CV + production-C++ upskilling. Ubie and Algomatic offer the best GenAI tech fit but the worst English-language fit. The rest of the report breaks each company down across ten dimensions, with a comparison table and prioritized application order at the end.

---

## 1. Mercari (TSE Prime: 4385)

### Company snapshot
Japan's largest C2C marketplace; group revenue **¥187B FY2024**, **23M+ MAU**, **4B+ cumulative listings**. Mercari runs an explicit AI organization of ~35+ ML engineers in Tokyo plus a separate Merpay ML team. Sub-teams visible on `ai.mercari.com/en/people`: **AI/LLM (Max Frenzel)**, Search, Recommendation, **Item Understanding (Ryan Ginstrom)**, **Image Search & Edge AI (Yusuke Shibui/Chica Matsueda)**, **Trust & Safety ML (Bosco Yung)**, ML Platform, and Applied AI Research. ~40% of engineers are non-Japanese, from 40+ countries. **English is the preferred meeting language**, with a Language Education Team running three English courses plus Japanese coaching and "Yasashii Communication" seminars. Recent context: Mercari **Hallo was discontinued (service ends Dec 18, 2025)** after launching March 2024; Mercari US suffered **a ~45% layoff in June 2024 plus a second 20% cut** with 60% of US@Tokyo team shifted back to Japan; founder Yamada took over US CEO Jan 2025; the new **Mercari Global App** launched Sept 26, 2025. The "AI-Native culture" push and "Customer Perspective" addition to the Culture Doc are the headline 2025 themes.

### Active CV/ML roles
Mercari does not use the "Computer Vision Engineer" title; CV work lives inside ML Engineer roles. Recurring titles include **Software Engineer (Machine Learning & Search)**, **SWE (ML & Recommendation)**, **Senior SWE (ML & Recommendation)** (Tokyo/Bengaluru hybrid), **Machine Learning Platform Engineer**, **Engineering Manager – Machine Learning**, **ML Engineer – Item Understanding/Catalog**, **ML Engineer – Trust & Safety**, **AI/LLM Engineer**, **Search Engineer (ML)**, and **Edge AI Engineer**.

### Hard skills
Python + **Go** (Mercari is famously a Go shop and ML services are fronted by Go microservices), PyTorch/TensorFlow, scikit-learn. Heavy GCP: **GKE, BigQuery, Dataflow, Pub/Sub, Vertex AI, Vertex AI Vector Search**. **Kubeflow** is the canonical pipeline tool — Subodh Pushkar contributes upstream. The Dec 2023 engineering.mercari.com post documents fine-tuning **CLIP** to replace InceptionV3 for listing-image embeddings (~50–100M images, 512→64 dim reduction yielding "**80% More Budget-Friendly**" indexing). Edge AI uses TFLite + MediaPipe; AI Listing Support uses **GPT-4o mini** per the OpenAI case study. Search ranking uses BERT + GBDT.

### Soft skills
English-first, design-doc heavy ("DevDojo" formally teaches RFCs), blameless culture, GitHub-based review, and a strong A/B experimentation culture. Japanese is not required day-to-day; visa + relocation provided.

### Day-to-day
Per JDs and engineer bios: design doc → BigQuery exploration → PyTorch/Kubeflow training on GKE → A/B experiment via internal platform → online evaluation → release of a Go-fronted model microservice. Bosco Yung (T&S): *"ML solutions for item moderation and other trust and safety quests."* Teo Zosa (Search): "architecting and implementing highly scalable and resilient ML systems."

### Junior vs mid
Mercari publishes its **Engineering Ladder** (github.com/mercari/mercari-engineering-ladder) and uses MG1→MG5 IC grades with MG6/MG7 Principal/Distinguished. **MG1** = new grad / 0–2 yrs, scoped tasks under a TL. **MG2** ~2–4 yrs, owns feature. **MG3** Senior ~4–7 yrs, owns project across services, mentors. **MG4** Staff, 7+ yrs. Senior ML (Rec) JD explicitly asks for "**3+ years experience developing applications in Python and one of: Go, C/C++, Java**" plus DL/LLM production experience.

### Interview process
Recruiter screen → **HackerRank/Codility** (~75 min, 2 problems, medium-hard; one Glassdoor 2024 candidate reported LeetCode-Hard bit manipulation + graph DP) → **take-home ML assignment (~1 week)** → 1.5h technical with 2 engineers → **ML system design** → behavioral + EM round. Total 3–8 weeks, all in English. Take-homes are the most cited filter.

### Compensation
Levels.fyi (Greater Tokyo, March 2026): MG1 ~**¥7.44M** total, MG2 ~¥9–10M, MG3 ~¥11–12M, MG4 ~¥12.7–18.7M. ML Engineer band Japan: **¥8.12M (MG1) to ¥18.43M+**. **RSUs vest 33.3%/yr** (TSE Prime listed). Semi-annual performance bonus (~¥0.5–1.5M at MG1–3). Visa sponsorship, pre-join English classes, post-join Japanese classes, "Your Choice" remote/office.

### Application channels
`careers.mercari.com/en/jobs/` (primary), LinkedIn, TokyoDev, Japan Dev, Wantedly, Built In / Relocate.me / echojobs (reposts), referral program, **Build@Mercari** paid intern with visa/relocation.

### Fit assessment
**Strong NOW (MG1–MG2):** the **Edge AI / Image Search team** is a near-perfect fit — the candidate's ExecuTorch Android style-transfer experience is genuinely rare and maps directly to TFLite/MediaPipe on-device inference. **AI/LLM Team** (Frenzel) for AI Listing Support / AI Assistant maps to LangGraph/RAG/Gemini. **Item Understanding** maps to CLIP fine-tuning (LoRA/QLoRA transfer) and **Trust & Safety ML** maps to multimodal image+text classification. **Plausible mid (MG3) in 12–18 months:** Search ML, Recommendation, ML Platform — each requires more learning-to-rank, causal inference, or production Kubernetes ownership. **Gaps to fill:** Go (build 1–2 production Go services), Vertex AI / Vertex AI Vector Search (reproduce the "80% More Budget-Friendly" CLIP blog), Kubeflow in production, and ideally an A/B experimentation portfolio. Avoid Mercari US/US@Tokyo (heavy cuts) and Hallo (discontinued).

---

## 2. Recursive (recursiveai.co.jp)

### Company snapshot
Tokyo-headquartered applied AI consulting firm founded August 2020 by **Tiago Ramalho** (ex-DeepMind, ex-Cogent Labs) and Katsutoshi Yamada. HQ at Shibuya S-6 Building. **~59 staff (FT + outsourced + interns) as of July 2024** per official press release; PitchBook shows 46 FTE. 20+ nationalities, 53% engineers, 38% women per a TokyoDev employee interview. **Funding stage: Seed/strategic** — there is **no Series A** for this Recursive; do not confuse with the unrelated "Ricursive Intelligence" (Palo Alto) or "Recursive Superintelligence" (London/SF). Awarded **OpenAI "Token of Appreciation"** for processing 10B+ tokens. Clients include Suntory (100m-mesh rainfall downscaling), KDDI (ad-creative generation), IHI × Sumitomo Forestry (NeXT FOREST groundwater AI), TRUSTBANK, Fujitsu Learning Media, COACH A. NeurIPS 2025 Climate-Change workshop accepted their **SpLIIF** (Sparse Local Implicit Image Function) sub-km weather downscaling paper.

### Active roles
**Machine Learning Engineer (all levels)** — TokyoDev JD explicitly: *"We also welcome applications from candidates with industry experience in the fields of Machine Learning, AI and Computer Vision."* **Senior Machine Learning Engineer** (~5 yrs), **LLM Engineer** ("Basic Japanese / Japan residents only / Partially remote"), **AI Solutions Specialist** (JLPT N1 required, client-facing), **Senior SWE Fullstack**, **Business Solutions Architect**. No dedicated CV Engineer title.

### Hard skills
Python + PyTorch, generative models including diffusion (the MLE JD specifically lists *"generating images and videos using generative AI"*), HuggingFace, LangChain-style RAG (their **Flow Benchmark Tools** is open-sourced for RAG eval), time-series forecasting, implicit neural representations (SpLIIF), AWS + GCP. NVIDIA Physics-AI tooling partnership (NVIDIA AI Day Tokyo).

### Soft skills
**English is the default working language.** Client-facing communication is central: *"Collaborate closely with our project managers...thoroughly document your work."* Mentorship at senior level. Mission alignment (sustainability/SDGs) explicitly emphasized.

### Day-to-day
Consulting cadence — 3–6 month enterprise engagements: scoping → literature review → modeling proposal → data engineering → training/eval → cloud deployment → handoff. Between client projects, engineers work on internal products (**FindFlow** bilingual enterprise RAG, **Flow Benchmark Tools**, Borealis digital-twin platform, route scheduler). Annual all-hands retreat with global teammates flown in.

### Junior vs mid
The MLE listing explicitly accepts **all experience levels** ("basic understanding of programming and a willingness to learn more"). Senior MLE asks for **~5 years AI R&D**, depth in 1–2 areas, mentorship of small teams.

### Interview process
Glassdoor (5 reports, **avg 33 days end-to-end**): **4 rounds for MLE:** (1) Python coding — three easy/medium LeetCode questions; (2) ML-specific interview covering SGD, optimizers, loss functions, hyperparameters + discussion of a pre-submitted take-home; (3) **30-minute presentation of a recent AI paper of your choice**; (4) culture-fit chat, often with CEO Tiago. All English.

### Compensation
**No salary range published on TokyoDev** for Recursive's Senior MLE — they negotiate individually. Triangulated working estimates: junior ¥6–9M, mid ¥9–13M, senior ¥12–16M+. Equity expected to be modest given Seed-stage cap table. Flex-time, hybrid, generous PTO, relocation supported.

### Application channels
**TokyoDev is the primary external pipeline** with verified company page + employee interview (Xyza Rivera) — they tag Recursive as one of the most English-friendly AI startups in Tokyo. Direct apply via recursiveai.co.jp/en/careers or recursive.breezy.hr. LinkedIn (Tiago is active), Wantedly casual visit, referrals.

### Fit assessment
**Excellent fit, both technically and culturally.** Recursive is one of the most GenAI/RAG-intensive consultancies in Japan (10B+ OpenAI tokens, FindFlow, Flow Benchmark Tools, agentic R&D research agents). Their LLM Engineer JD asks for "*prompt design, model/RAG integration, backend APIs, and cloud performance optimization*." The candidate's **activation-steering work is highly valued** — Recursive's interview literally asks you to present a recent AI paper, and the company is research-flavored (DeepMind heritage). The CV/Android-ExecuTorch background opens optional adjacency to the **SpLIIF satellite/weather downscaling line** and the handwritten-math-grading line. **Junior MLE (¥6–9M) is realistic now;** mid is plausible with demonstrated independent project ownership. The activation-steering portfolio is the differentiating asset here.

---

## 3. Citadel AI (citadel-ai.com)

### Company snapshot
Founded Dec 2020 by **Hironori "Rick" Kobayashi** (CEO) and **Kenny Song** (CTO, ex-Google Brain PM on TensorFlow/AutoML, UTokyo). Mission: AI reliability and trustworthy ML testing. **¥520M (~$3.7M) Series A in July 2023** led by UTokyo IPC with Coral Capital, ANRI, Suntory Holdings, Mitsubishi UFJ Capital. Headcount ~20–40 (Crunchbase: 11–50). Engineers from Google/Brain, Waymo, Paidy, Stripe, Apple, Meta, PayPal, Toyota. Products: **Citadel Lens** (automated ML model testing — tabular, image classification, object detection added Feb 2023, NLP) generating robustness, fairness, explainability, **ISO/IEC TR 24029** and EU AI Act reports; **Citadel Radar** (production monitoring); **LangCheck** (open-source multilingual LLM eval on GitHub). Customers: medical imaging (joint research with **DeepEyeVision/Jichi Medical University** on a 50-disease retinal classifier), autonomous driving, manufacturing visual inspection, **Tokio Marine** (first Japanese financial-industry adoption). Embedded with **BSI** for EU AI Act conformity assessment.

### Active roles
Currently visible: **AI Research Engineer** (Ashby, current open) and **Frontend Engineer**. Historically listed (Kenny keeps them warm): **Computer Vision Engineer**, Backend Engineer, DevOps Engineer, Software Engineer, Founding Software Engineer, Solutions Engineer.

### Hard skills
Python (mandatory), PyTorch + TensorFlow. CV across classification, object detection, segmentation (Lens supports all three). Pytest, Docker, PostgreSQL, Redis/RQ, Flask APIs. GCP-primary (GCE/GCR/GCS/Cloud SQL), AWS/Azure for enterprise. Research literacy in **adversarial ML, XAI, MLSys** explicit in Founding SWE JD: *"if you are already an ML expert, you may read papers in adversarial ML, XAI, and MLSys."* Robustness methodology (ISO/IEC TR 24029, ImageNet-C–style corruptions). Domain bonuses: medical imaging, autonomous driving.

### Soft skills
**English required, Japanese optional** for most engineering roles. *"Slope is more important than y-intercept"* — explicit culture statement valuing learning velocity. Strong SWE/documentation/code-review culture inherited from Google-Brain/Waymo/Stripe DNA. Hybrid 1–2 days/week in office, flextime.

### Day-to-day
Build and extend Lens test suites for image models (classification, detection); implement synthetic-perturbation generators (camera-hardware shifts, lighting, focus, brightness); build the **Bias Finder** for auto-discovery of underperforming data slices; integrate Lens with customer models (medical imaging at DeepEyeVision, manufacturing inspection, AD); read and prototype from arXiv/CVPR/NeurIPS.

### Junior vs mid
JDs are explicit: **CV Engineer ≥1 yr CV SWE** (lowest YoE bar of any Citadel role — junior-to-mid friendly). Backend/SWE ≥2 yrs. DevOps ≥3 yrs. **AI Research Engineer ≥2 yrs GenAI R&D** plus paper-synthesis ability.

### Interview process
No public structured leak. Initial response within 24–48 hrs; emails for SWE/CV/DevOps go directly to **kenny@citadel.co.jp** (founder personally screens). Compensation described as "*a flexible combination of salary and stock options*" — equity is part of the discussion early. Typical small-startup loop (recruiter → coding screen → take-home → on-site with team including Kenny), all in English.

### Compensation
TokyoDev historically displays Citadel CV Engineer range around **¥9M–¥15M**. Synthesized bands: **Junior (0–3 yrs CV) ¥7–11M base + meaningful options; Mid (3–6 yrs) ¥11–16M base + options.** Significant equity stake emphasized for early hires.

### Application channels
TokyoDev (primary — they ask applicants to mention TokyoDev), direct via citadel-ai.com/careers (Airtable form for legacy roles, Ashby for AI Research Engineer), email kenny@citadel.co.jp directly, LinkedIn (Kenny Song is responsive to DMs), Wantedly, FoundX/Coral Capital portfolio events.

### Fit assessment
**Strong fit, particularly for AI Research Engineer.** The candidate's **activation steering is HIGHLY relevant** to Citadel's mission — activation steering is state-of-the-art interpretability/safety work, exactly what AI Research Engineer is meant to operationalize into Lens for VLMs/LLMs. This is the **single strongest differentiator** in the candidate's profile for this company. Python/PyTorch + OCR (similar to detection/recognition under perturbation), on-device CV (perturbation pipelines are mechanically similar to style transfer), and MLOps adjacency all align. Best fit: **AI Research Engineer (open now)** or cold-email Kenny for the **Computer Vision Engineer** title. Recommended approach: apply via TokyoDev *and* email Kenny directly, leading with (a) activation-steering as AI-safety signal, (b) production CV/OCR, (c) MLOps fluency. Reference DeepEyeVision and the BSI/EU-AI-Act work to show domain awareness.

---

## 4. MUJIN (mujin.co.jp)

### Company snapshot
Tokyo industrial-robotics unicorn founded 2011 by **Rosen Diankov** (CMU PhD, OpenRAVE creator) and **Issei Takino**. HQ in Koto-ku Tokyo + offices in Atlanta, Guangzhou, Eindhoven. **MujinController / MujinOS** is a vendor-agnostic, "teachless" robot controller using a real-time non-volatile digital twin. Customers: Fast Retailing (Uniqlo), JD.com (world's first fully automated e-commerce warehouse), Walmart, Paltac, Toyota Group, Askul. **Funding: Series C $85M Sept 2023 (SBI-led) + $18M extension Dec 2023 (Japan Post Capital); Series D first close $233M December 2, 2025 ($133M equity led by NTT Group with QIA co-lead, Mitsubishi HC Capital Realty, Salesforce Ventures; +$100M debt).** Total raised ~$311–341M. Valuation ~¥118.6B (~$800M–$1B). **IPO widely expected but not yet TSE-filed**; originally targeted 2025–2026, likely pushed by the late-2025 Series D. Headcount ~450 globally (PitchBook), engineering ~150–200 with Tokyo as the densest hub.

### Active roles
**Computer Vision Engineer (3D Object Detection & Pose Estimation)**, Senior CV Engineer, Machine Learning Engineer (newer), Robotics Engineer, Motion Planning Engineer, Perception Engineer, **Robot Application Engineer** (customer-facing), Software Development Engineer (Cloud Team), Frontend (React/TypeScript), Hardware/EE.

### Hard skills
**C++ is heavily emphasized — MUJIN is a C++ shop.** The MUJINspire interview-prep blog: *"Generally, computer vision roles will involve using C++ and Python."* Glassdoor Jan 2026: technical assessment had C++ and Python MCQs + 2 medium DSA, with interviewers "*insisting on coding only in C++*." Python for scripting/ML training/prototyping. **3D vision: point clouds (PCL, Open3D), depth sensors (Photoneo, Zivid, Ensenso, RealSense), 6-DoF pose estimation (PPF, ICP variants, learned methods like PoseCNN/DenseFusion/FoundationPose), 3D reconstruction, feature matching, calibration, projective geometry.** Classical CV (OpenCV, stereo geometry). PyTorch is used selectively; their CV team lead's blog frames Mujin's approach as "**beyond deep learning**" — classical/geometric fundamentals are required to be taken seriously. Robotics: manipulation, motion planning (RRT/RRT*, **OpenRAVE** — Diankov's library — and OMPL), kinematics, IKFast. ROS is *not* central — Mujin has its own internal framework descended from OpenRAVE. Linux real-time, multithreading, profiling.

### Soft skills
Bilingual reality: per Japan-Dev, *"Mujin's Japanese requirement varies based on the position, but they don't require Japanese skills for developers... but Mujin still has a lot of employees that can't speak English. So the more Japanese you know, the better."* Customer-facing engineers realistically need conversational Japanese. **8:45 AM standup is sacred** (CTO-enforced). Glassdoor: 2.7/5 overall, 2.2/5 WLB, 2.5/5 comp — recurring themes of long hours (8:30–23:00 cited), 24/7 customer on-call on deployment teams, and a CTO-centric culture.

### Day-to-day
Implement and improve SOTA detection/pose-estimation methods in production C++; build perception pipelines for new SKUs; hand-eye/multi-sensor calibration in factory environments; debug 24/7 customer production failures; real-time performance optimization (<100ms perception cycles); customer commissioning onsite (one Glassdoor reviewer reported being onsite for a year — outlier but indicative).

### Junior vs mid
Published CV Engineer JD specifies **"mid-level or above"** — the headline role is not entry-level. New-grad pipeline exists for Tokyo Tech / U-Tokyo / CMU / Stanford. Mid (3–7 yrs) owns a perception module. Senior (7+ yrs) owns an algorithm family.

### Interview process
Per 42 Glassdoor reports (avg 25 days, **3.29/5 difficulty**): recruiter → online test (C++/Python MCQ + DSA) → hiring manager whiteboard (CV basics, geometry, feature matching) → system/scenario design (real industrial perception problems) → onsite with team and potentially **CTO Rosen Diankov**. Some take-homes reported (one 16-hour example).

### Compensation
Levels.fyi median Tokyo SWE TC **¥10.01M**, range ¥7.65M–¥13.0M. Glassdoor Tokyo "Engineer" base ¥8.0M median. Japan-Dev band ¥6M–¥11M. **Practical bands: new grad ¥5–6M, junior ¥6–8M, mid CV/ML ¥8–13M (JD's CV role realistically ¥9–12M), senior ¥13–18M+.** Pre-IPO stock options with 4-year cliff vesting reported (Glassdoor reviewers complain about non-grants after vesting — verify at offer). Free buffet lunch, on-site gym, visa + flights + housing.

### Application channels
mujin.co.jp/en/careers → **jobs.lever.co/mujininc**, LinkedIn, Japan-Dev (9+ active listings, top-listed), Wantedly (long-form Japanese stories), University recruiting (Tokyo Tech, U-Tokyo, CMU Robotics, Stanford), conferences (iREX, ICRA, Automate). **MUJINspire blog is required pre-interview reading.**

### Fit assessment
**MUJIN is the largest skill gap of all nine companies for this candidate.** Honest mapping:
- ✅ Python — covered. ⚠️ PyTorch is *secondary* at MUJIN; the CV lead frames work as "beyond deep learning" and the CEO has publicly disparaged pure-ML profiles.
- ⚠️ On-device CV (ExecuTorch) modestly transfers but Mujin's "embedded" is industrial real-time C++ on a controller.
- ❌ **C++** — biggest single gap; interviews insist on C++.
- ❌ **3D vision** (PCL, Open3D, ICP, PPF, 6-DoF pose) — Mujin's core; absent in candidate.
- ❌ Robotics fundamentals (manipulation, motion planning, OpenRAVE/OMPL), industrial depth sensors.

**Verdict: Mid-level CV Engineer is unrealistic now.** Needs **12–18+ months of C++ chops, hands-on 3D vision projects (point-cloud segmentation, ICP pose, bin-picking demo with RealSense + arm in Gazebo/Isaac Sim), and ideally OSS contributions to PCL/Open3D/MoveIt/OMPL or a 3D-perception publication.** Even junior is tight without working C++. Adjacent bridge: Cloud Team / Platform SWE roles use multithreading + Python/C++ + distributed systems and could be a foot-in-the-door. Apply when you can confidently whiteboard "explain ICP, why it fails, and how PPF complements it" in C++.

---

## 5. Woven by Toyota (woven.toyota)

### Company snapshot
Toyota's mobility-software subsidiary, headquartered in Nihonbashi Muromachi Mitsui Tower. **TRI-AD (2018) → Woven Planet (2021) → Woven by Toyota (April 2023)**, formally becoming the 18th Toyota Group member in Feb 2024. Headcount **1,000–5,000** (ZoomInfo). Four pillars: **AD/ADAS**, **Arene** (vehicle OS — first shipped on RAV4 2025, expanding across BEVs 2026), **Woven City** (175-acre living lab in Susono, launched Sept 2025), **Cloud & AI**. **2024 layoffs primarily hit US offices**; Woven Union (woven-union.org) formed in response. Toyota's **$3.3B (¥500B) AI/AD infrastructure JV with NTT** (Oct 2024) and **Woven Capital Fund II $800M** (Sept 2025) underwrite the strategic refocus on Arene + AD core. Estimated **100–250 CV/ML engineers** across Perception, ML Platform, Motion Planning ML, Cloud & AI, Vision AI Platform. **English-first** — Japan-with-Silicon-Valley operating culture.

### Active roles
Senior ML Engineer Perception (Tokyo/Palo Alto), Senior ML Engineer Autonomy (3-day hybrid), **Staff ML Engineer – End-to-End Autonomy** (cross-org with TRI: foundation models, world models, VLAs), Senior ML Engineer Human Simulation AI, **ML Platform Engineer** (petabyte data curation), Data Scientist Perception, CV Research Scientist Vision AI Platform (Woven City — video understanding, multimodal LLMs), **Vehicle Perception ML Productionization Engineer**. Internships (12-week, paid, return-offer pipeline).

### Hard skills
**Python + C++** (production AD requires C++; HackerRank C++ challenge used in interviews). PyTorch preferred. **3D perception**: PointPillars, CenterPoint, CenterFormer-class; **BEV representations** (BEVFormer, BEVFusion); multi-view geometry; **NeRF, Gaussian splatting**, SLAM. Sensor fusion (camera + LiDAR + radar), segmentation, lane detection, occupancy. Foundation-model/E2E: world models, LLMs, **VLAs (vision-language-action)** — explicitly on the Staff E2E role; TRI side runs Diffusion Policy and Large Behavior Models. **NVIDIA CUDA, ONNX, TensorRT** for vehicle-ECU deployment (DRIVE Orin/Thor implied). Distributed training (FSDP/DDP), Ray, Kubeflow. Petabyte-scale multimodal data. Nice-to-haves: top-tier publications (CVPR/NeurIPS/ICCV/ECCV), Autoware/ROS2 exposure, ISO 26262 / MISRA awareness.

### Soft skills
English primary working language. "**Giver mindset**" phrase recurs (Toyota Production System import). RFC/design-doc heavy. Glassdoor 2.8/5 (31 reviews), only 42% recommend to a friend, 12% positive business outlook — reflecting layoff/restructuring impact. Mission and people praised; bureaucracy and pay flatness common complaints. **Hybrid 3 days/week.**

### Day-to-day
Train and fine-tune perception models on petabyte multimodal logs, run ablations, evaluate on KPI dashboards, ONNX→TensorRT→DRIVE optimization, push to vehicle test fleet, coordinate with safety/validation engineers and Scale-AI-class vendors. ML Platform roles lean toward distributed systems / data engineering.

### Junior vs mid
Levels.fyi confirms **L3/L4/L5/L6** structure. **L3** (junior/new grad): small components, learn the stack — pure new-grad ML perception IC openings are rare; entry is via the **internship → return offer pipeline**. **L4** (mid): owns model/pipeline end-to-end, ML system design. **L5** Senior: technical lead on a sub-team. **L6 Staff/Principal**: e.g., E2E autonomy lead spanning Woven + TRI.

### Interview process
~45–60 days, 5–7 rounds: recruiter → take-home (CSV/data CLI tool 2–5 hrs, sometimes ML mini-project) → HackerRank coding → technical #1 (ML fundamentals, e.g., diagnose train/val plot, transfer learning, DS&A) → technical #2 (resume deep-dive + CV/3D domain) → **ML system design** (data-curation pipeline at petabyte scale) → cross-functional → hiring manager behavioral → HR/VP final. **All English** via Google Meet. Gaijineer calls it "one of the most humane interview processes"; Glassdoor is more mixed (24–25% positive, slow feedback complaints).

### Compensation
Levels.fyi Greater Tokyo SWE: **L3 median ¥10.1M, L4 ~¥14.4M, L5 median ¥19.1M (range ¥17.2M–¥22.0M)**. ML Engineer L4 median **¥16.95M** (range ¥11.38M–¥22.52M) — ML carries a 10–20% premium over generic SWE. Pay mix is **base + bonus**; stock grant is small (~$664/yr avg in Japan dataset). Total Japan median across roles ~¥14–15M ($95–100K USD). **Top tier for ML in Japan, comparable to Mercari/Rakuten Crimson senior bands; US-based equivalents are 2.5–3× higher** (Staff MLE Palo Alto $161–264.5K base alone). Japanese social insurance, generous PTO, visa + relocation, family planning, in-house training.

### Application channels
woven.toyota/en/careers (Lever ATS), LinkedIn, employee referrals (highly effective — Blind/LinkedIn), niche boards (Creative Tokyo, moaijobs, Japan-Dev), **internship pipeline (only reliable junior path)**, university recruiting (UTokyo/Keio/Tokyo Tech/JAIST + US schools), CVPR/ICCV/NeurIPS direct recruiting. TRI (Toyota Research Institute) is a separate US-only entity — not a Japan path.

### Fit assessment
- ✅ Python/PyTorch baseline matches every JD.
- ✅ **On-device CV experience is genuinely valuable** — maps to the Vehicle Perception Productionization role and TensorRT/ONNX bullets.
- ✅ **Activation-steering on LLMs is strategically relevant** to the emerging end-to-end / VLM-for-AD work (DriveVLM, DriveLM, VLAs) and to the Woven City Vision AI Platform's Video LLM efforts — credible talking point with the Staff E2E team and Woven City team.
- ❌ No 3D / multi-view geometry / BEV / sensor fusion — single largest gap; AD perception roles essentially require it.
- ❌ No production C++ — explicitly required, with a HackerRank C++ challenge.
- ❌ No AD domain knowledge (ISO 26262, automotive sensor stacks, DRIVE Orin/Thor).

**Verdict: NOT a fit for AD Perception IC (junior or mid) right now.** Plausible paths today: **(1) ML Platform Engineer AD/ADAS** (uses PyTorch + deployment + on-device without 3D); **(2) Vision AI Platform / Woven City** (multimodal LLMs and video understanding — activation-steering / LLM background is directly relevant); (3) Data Scientist Perception (analysis-leaning). Mid-level AD perception requires ~18 months of deliberate upskilling: BEV/3D-detection project on nuScenes/Waymo Open Dataset, C++ inference, sensor fusion + camera intrinsics/extrinsics, ideally OSS contribution to Autoware/OpenPCDet/mmdetection3d, plus NVIDIA DRIVE/TensorRT deployment artifacts.

---

## 6. Tier IV (tier4.jp)

### Company snapshot
Japanese deep-tech startup behind **Autoware**, the world's first open-source AD stack. Founded December 2015 by **Dr. Shinpei Kato** (now Specially Appointed Professor at U Tokyo). HQ Kitashinagawa Tokyo + Nagoya + Kashiwa-no-Ha test site. **~330 employees (Jan 2024)**; funding $339M–$448M cumulatively (CB Insights includes debt). Series A "over $100M" led by Sompo 2021; later corporate investments from **Bridgestone, Isuzu (¥6B, 2024), Mitsubishi, Suzuki, Sumitomo Mitsui, MUFG, KDDI**. Products: **Pilot.Auto** (AD stack), **Web.Auto** (cloud DevOps for AD), **Edge.Auto** (sensor+ECU reference HW), **AWSIM** simulator, **ODD-Mapper**, **Co-MLOps Platform**. Customers: Isuzu route-bus pilot (Hiratsuka 2024), **Nippon Steel** heavy-duty intra-plant transporters (Nagoya 2025), Next Mobility (Fukuoka). Founded the **Autoware Foundation** (>100 member orgs). Pivoting toward "**AD 2.0 — end-to-end neural models**" with a Level-4+ E2E architecture announced July 2025 and research collaboration with **Prof. Matsuo's U Tokyo lab**.

### English-friendliness
**Moderate.** International engineers on senior teams (Maxime Clement/France, Simon Thompson/AU, Jacob Lambert/Canada, Kok Tan/Malaysia). Many Herp JDs are Japanese-only; TIER IV Talks meetups typically Japanese. **Less English-default than Woven**; business Japanese is advantageous for Web.Auto and customer-facing teams.

### Active roles
careers.tier4.jp (Herp) lists ~21 open Autoware & AI roles. CV/ML-relevant: **Perception Engineer (1007)** — detection/segmentation/tracking/prediction in ROS 2/C++; **Perception ML Engineer (1121)**; **Edge AI Engineer (1203)** — DNN deployment on GPU/FPGA/AI accelerators; **ML Senior Engineer End-to-End AD (1022)**; **Co-MLOps AI/MLOps Engineer (1207)** — Active Learning, Auto-labeling, Domain Adaptation, generative-AI data; **Sensing Engineer (1006)**; Planning & Control; **Data Scientist (1024)**; **Data Operation Lead (1118)**; Software Performance Engineer; Software Release Engineer (Autoware CI/CD).

### Hard skills
**C++17/20 and ROS 2 (Humble→Jazzy)** non-negotiable for perception. Python for training/eval/MLOps/auto-labeling. **PyTorch + mmdetection3d** (official Autoware CenterPoint training recipe runs on mmdetection3d / PyTorch 1.13 / CUDA 11.6). **ONNX → TensorRT** deployment; Tier IV's blog covers their custom C++ TensorRT/ONNX-Runtime inference framework for **SAM2 auto-labeling**. 3D detection: **CenterPoint, PointPillars** (deployed in `autoware_lidar_centerpoint`), TransFusion, BEV emerging. 2D: **YOLOX** in production. Segmentation: **SAM2** for auto-labeling. Camera–LiDAR–radar fusion (RoiClusterFusion, RoiPointcloudFusion, RoiDetectedObjectFusion). MOT (Kalman + association). LiDAR drivers via the **Nebula** project (Hesai, Velodyne, RoboSense, Continental, Seyond). Calibration, OpenCV. SLAM: **NDT matching** is canonical Autoware. NVIDIA **Jetson AGX Orin/Xavier** (edge-auto-jetson). AWSIM (Unity), scenario_simulator_v2. Web.Auto: AWS (Redshift, Glue, Kinesis, EMR, ECS/EKS, Fargate), gRPC/Protobuf, DDD microservices.

### Soft skills
**Develop-in-public OSS mindset.** Perception/Planning/Quality JDs literally list "Contribute autonomous driving technology to OSS" as a duty. Software Quality JD: *"You can showcase your output to the world through contributions to a large OSS codebase."* Heavy documentation (docs on docswell.com/s/TIER_IV, autoware-documentation.github.io, medium.com/tier-iv-tech-blog). Global collaboration with Autoware Foundation members (LeoDrive Turkey, ITRI Taiwan, Robotec.ai Poland, AutoCore China). Mixed English/Japanese; Web.Auto JP-heavy, Perception R&D more bilingual. Academic ties — many PhD hires.

### Day-to-day
Author and merge PRs to `autowarefoundation/autoware_universe`, `tier4/AWML`, `tier4/autoware_perception_evaluation`, `tier4/nebula`, `tier4/tier4_perception_dataset`. Survey latest perception papers (BEV, sensor fusion, occupancy, world models) and reproduce them. Train CenterPoint/TransFusion/SAM2 on internal + nuScenes + Argoverse-2 + Co-MLOps data. Export to ONNX, optimize TensorRT for Jetson Orin; INT8/FP16 quantization. Run perception_eval benchmarks via driving_log_replayer_v2 in CI. Field debugging on Isuzu/Nippon Steel deployments. Triage external Autoware GitHub issues. Present at TIER IV Talks (Japanese) and Autoware Foundation TSC (English).

### Junior vs mid
**Junior / new grad (0–3 yrs):** Onboarded via "**実車研修**" (real-vehicle training) at Kashiwa-no-Ha; starts by fixing bugs in autoware_universe under an architect's mentorship. **Mid (3–6 yrs):** Owns a module (e.g., LiDAR-camera fusion, traffic-light recognition). **Senior/Architect (6+ yrs):** Designs roadmap, leads OSS community for their domain, interfaces with customer OEMs.

### Interview process
Recruiter / casual screen → document screening (**GitHub or portfolio strongly weighted; JDs explicitly request portfolio links**) → 1–2 technical rounds (ROS, C++, 3D geometry, past CV/ML projects) → coding take-home or system design (build a tracker; build a small ROS node) → onsite + potential ride-along at Kashiwa-no-Ha. Language depends on team.

### Compensation
Tier IV **does not publish on TokyoDev or Japan Dev**; recruits via own Herp portal. Glassdoor data sparse and paywalled. Triangulated estimates: **new grad/junior ¥6–9M, mid ¥9–14M, senior ¥14–20M, staff/principal ¥20M+**. **Pre-IPO stock options** part of senior packages; the corporate-investor cap table (Bridgestone, Isuzu, Mitsubishi, Suzuki, SMBC, MUFG) suggests a TSE Growth IPO trajectory. Position: **lower than Woven or Turing; higher than traditional OEM/Tier-1s; comparable to Preferred Networks / Rapyuta.**

### Application channels
**careers.tier4.jp → herp.careers/v1/tier4** (50+ live reqs; CV + GitHub/LinkedIn/portfolio requested). Single most distinctive feature of Tier IV: **GitHub contributions to autowarefoundation/autoware or tier4/* are the biggest differentiator** — architects literally search GitHub for active contributors. LinkedIn (Satoshi Tanaka, Simon Thompson post hiring callouts), TIER IV Talks on connpass, tech.tier4.jp, Wantedly, university channels (Kato lab + Matsuo lab).

### Fit assessment
**Critical gap analysis:** Transferable strengths are Python, PyTorch, on-device deployment (the latter maps cleanly to the Edge AI / Perception ML JDs that talk about ONNX → TensorRT on Orin/Xavier). **Critical gaps for Perception roles:** no ROS 2/C++17 production, no 3D vision/LiDAR, no multi-sensor calibration/fusion, limited probabilistic state estimation. **Verdict: NOT a fit for Perception Junior immediately** — most hires are robotics/CV PhDs or experienced C++ robotics engineers.

**Realistic alternative tracks:** Co-MLOps AI/MLOps Engineer (1207 — Python, MLOps, auto-labeling, Active Learning, generative-AI data; less ROS-dependent), Data Scientist AD (1024 — Python-centric driving-log analysis), Edge AI Engineer (1203 — DNN deployment optimization is genuinely transferable from on-device CV), Backend/Senior Data Engineer for Web.Auto (1107/1195 — closer to general SWE).

**Ramp-up to credibility (~6–9 months):** Install Autoware + AWSIM end-to-end → grab "good-first-issue" PRs on autoware_universe (3–5 merged) → improve a perception module like adding INT8 calibration to `autoware_lidar_centerpoint` → reimplement a recent 3D detection paper (BEVFusion or StreamPETR) on a Jetson Orin Nano with a blog writeup → attend TIER IV Talks. **Tier IV is unusually friendly to this strategy because their codebase IS their hiring funnel** — your contributions are evaluated in the same review system you'd use as an employee. This is unique among the nine companies.

---

## 7. PayPay (paypay.ne.jp)

### Company snapshot
Japan's dominant QR-payment platform — **~70M registered users as of July 2025** (one in two Japanese smartphone users). JV of SoftBank + LY Corporation; Paytm sold its remaining stake to SoftBank for ~$279M in late 2024. **Nasdaq IPO delayed twice** (US shutdown late 2025; Iran shocks March 2026) but still actively in motion — RSUs in PayPay shares carry real near-term liquidity potential. Engineering: members from **50+ countries, ~50% non-Japanese engineers, ~80% of product team discussions in English** per TokyoDev. **ML/CV use cases:** fraud detection at >1,000 TPS, **eKYC** (OCR for My Number Card, driver's license, residence card — strict Japanese 犯収法 requirements), face matching + liveness, **credit scoring** for PayPay Card and BNPL, recommendations (PayPay Mall, **PayPay Securities ML stock recommendation** launched July 8, 2025), AML/risk, NLP chat automation, internal LLM copilots.

### Active roles
Machine Learning Engineer (Data Insights, ¥9–12M), **Senior ML Engineer** (¥9–13M), **Senior Data Science Engineer** (¥9–13M), **ML Engineer Credit Modeling** (3+ yrs FinTech, remote-Japan), Data Science Engineer (¥9–12M), Data Analyst with ML/GenAI focus, MLOps/ML Platform roles (¥7.5–15M+). No dedicated CV title — OCR/document AI and image-based fraud signals live inside the ML/Fraud/eKYC teams.

### Hard skills
Python + SQL (mandatory), Java/Scala for platform-integration. **PyTorch, TensorFlow, scikit-learn, XGBoost, Keras** (all five listed in Credit Modeling JD). Apache Spark, Hadoop, Jupyter, Git, Jenkins. MySQL/PostgreSQL. **AWS-centric**: Glue, **SageMaker**, Athena, S3, EMR, Redshift, Kinesis. Docker/Kubernetes (EKS). BigQuery, Spark, Kafka/Kinesis streaming. MLOps practices (drift monitoring, automated retraining, model registries). **CV-relevant (inferred):** OCR engines (Tesseract, PaddleOCR, AWS Textract), document layout models, face recognition/verification, liveness, anomaly detection. LLM/GenAI tagged on newer ML postings. Strong stats foundation for fraud/credit.

### Soft skills
English working language confirmed; **Japanese not required and "will not hold you back."** PayPay "5 Senses" values: Unparalleled speed, Commitment, Logical thinking, Professionalism, Day-1 mindset. Scale-mindset; cross-functional with risk/compliance/PM.

### Day-to-day
Execute the full ML lifecycle (EDA → feature engineering → training → eval → serving) on AWS; build feature pipelines (Spark/Glue); deploy models to systems serving 65M+ DAU; monitor drift; **A/B and hypothesis testing is a named responsibility**; on-call rotation for credit/fraud. **Super Flex Time, no core hours, WFA allowance ¥100,000.**

### Junior vs mid
JDs specify **3+ years** for ML/DS Engineer; **5–7+** for Senior. Levels.fyi uses **P2/P3** SWE bands: P2 ¥8.71M, P3 ¥12.24M median, max reported ¥14.28M. PayPay Card SWE median ¥10.78M. Junior is **not a strong target** at PayPay — most JDs require 3+ years; new-grad pipeline mostly flows to SWE/DA. **Mid (3–7 yrs) is the sweet spot.**

### Interview process
Per PayPay's own description, Glassdoor (113 interviews), and InterviewBit: application review (English CV) → coding test or take-home (HackerRank ~90 min, DS&A — binary trees, two-pointer) → **3–4 interview rounds** (ML deep-dive on past projects, ML system design, behavioral on 5 Senses, hiring manager) → offer. **All English.** Average ~30 days; difficulty 2.98/5. Glassdoor candidate experience 23.9% positive — complaints about long wait times after final round.

### Compensation
Posted TokyoDev ranges: ML Engineer ¥9–12M, Senior ML/DSE ¥9–13M. Levels.fyi May 2026: SWE median ¥8.88M, P2 ¥8.71M, P3 ¥12.24M, max reported ¥14.28M. **Near the top of Japanese fintech compensation; below FAANG Tokyo (Google Tokyo SWE median ~¥25M).** Total comp components: base + annual special incentive (company + individual perf) + **RSUs in pre-IPO PayPay shares (meaningful upside if 2026 Nasdaq listing completes)**, WFA allowance ¥100K, late OT allowance, visa sponsorship, **free Japanese lessons during working hours**, 14 days annual leave + 5 personal.

### Application channels
**TokyoDev is the highest-signal channel** — PayPay posts include explicit salary ranges, transparent language requirements, and most ML roles tagged "No Japanese required" and "Apply from abroad" with remote-in-Japan policy. Then about.paypay.ne.jp/career/en/, LinkedIn (recruiter Haruka Murasaki), Japan Dev (9 roles), Relocate.me, Wantedly, referrals.

### Fit assessment
**PayPay is one of the strongest realistic targets for this candidate.** Three reasons:

**(a) Azure Doc Intelligence OCR → PayPay eKYC is a near-perfect signal.** Every onboarding wallet user, PayPay Card applicant, and PayPay Securities/Banking customer goes through eKYC. Document AI experience is directly transferable — even though PayPay uses AWS Textract/SageMaker not Azure, the methodology (CER, exact-match metrics, document-layout reasoning, pipeline architecture) is highly portable. This is **the single most differentiating signal the candidate brings.**

**(b) LangGraph/RAG aligns with PayPay's GenAI direction** — postings explicitly tag "Generative AI"; internal LLM copilot, chat-support automation, and merchant GenAI products are real workstreams.

**(c) Activation steering** is less directly relevant but signals depth in modern ML internals — useful for senior conversations.

**Recommended targeting:** apply now to **Machine Learning Engineer (Data Insights)** and **Data Science Engineer** roles (¥9–12M) via TokyoDev. **Junior fit: STRONG** despite the nominal 3+ year floor — candidates with production OCR + LLM/RAG portfolios and a strong coding screen typically clear it. **Mid fit: 12–18 months out** after building production deployment + A/B at scale + SageMaker/EMR fluency. Realistic offer band today **¥8–11M total**. Lead the CV with the OCR/eKYC framing; bridge Azure → AWS proactively (mention Textract/SageMaker self-study).

---

## 8. Ubie (ubie.life / ubiehealth.com)

### Company snapshot
Tokyo healthtech founded May 2017 by Dr. Yoshinori Abe (CEO/MD, U-Tokyo) and Kota Kubo (Co-CEO, software engineer). **Symptom Checker B2C: 13M+ MAU globally (12M+ Japan, 2M+ US), growing 25% MoM in US.** Trained on 50,000+ peer-reviewed publications + a 50+ specialist feedback panel. **Ubie Medical Navi (B2B):** AI tablet pre-consultation deployed at **1,700–1,800+ Japanese hospitals/clinics**. Newer products: Doctor's Note, Checkup, **Ubie Consult** (April 2026 US launch LLM chat with photo upload, OTC advice, triage). Funding: ~$115M over 17 rounds (Crunchbase); Series C extension $19M 2022; Mayo Clinic Platform Accelerator (June 2025). **Frequently called a unicorn but no clean priced >$1B round visible on Crunchbase.** Headcount (Mar 2025): Tokyo ~240, APAC HQ Singapore ~10, US HQ ~4 — total **~250** (smaller than the task brief's "~400"). **Holacracy since 2019**, OKR-driven, completely flat. **CV presence is very limited** — heavy NLP/LLM and Japanese clinical NLP; Consult accepts photo uploads but this is LLM-multimodal rather than DICOM/radiology/segmentation.

### Active roles
**シニアソフトウェアエンジニア（機械学習システム）**, **機械学習エンジニア**, データエンジニア / アナリティクスエンジニア, **LLMアプリケーションエンジニア**, **コーポレートAIエンジニア** (internal productivity AI), general SWE Web. **No dedicated CV role currently posted.**

### Hard skills
Python (ML), TypeScript/Next.js, Go, Kotlin, Ruby (legacy), Java, some Rust. LangChain, **LangGraph**, internal "ChatGPT-like" tools wrapping **Gemini/GPT/Claude**, RAG, prompt engineering, evaluation harnesses, **AlloyDB for PostgreSQL** (vector workloads). PyTorch/Transformers in older JDs. Japanese tokenization (MeCab/Sudachi) implicit. **GCP-centric** (BigQuery, Vertex AI, AlloyDB, Gemini API). Kubernetes, Docker, dbt. Medical/clinical: ICD groupings, clinical vignette evaluation, RLHF-style physician feedback loops. Their published medRxiv work uses LLM-assisted medical entity linking. **CV-specific: essentially none publicly advertised.**

### Soft skills
**Six "Ubieness" cultural pillars:** Adaptable, Impact-Oriented, Autonomous/Ownership-Minded, Humble, Endlessly Curious, Collaborative/Candid/Constructive. Plus the Japanese rubric: 突破力, 全社への当事者意識, 論理性, ラーニングアニマル, 率直なコミュニケーション. **Holacratic, consent-based decision-making** — no managers; feedback flows via Slack bots. Strong product-engineering-physician collaboration. **Bilingual but Japanese-leaning** — DeepL-in-Slack is the accommodation, not English-first meetings. Global Team page currently says *"We're all filled up on full-time roles at the moment"* — the **English-track ML door is essentially closed in May 2026**.

### Day-to-day
Train/eval medical classifiers and question-selection models; build RAG pipelines over 50,000+ medical-article corpus + physician feedback; prototype LLM features (summarization, Consult chat, discharge summary, EHR aids) and ship into Medical Navi; evaluate clinical safety via vignette simulations (their medRxiv paper reports Top-5/Top-10 of 63.4%/71.6% on 328 vignettes); A/B test alongside MDs labeling outputs; internal-productivity AI work (Cursor rollouts, internal ChatGPT clone). Less research-y; more **product-shipping** with 600+ releases/year claim.

### Junior vs mid
Most open JDs are **mid-skewed** — Senior SWE ML Systems and LLM App Engineer require tech-lead or EM experience. They publish Forbes Under 30 ML engineers among recent hires; many came from Mercari, Sansan, BrainPad, Eureka, Bay Area FAANG-adjacent.

### Interview process
~2 weeks–1 month, fully online via Google Meet. Casual chat (15-min free entry from website) → multiple technical/structured interviews → **cultural Ubieness deep-dive** → final with co-CEO or senior leadership. *"Most of the topics are about your Ubieness"* — culture-fit weight is unusually high. Mix English/Japanese (Japanese dominant for Japan-track ML).

### Compensation
OpenMoney (n=13): avg ¥11.05M, median **¥8.47M**, range ¥5.20–¥20.50M. Synthesized ML bands: junior ¥6–9M, mid ¥9–13M, senior/staff ¥13–18M+. **Stock options for all employees** — base salary lower than Mercari/PayPay/SmartNews at equivalent levels; **equity is the differentiator**. Compensation is **company-wide OKR-linked**, not individual-perf evaluated (explicit Holacracy choice). 45 hours/month fixed OT included in base; reported actual avg ~10 hrs/month.

### Application channels
**Wantedly is primary** (heavy casual visit / カジュアル面談 funnel), recruit.ubie.life / recruit-data.ubie.life, HERP Careers, LinkedIn (mostly Global/US), recruit.ubiehealth.com (currently closed), referrals via public Slack workspace. Ubie does **not** currently post on TokyoDev's vetted board.

### Fit assessment
**Strong technical fit, significant language barrier.** RAG + LangGraph + Gemini stack is **a direct match** to Ubie's internal stack — the LLM Application Engineer and Corporate AI Engineer roles read almost like a checklist for the candidate. OCR experience maps to the Gemini-multimodal discharge summary pipeline and Consult's photo-upload. **Activation steering / interpretability is genuinely interesting for medical safety** (hallucination prevention, refusal calibration) — would resonate strongly with **Ubie Lab** (internal LLM-safety research group founded July 2023).

**Weak fits:** style-transfer CV doesn't map to anything Ubie does (no radiology/pathology/imaging line). Don't lead with CV — reframe as multimodal Gemini/photo-input work. **Japanese medical NLP roles require business-level Japanese** to work with physicians and Japanese clinical text. **Global Team is currently closed.**

Best fit: **LLM application engineer on Consult (US-facing, most English work)** or **Corporate AI Engineer (internal tools, less medical-text exposure)**. **Junior level is realistic** if Japanese is at least conversational. **Mid is 18+ months out** — needs business-level Japanese, demonstrated tech-lead/EM experience, and (per Ubie norm) a 入社エントリ-style blog presence on Zenn/note. No classical CV career path here.

---

## 9. Algomatic (algomatic.jp)

### Company snapshot
Tokyo GenAI-native startup studio founded April 13, 2023 by **Shunsuke Otsuka (大野峻典)** — U Tokyo deep learning lab, ex-Indeed, founder of Algoage (M&A'd into DMM Group 2020). CTO **Yuki Nanri (南里勇気)** joined June 2023 via Bison Holdings M&A. **Funding: single ~¥2B (~$13–14M) strategic investment from DMM.com at founding** — not a traditional VC round; Algomatic is part of the DMM group. No formal Series A publicly announced. **Headcount ~43–50 FT** as of late 2025 (smaller than the task brief's 50–80 estimate). Org is a "startup studio" with multiple sub-companies, each with independent P&L: **NEO(x)** (BtoB GenAI/AI agents), **Algomatic Works** (HR — Recruita AI), **AX** (enterprise GenAI consulting), **LLM STUDIO**, **にじボイス Nijivoice** (emotion-aware TTS), **シゴラクAI** (secure ChatGPT for enterprise), **アポドリ Apo-dori** (sales-call AI agent, ARR ~¥110M in 6 months). **Not a real-estate-AI company** (the task brief was slightly off on this) — multimodal exposure is HR document understanding, CAD/manufacturing support, and voice.

### Active roles
**機械学習・生成AIエンジニア (ML / GenAI Engineer)**, **基盤モデルアプリケーションエンジニア (Foundation-Model Application Engineer)**, Tech Lead (GenAI for BtoB sales), Web/Full-stack Engineer (Algomatic Works), NEO(x) Engineer, **生成AIプロダクト開発インターン**, prompt-engineering roles. **No dedicated CV role.**

### Hard skills
ML dev experience 3+ years (for ML/GenAI Engineer). Python mandatory + PyTorch/TensorFlow. **論文再現実装** (ability to reimplement papers — high bar for the ML/GenAI role specifically). CS MS/PhD or 2+ years equivalent. Backend Python (FastAPI) + some Go; Frontend TypeScript/React/Next.js/Hono; Cloud AWS (ECS, Aurora) + GCP (BigQuery, Looker Studio) mixed; Docker, K8s. LLM stack: OpenAI (GPT-3.5/4/4o), Anthropic, **Google Gemini** (heavily used as both product API and dev tool), **LangChain, LlamaIndex, LangGraph, LangSmith**. RAG: pgvector mentioned. Heavy **agentic patterns**: Router architecture, Tool Calling, multi-step agents (NEO(x) engineer @catshun_ has written extensively). Multimodal: screenshots + text RAG, TTS. Fine-tuning + prompt engineering + RAG explicitly listed for Foundation-Model App Engineer.

### Soft skills
Team-output maximization, ambiguity tolerance, learning-from-failure, insatiable curiosity. **Japanese is a real and significant factor.** Every job page is Japanese-only. No English career page, no English version of jobs.algomatic.jp, no English-language podcast/blog. Customer base is Japanese enterprise. **Business-level Japanese (JLPT N2+) is effectively required in practice.**

### Day-to-day
Build LLM-powered features in 0→1 product launches across parallel sub-companies; prompt engineering + eval design (LangSmith); RAG construction; build agents with LangGraph (Router, Tool-Calling, Agentic RAG); frontend integration; cost/latency management of API calls; occasional fine-tuning; for ML/GenAI Engineer specifically — paper reading + reimplementation, custom algorithm dev. Engineers present at meetups (LangChain Meetup Tokyo #3, Oct 2024).

### Junior vs mid
**Junior (0–2 yrs):** Likely intern or "Web Engineer" full-stack-with-AI. **Mid (2–5 yrs):** Owns a sub-product's ML/AI stack within one of the sub-companies — closer to founding engineer of a sub-line than typical mid-IC. **Tech Lead:** Owns business-line tech direction; usually paired with a sub-company CEO.

### Interview process
カジュアル面談 → multiple interview rounds (3–4 implied) → reference checks → offer. **Conducted in Japanese.** Process is reportedly fast. For prompt-engineering roles, Algomatic pioneered a "**prompt question**" written exam format.

### Compensation
**Not published** ("年俸制、経験・スキルを考慮の上、当社規定により優遇"). Inferred ranges (Findy/Green comparables): **junior ~¥5–7M, mid ¥8–12M, senior/Tech Lead ¥12–18M** (裁量労働制 noted for Tech Lead). Stock options likely but unusual structure given DMM-group ownership. **Below Mercari/PayPay/Sakana AI levels.**

### Application channels
**Wantedly is primary** (casual visit funnel built in), jobs.algomatic.jp (Notion-hosted via Wraptas), **Findy** (engineer-specific listings, more detail), HERP Careers, Green, direct email hr@algomatic.jp, engineer referral via Tech Blog/Zenn/X (@catshun_, @neonankiti). **Not on TokyoDev** — aligns with Japan-first positioning.

### Fit assessment
**EXCEPTIONAL technical fit, SIGNIFICANT language barrier.**

- ✅ LangGraph: explicitly used in Algomatic Tech Blog production (Router pattern, Agentic RAG) — direct match.
- ✅ RAG: core competency across multiple sub-companies.
- ✅ Gemini: heavily used internally.
- ✅ LoRA/fine-tuning: explicitly listed as desired for Foundation-Model App Engineer.
- ✅ Activation steering / interp: aligns with LLM STUDIO sub-org and 論文再現実装 requirement; CEO has academic ML background.
- ❌ **All hiring funnel is Japanese.** No English application path, no precedent of English-only hires. All customers Japanese enterprise.

**Verdict:** Highest-ceiling technical fit but lowest-floor language fit. **If candidate has N2+:** apply via Wantedly casual visit immediately — strong probability of mutual fit. **If candidate is N5/sub-N3:** realistic only with truly exceptional portfolio plus willingness to commit to learning Japanese aggressively post-hire; best framing would be a direct cold reach to CTO @neonankiti, bypassing the standard Japanese funnel.

---

## Cross-company comparison

| Company | English-friendly | CV depth required | Junior fit | Mid fit (12–18mo) | Comp range (mid, JPY) | Primary channel |
|---|---|---|---|---|---|---|
| **Mercari** | ★★★★★ (~40% non-JP, English meetings) | Medium — CLIP fine-tuning, on-device, multimodal | **Strong** (Edge AI, AI/LLM, Item Understanding) | Strong (Search, Rec, Platform) | MG2–MG3 ~¥9–13M | careers.mercari.com / TokyoDev |
| **Recursive** | ★★★★★ (English default, 20+ nationalities) | Low–medium — opportunistic (sat imagery, generative) | **Strong** (MLE all levels) | Strong (Senior MLE) | ~¥9–13M (not published) | TokyoDev (primary) |
| **Citadel AI** | ★★★★★ (English internal) | Medium — robustness testing, detection/seg/cls | **Strong** (CV Eng 1yr min; AI Research) | Strong | ¥11–16M base + options | TokyoDev + kenny@citadel.co.jp |
| **MUJIN** | ★★★ (bilingual; JP helps) | **Very high** — C++, 3D, 6-DoF pose | Tight (C++ required) | Stretch — 12–18mo upskill | ¥8–13M | jobs.lever.co/mujininc / Japan-Dev |
| **Woven by Toyota** | ★★★★★ (English-first) | **Very high** — BEV, 3D, sensor fusion, C++ | Internship pipeline only | Stretch for AD perception; Platform/Vision AI fits sooner | L4 ML ~¥14–17M | woven.toyota careers / Lever |
| **Tier IV** | ★★★ (more JP than Woven) | **Very high** — ROS 2, C++, LiDAR | Tight without ROS/C++ | Stretch for Perception; viable for Co-MLOps/Data Sci/Edge AI | ¥9–14M (est., unpublished) | careers.tier4.jp + GitHub PRs |
| **PayPay** | ★★★★★ (~80% English in product) | Low — OCR/eKYC, fraud, recsys | **Strong** (OCR/eKYC fit is rare) | Strong | ¥9–13M + pre-IPO RSU | TokyoDev (primary) |
| **Ubie** | ★★ (Japanese-leaning; Global Team closed) | Very low — NLP/LLM focused | Possible if N2+; tight if not | 18+ months out | ¥9–13M + SO | Wantedly (primary) |
| **Algomatic** | ★ (Japanese-only funnel) | Low — LLM/agent-centric, multimodal opportunistic | Tight without N2 | Tight without N2 | ¥8–12M (est.) | Wantedly (primary) |

---

## Where to apply first: prioritization for the candidate

Given the candidate's stack (Python/PyTorch, LoRA/QLoRA, Gemini, **Azure Document Intelligence + OCR**, RAG with Chroma/Qdrant, **LangGraph**, **on-device CV via ExecuTorch Android style transfer**, **LLM activation-steering experiments**), English-speaking, learning Japanese, with SWE background:

**Tier 1 — Apply immediately (next 0–4 weeks).** These four offer the strongest signal match for the candidate's existing portfolio with English-friendly cultures and clear application paths.

1. **PayPay — Machine Learning Engineer (Data Insights) and Data Science Engineer via TokyoDev.** The OCR / Azure Document Intelligence experience is the **single most differentiating signal** for PayPay's eKYC team. English-first, remote-in-Japan, transparent ¥9–12M bands. Lead the CV with the OCR/eKYC framing; bridge Azure → AWS proactively. Realistic offer today: ¥8–11M.

2. **Mercari — Edge AI / Image Search ML Engineer + AI/LLM Engineer (MG1–MG2) via careers.mercari.com.** The ExecuTorch Android on-device CV experience is genuinely rare and maps directly to TFLite/MediaPipe inference. Pair with an AI/LLM application (LangGraph/RAG/Gemini fit). Pursue a referral via publicly visible engineers like Andre Rusli. Realistic offer: ¥7–10M MG1–MG2.

3. **Citadel AI — AI Research Engineer (current) and cold-email Kenny for the Computer Vision Engineer line.** **Activation steering is the perfect signal** — it sits squarely in Citadel's interpretability/safety R&D agenda. Apply via TokyoDev *and* email kenny@citadel.co.jp directly, referencing the DeepEyeVision/BSI/EU-AI-Act work. Realistic offer: ¥9–14M base + meaningful options.

4. **Recursive — Machine Learning Engineer (all levels) or LLM Engineer via TokyoDev.** GenAI-research-flavored consulting with the strongest cultural fit for an English-speaking candidate with research curiosity. The interview literally requires presenting a recent AI paper — activation steering is a natural pick. Realistic offer: ¥6–10M junior, ¥9–13M mid.

**Tier 2 — Conditional / strategic. Apply if conditions match.**

5. **Ubie — LLM Application Engineer (Consult, US-facing).** Apply *only* if Japanese is at least conversational, or if you can land the rare English-track Consult role. Strong technical fit but Global Team currently closed.

6. **Algomatic — Foundation-Model Application Engineer.** Apply *only* if JLPT is N2+. Highest-ceiling technical fit (LangGraph/Gemini/LoRA/research curiosity) but Japanese is a near-disqualifier otherwise. If interested, cold-reach CTO @neonankiti directly.

**Tier 3 — Defer 12–18 months while upskilling.** These four have skill gaps too large for direct application now.

7. **Woven by Toyota — Vision AI Platform / ML Platform (achievable in 6–9 months); AD Perception 18 months out.** Activation-steering on LLMs is strategically relevant to VLM-for-AD work. Bridge via ML Platform or Woven City Vision AI before targeting AD Perception. Requires BEV/3D-detection portfolio + production C++.

8. **Tier IV — Co-MLOps Engineer or Edge AI Engineer (6–9 months); Perception Engineer 12+ months out.** Tier IV's codebase IS their hiring funnel — start contributing merged PRs to autowarefoundation/autoware now. ONNX/TensorRT on Jetson Orin is a directly transferable bridge from on-device CV.

9. **MUJIN — Cloud Team / Platform SWE as bridge (6–9 months); CV Engineer 12–18+ months out.** Largest gap of all nine — requires production C++, point clouds, ICP/PPF, robotics fundamentals. Apply only after a public bin-picking portfolio + serious C++ chops. Glassdoor culture flags (2.7/5 overall, long hours, in-office) are also worth weighing.

**Recommended sequencing:**
- **Weeks 0–2:** Apply to PayPay, Mercari (Edge AI + AI/LLM), Citadel AI, Recursive in parallel.
- **Weeks 0–8 in parallel:** Reproduce the Mercari "80% More Budget-Friendly CLIP" Vertex AI Vector Search blog post; build a small Go service fronting a PyTorch model; ship an internal RAG demo on AWS SageMaker (PayPay bridge) and on Vertex AI (Mercari bridge); make 3–5 merged PRs to autowarefoundation/autoware (Tier IV bridge).
- **Months 3–6:** Begin Japanese to N3, plus a Jetson Orin TensorRT INT8 deployment side-project (Tier IV/Woven bridge).
- **Months 6–12:** A public BEV/3D-detection project on nuScenes (Woven/Tier IV bridge); serious C++17 work (MUJIN bridge).
- **Months 12–18:** Apply to Woven Platform, Tier IV Edge AI / Co-MLOps, and reconsider Ubie/Algomatic with stronger Japanese.

The single highest-conviction action is **applying to PayPay (Data Insights ML / Data Science Engineer) on TokyoDev this week** — it is the rare case where the candidate's existing Azure Document Intelligence / OCR experience plus LLM/RAG portfolio is genuinely differentiating against the typical applicant pool, and the offer band and remote-in-Japan policy directly match the candidate's situation.