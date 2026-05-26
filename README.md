# RAG System for Cybersecurity Practitioner Guidelines

> An open-source Retrieval-Augmented Generation (RAG) pipeline that retrieves phishing defence knowledge from authoritative standards (NIST, CISA, APWG) and generates structured, actionable guidelines for mid-sized enterprise security practitioners using Llama 3.3 70B.

---

## Table of Contents

- [Overview](#overview)
- [Research Questions](#research-questions)
- [Results](#results)
- [System Architecture](#system-architecture)
- [Knowledge Base](#knowledge-base)
- [Repository Structure](#repository-structure)
- [Reproduction Instructions](#reproduction-instructions)
- [Dependencies](#dependencies)
- [Troubleshooting](#troubleshooting)
- [Ethical Considerations](#ethical-considerations)
- [References](#references)

---

## Overview

Phishing is the most prevalent cyber attack vector facing mid-sized enterprises. Security practitioners rely on guidance from NIST Special Publications, CISA advisories, and APWG threat intelligence reports — but these documents are voluminous, technically dense, and periodically superseded. This project builds a RAG system to bridge that gap.

The system:
1. **Retrieves** the top-3 most semantically relevant chunks from a 2,775-chunk knowledge base
2. **Generates** exactly 3 structured, actionable guidelines per practitioner query
3. **Evaluates** output quality using the RAGAS framework and 3 additional custom metrics

All components are open-source and run on Google Colab free tier with a free Groq API key — no paid services required.

---

## Research Questions

This project addresses the following research questions:

| # | Research Question |
|---|---|
| RQ1 | Can an open-source RAG system (Llama 3.3 70B) generate actionable phishing defence guidelines grounded in authoritative standards (NIST, CISA)? |
| RQ2 | How do RAGAS scores vary across different practitioner query types — technical, procedural, and regulatory? |
| RQ3 | What are the primary failure modes of reference-free RAG evaluation on a cybersecurity-domain corpus? |
| RQ4 | What ethical risks arise from deploying a US-standards-centric RAG system to practitioners in diverse regulatory contexts? |

---

## Results

### RAGAS Evaluation (10 queries, all targets met)

| Metric | Mean Score | Target | Result |
|---|---|---|---|
| Context Precision | **0.917** | > 0.6 | Pass |
| Faithfulness | **0.862** | > 0.7 | Pass |
| Answer Relevancy | **0.673** | > 0.6 | Pass |

### Additional Metrics

| Metric | Mean Score | Description |
|---|---|---|
| Source Diversity | 0.60 | Proportion of unique sources among top-3 retrieved chunks |
| Citation Precision | 0.30 | Fraction of answers including authoritative source citations |
| Actionability Score | 0.235 | Proportion of imperative action verbs in generated guidelines |

### Comparison with Prior Work

| System | Context Precision | Faithfulness | Answer Relevancy |
|---|---|---|---|
| CyberBOT (Zhao et al., 2025) — GPT-4 | 0.740 | 0.820 | 0.790 |
| RAGAS baseline (Es et al., 2024) — GPT-3.5 | 0.680 | 0.710 | 0.740 |
| **This work — Llama 3.3 70B (free)** | **0.917** | **0.862** | 0.673 |

This system exceeds CyberBOT on both context precision (+0.177) and faithfulness (+0.042) using zero-cost open-source infrastructure.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  LAYER 1 — KNOWLEDGE BASE               │
│                                                         │
│  PDF Sources (10 docs)          Q&A Dataset             │
│  NIST · CISA · APWG · arXiv     Rowden/CybersecurityQAA │
│         ↓                              ↓                │
│    Text Extraction            Q+A Formatting            │
│    Chunking (500w, 10% overlap)   (1 row = 1 chunk)     │
│         ↓                              ↓                │
│         └──────────────┬───────────────┘                │
│                        ↓                                │
│         Embedding: all-MiniLM-L6-v2 (384-dim)           │
│         Vector Store: ChromaDB HNSW (cosine sim)        │
│         Total: 2,775 chunks                             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│             LAYER 2 — RETRIEVAL + GENERATION            │
│                                                         │
│  Practitioner Query                                     │
│       ↓                                                 │
│  Query Embedding → Top-3 Retrieval (cosine similarity)  │
│       ↓                                                 │
│  Prompt Assembly (system rules + context + query)       │
│       ↓                                                 │
│  Llama 3.3 70B via Groq (temp=0.2, max_tokens=512)      │
│       ↓                                                 │
│  3 Structured Guidelines with source citations          │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│                  LAYER 3 — EVALUATION                   │
│                                                         │
│  RAGAS: Context Precision · Faithfulness · Answer Rel.  │
│  Custom: Source Diversity · Citation · Actionability    │
│  Judge LLM: Llama 3.1 8B via Groq (separate quota)      │
└─────────────────────────────────────────────────────────┘
```

### Component Justification

| Component | Tool | Reason Selected |
|---|---|---|
| Embeddings | `all-MiniLM-L6-v2` | 384-dim, CPU-compatible, strong semantic similarity benchmark performance (Reimers & Gurevych, 2019) |
| Vector store | ChromaDB (HNSW) | Persistent, pure Python, no server required, sub-linear query time |
| Generation LLM | Llama 3.3 70B via Groq | Open-source (Meta licence), outperforms GPT-3.5 on reasoning, 300 tok/s on Groq LPU |
| RAGAS judge | Llama 3.1 8B via Groq | Separate daily token quota; sufficient for binary claim-grounding classification |
| Evaluation framework | RAGAS 0.4.x | Reference-free (no ground truth needed); standard in RAG evaluation literature |
| Chunk size | 500 words (10% overlap) | Within all-MiniLM-L6-v2's 512-token limit; optimal per Juvekar & Purwar (2024) |

---

## Knowledge Base

The knowledge base blends two complementary source types:

### PDF Sources (auto-downloaded in cell 2.1)

| Document | Publisher | Year | Chunks | Notes |
|---|---|---|---|---|
| NIST SP 800-61r3 | NIST | Apr 2025 | 32 | Supersedes 800-61r2; incident response |
| NIST SP 800-53r5 | NIST | Sep 2020 | 442 | 1,196 security/privacy controls |
| NIST SP 800-53Ar5 | NIST | Jan 2022 | 520 | Assessment procedures for 800-53 |
| NIST SP 800-177r1 | NIST | Feb 2019 | 99 | SPF, DKIM, DMARC email authentication |
| NIST SP 800-115 | NIST | Sep 2008 | 77 | Technical security testing guide |
| CISA Phishing Guidance | CISA/NSA/FBI/MS-ISAC | Oct 2023 | 9 | Joint agency phishing defence |
| CISA Counter-Phishing | CISA | Aug 2023 | 4 | Non-federal organisation guidance |
| APWG Q4 2024 | APWG | Mar 2025 | 7 | Phishing threat intelligence |
| APWG Q1 2025 | APWG | Jul 2025 | 7 | Most recent quarterly report |
| Persuasion & Phishing | arXiv (2412.18488) | Dec 2024 | 16 | Social engineering survey |

### CISA PDFs (manual download required)

CISA blocks automated downloads. Save these two files to your `docs/` folder manually:

```
cisa_phishing_guidance_2023.pdf
→ https://www.cisa.gov/sites/default/files/2023-10/Phishing%20Guidance%20-%20Stopping%20the%20Attack%20Cycle%20at%20Phase%20One_508c.pdf

cisa_counter_phishing_2023.pdf
→ https://www.cisa.gov/sites/default/files/2023-09/CISA_CEG_Counter-Phishing_Guidance_for_Non_Federal_Orgs%20Aug-23%20Revision.pdf
```

### Q&A Dataset (auto-loaded via HuggingFace)

| Dataset | Source | Rows | Validation |
|---|---|---|---|
| `Rowden/CybersecurityQAA` | Squire & Thornton (2024) | 1,562 | 449 expert-validated; full non-expert validated |

Each row is grounded in a named standard clause (PCI DSS, NIST SP 800-53, ISO 27001). The `evidence` field links each answer to its source text, enabling RAGAS faithfulness verification.

---

## Repository Structure

```
assignment3-rag-cybersecurity/
│
├── README.md                              # this file
├── assignment3_RAG_system.ipynb           # main notebook (40 cells)
│
├── outputs/
│   ├── ragas_scores_final.csv             # per-query RAGAS scores (3 metrics)
│   ├── additional_metrics.csv             # source diversity, citation, actionability
│   ├── rag_results.json                   # generated guidelines for all 10 queries
│   ├── ragas_scores_chart.png             # Section 6.1 bar chart
│   ├── chart1_metric_heatmap.png          # Section 6.4 heatmap
│   ├── chart2_source_diversity.png        # Section 6.4 source diversity
│   ├── chart3_actionability_score.png     # Section 6.4 actionability
│   ├── chart4_knowledge_base_composition.png # Section 6.4 knowledge base pie
│   ├── chart5_citation_presence.png       # Section 6.4 citation presence
│   ├── chart6_score_distribution_boxplot.png # Section 6.4 score distribution
│   └── comparison_prior_work.png          # Section 6.5 prior work comparison
│
└── docs/
    └── README_sources.md                  # manual download instructions for CISA PDFs
```

> **Note:** `chunks_index.json` (~80 MB) and the `chroma_db/` folder are not included in this repository — both are regenerated automatically when the notebook is run. The ChromaDB index persists to Google Drive between sessions.

---

## Reproduction Instructions

### Platform

The notebook is designed for **Google Colab** and is platform-independent — no local Python environment, no GPU, and no paid services are required. It runs identically on Windows, macOS, and Linux because execution is entirely cloud-based.

### Prerequisites

1. A Google account (for Colab and Google Drive)
2. A free Groq API key — sign up at [console.groq.com](https://console.groq.com) (email only, no credit card)

### Step-by-Step Instructions

**Step 1 — Open the notebook in Colab**
```
File → Upload notebook → select assignment3_RAG_system.ipynb
```
Or click: [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com)

**Step 2 — Add your Groq API key to Colab Secrets**
```
Click the Sceret key icon in the left sidebar
→ Add new secret
→ Name:  GROQ_API_KEY
→ Value: gsk_your_key_here
→ Toggle "Notebook access" ON
```

**Step 3 — Mount Google Drive (cell 1.1b)**

Run cell 1.1b and authorise access when prompted. All outputs will be saved to `MyDrive/assignment3_RAG/` and will survive session disconnections.

**Step 4 — Run cells top to bottom**

> Do NOT use Runtime → Run all. Run cells one at a time so you can catch any issues.

| Cell | Action | Expected time |
|---|---|---|
| 1.1 | Install packages + patch ragas | ~3 min |
| 1.1b | Mount Google Drive | ~30 sec |
| 1.2–1.4 | Imports, config, Groq test | ~30 sec |
| 2.1 | Download 8 PDFs automatically | ~3 min |
| 2.1 (manual) | Upload 2 CISA PDFs to `docs/` folder | ~2 min |
| 2.2–2.6 | Extract text, chunk, embed, index | ~15–20 min |
| 3.1–3.2 | Retrieval function + spot-check | ~1 min |
| 4.1–4.3 | Generation for 10 queries | ~2 min |
| 5.1–5.5 | RAGAS + additional metrics | ~35–40 min |
| 6.1–6.5 | Charts and analysis | ~2 min |
| 7.1–7.3 | Ethics and final summary | ~1 min |
| Final | Download outputs as zip | ~1 min |
| **Total** | | **~60–70 min** |

**Step 5 — Upload CISA PDFs (during cell 2.1)**

After cell 2.1 runs, open the Files panel in Colab (folder icon in left sidebar), navigate to `content/drive/MyDrive/assignment3_RAG/docs/`, and upload:
- `cisa_phishing_guidance_2023.pdf`
- `cisa_counter_phishing_2023.pdf`

Then continue from cell 2.2.

**Step 6 — Resume after disconnection**

If Colab disconnects mid-run, re-run cells 1.1 → 1.4, then skip Section 2 entirely (ChromaDB is already saved to Drive), and resume from wherever you left off. Generation and RAGAS evaluation both use checkpoint files that save progress after every query.

### Expected Output

After a successful run, `MyDrive/assignment3_RAG/` will contain:

```
chunks_index.json               (~80 MB — full knowledge base)
rag_results.json                (10 generated guideline sets)
ragas_scores_final.csv          (per-query RAGAS scores)
additional_metrics.csv          (diversity, citation, actionability)
ragas_checkpoint.json           (evaluation checkpoint)
generation_checkpoint.json      (generation checkpoint)
ragas_scores_chart.png          + 7 other chart images
assignment3_outputs.zip         (all files zipped for download)
```

---

## Dependencies

All packages are installed automatically in cell 1.1. Versions listed below are those confirmed working at time of submission.

| Package | Version | Purpose |
|---|---|---|
| `pypdf` | 4.x | PDF text extraction |
| `sentence-transformers` | 3.x | all-MiniLM-L6-v2 embeddings |
| `chromadb` | 0.5.x | Vector store with HNSW index |
| `datasets` | 2.x | Load CybersecurityQAA from HuggingFace |
| `openai` | 1.x | Groq API client (OpenAI-compatible) |
| `ragas` | 0.4.x | RAGAS evaluation framework |
| `langchain-openai` | 0.1.x | RAGAS LLM wrapper for Groq |
| `langchain-community` | 0.2.x | HuggingFace embeddings for RAGAS |
| `langchain-huggingface` | 0.0.x | HuggingFace integration |
| `matplotlib` | 3.x | Evaluation charts |
| `pandas` | 2.x | Results DataFrames |
| `tqdm` | 4.x | Progress bars |

> **Python version:** 3.12 (Google Colab default as of 2025)

---

## Troubleshooting

Common issues encountered during development and their solutions:

### `ModuleNotFoundError: No module named 'langchain_community.chat_models.vertexai'`
ragas 0.4.x has a broken import of ChatVertexAI that fails unless google-cloud-aiplatform is installed. Cell 1.1 automatically patches the `ragas/llms/base.py` file to remove this import after every install. If the error persists, re-run cell 1.1 without restarting the runtime first.

### `BadRequestError: 'n': number must be at most 1`
Groq hard-caps the `n` parameter at 1. RAGAS internally requests `n=3` for reproducibility scoring. The `GroqSafeChatOpenAI` subclass in cell 5.1 forces `n=1` on both synchronous and asynchronous generation paths. If this error appears, confirm cell 5.1 ran successfully before cell 5.3.

### `RateLimitError 429 — tokens per day exceeded`
Groq's free tier provides 100,000 tokens per day on `llama-3.3-70b-versatile`. RAGAS evaluation uses a separate model (`llama-3.1-8b-instant`) with its own 100k daily quota. If the generation model quota is exhausted, wait until midnight UTC for the quota to reset, then re-run cell 4.3 — the checkpoint will skip already-completed queries automatically.

### RAGAS evaluation produces `nan` scores
This occurs when all jobs fail due to rate limiting or timeouts. Cell 5.3 evaluates one query at a time with a 20-second sleep between queries and a 30-second sleep between metrics to respect the 6,000 tokens/minute limit. If `nan` scores appear, delete the RAGAS checkpoint file (`ragas_checkpoint.json`) and re-run cell 5.3.

### Colab disconnects during embedding (cell 2.6)
The ChromaDB index persists to Google Drive. On reconnect, re-run cells 1.1–1.4 to restore imports and the Groq client, then skip Section 2 entirely and continue from Section 3.

### CISA PDF shows very few chunks (< 5)
This is expected — the CISA Counter-Phishing document is only 1,475 words (4 chunks) and the CISA Phishing Guidance is 4,089 words (9 chunks). These are short documents, not extraction failures. Confirmed via `extract_pdf_text()` word count diagnostic in cell 2.2.

---

## Ethical Considerations

This system has known limitations that must be understood before any deployment:

- **US-regulatory bias:** ~97% of PDF content originates from US federal standards bodies (NIST, CISA, FBI). Generated guidelines referencing CISA or FBI IC3 reporting channels are not applicable to EU, Australian, or other non-US practitioners.
- **Citation compliance:** Only 3/10 generated answers include source citations despite explicit system prompt instruction. Do not assume citations will always be present.
- **Hallucination risk:** Faithfulness ranges from 0.636 to 1.000. The lowest-scoring queries (Q8: simulation frequency) contain claims not grounded in retrieved context. All outputs should be reviewed by a qualified security analyst before practitioner use.
- **Advisory only:** This system is not a substitute for qualified incident response professionals or legal compliance counsel.

See Section VI of the accompanying report for full ethical analysis and governance recommendations.

---

## References

| Citation | Paper |
|---|---|
| Lewis et al. (2020) | Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks. NeurIPS 2020. |
| Es et al. (2024) | RAGAS: Automated Evaluation of Retrieval Augmented Generation. EACL 2024. |
| Zhao et al. (2025) | CyberBOT: Towards Reliable Cybersecurity Education via Ontology-Grounded RAG. arXiv:2504.00389. |
| Blefaria et al. (2025) | CyberRAG: An Agentic RAG Cyber-Attack Classification and Reporting Tool. arXiv:2507.02424. |
| Reimers & Gurevych (2019) | Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks. EMNLP 2019. |
| Juvekar & Purwar (2024) | Introducing a new hyper-parameter for RAG: Context Window Utilization. arXiv:2407.19794. |
| Squire & Thornton (2024) | Evaluating the Effectiveness of LLMs as Cybersecurity Advisors. Journal of Cybersecurity Research. |
| APWG (2025) | Phishing Activity Trends Report, Q1 2025. |
| NIST (2025) | SP 800-61r3: Incident Response Recommendations. April 2025. |
| CISA (2023) | Phishing Guidance: Stopping the Attack Cycle at Phase One. October 2023. |

---

## Acknowledgements

Built as part of the Advanced Topics in Artificial Intelligence and Machine Learning course, University of South Australia. All components use open-source tools and free-tier APIs.
