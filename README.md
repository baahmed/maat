# Maat: The Agentic Legal Research Assistant for Competition Protection

Basant Mounir\*, Farida Madkour\*, Amira Abdelaziz\*, Asmaa Sami\*

This repository accompanies the paper *Maat: The Agentic Legal Research Assistant for Competition Protection*. It releases the case database, prompts, evaluation rubric, anonymized expert scores, and experimental results. **The source code of Maat is confidential institutional property and is not included.**

## About

Competition law experts spend much of their time locating and verifying precedents across large volumes of decisions. General assistants such as ChatGPT and Claude hallucinate cases, have shallow domain knowledge, and rarely give page-level citations. Legal assistants such as SaulLM-7B and LawChat are not grounded in official sources.

Maat is a multi-turn **ReAct agent** that routes an expert's request among specialized tools for four research tasks:

| Task | What Maat does |
|---|---|
| Precedent search | Finds cases in a curated database and falls back to verified web search when the database has no match |
| Precedent summarization | Produces a structured summary (background, case progression, market definition, legal and economic tests, conclusion) with page-level in-line citations |
| Precedent question-answering | Answers questions about a specific case using RAG over the (translated) decision text |
| Theoretical understanding | Builds a multi-stage, source-grounded explanation of a competition law concept using EU guidelines and decisions |

When a request is too ambiguous to route, Maat asks the user for clarification.

## Architecture

The agent has five tools: `database_search`, `web_search`, `answer_case`, `answer_theoretical`, and `ask_clarification`. A memory layer holds the ReAct scratchpad, chat history, and the cases retrieved so far (`session_cases`).

- **Database search.** The query is turned into a six-field vector (`case_id`, `case_title`, `jurisdiction`, `legal_basis`, `sector`, `companies`) and used as a filter. Titles are matched by cosine similarity (threshold 0.85) and companies by canonicalized-name subset matching. The top 5 cases are summarized.
- **Web search.** Perplexity Sonar proposes up to five cases. Each title is checked against official sources, and titles with none are discarded as hallucinated. The rest are filtered against the query's legal basis, sector, and companies. This raised retrieval precision from 51.0% to 85.7% on 20 test questions.
- **Case Q&A.** The top 8 chunks of the case text (by cosine similarity) are injected into a meta-prompted answer prompt with in-line citations.
- **Theoretical answers.** Six stages: question analysis, initial web response, outline planning, passage and case retrieval from EU guidelines, section-by-section refinement, and overview assembly.

The case documents are indexed with LlamaIndex (about 1,024-token chunks with 20-token overlap), embedded with `text-embedding-3-small`, and stored in Qdrant. The UI is a Flask web application.

## Dataset

[`competition_decisions.csv`](competition_decisions.csv) consolidates **11,489 decisions** from five authorities, de-duplicated across languages with English preferred:

| Jurisdiction | Source | Decisions |
|---|---|---|
| French | Autorité de la concurrence | 5,903 |
| Spanish | CNMC | 2,292 |
| Italian | AGCM | 2,041 |
| EU | European Commission (DG COMP) | 711 |
| German | Bundeskartellamt | 542 |

Data collection respected each site's `robots.txt` and throttled requests. Each competition authority approved building and sharing the database.

**Fields:**

| Field | Description |
|---|---|
| `url` | Link to the decision text |
| `jurisdiction` | EU, German, French, Spanish, or Italian |
| `case_id` | Unique case identifier |
| `title` | Description of the case |
| `decision_year` | Year of the decision |
| `nace_code`, `nace_label` | NACE section of the underlying economic activity. A coarse search facet, not a market definition |
| `legal_basis` | Legal basis of the case (for example Art. 101 TFEU, Art. 102 TFEU, merger) |
| `companies` | Raw company names |
| `companies_canonicalized` | Canonicalized company names used for search |
| `docx_path` | File in [`Translations/`](Translations) holding the English translation, when the original was not English |

Non-English decisions were translated to English with a few-shot `gpt-4o-mini` pipeline, so experts can verify Maat's answers. The translated documents are in [`Translations/`](Translations). The French sector labels were mapped to NACE using [`adlc_nace_sector_map.xlsx`](adlc_nace_sector_map.xlsx).

## Repository structure

```
.
├── competition_decisions.csv          # Combined case database (11,489 decisions)
├── adlc_nace_sector_map.xlsx          # French sector → NACE mapping
├── Translations/                      # English translations of non-English decisions
├── Theoretical Guidelines/            # EU guidelines used by answer_theoretical
├── Prompts/
│   ├── Agent/                         # ReAct agent system prompt
│   ├── Search Database/               # Query-vector field extraction
│   ├── Search Web/                    # Title retrieval, legal-basis/sector checks, vector alignment
│   ├── Case Summarization/            # One prompt per summary field (EU and US progression variants)
│   ├── Answer Case/                   # Active-case detection and question answering
│   ├── Answer Theoretical/            # The six-stage theoretical pipeline
│   └── Translation/                   # Translation prompt and LLM-as-a-judge evaluator
├── Experiments/
│   ├── Dataset Fields Extraction/     # 75-case manual review of LLM-extracted metadata
│   ├── Company Match/                 # Company-name canonicalization test set and error analysis
│   ├── Translation/                   # chrF++ / BLEU scores and LLM-judge results
│   ├── Web Search/                    # 20 questions, responses, and filter evaluation
│   └── Resource Utilization/          # Cost and runtime by tool chain
└── Evaluation/
    ├── Reviewer Material/             # Blinded case and theory responses (Q1–Q3) and the rubric
    ├── Raw Responses/                 # Anonymized scores from Reviewers 1–4
    └── Aggregated Responses.xlsx
```

## Data quality

| Check | Sample | Result |
|---|---|---|
| Title accuracy | 75 cases | 98.7% |
| Company list precision / recall | 75 cases | 92.9% / 82.4% |
| Sector accuracy | 75 cases | 82.7% |
| Company-name canonical match recall | 75 search terms | 86.2% |
| Translation, median BLEU / chrF++ | 300 document pairs (75 per language) | 38.0 / 64.6 |

The 75-case criteria are in [`Experiments/Dataset Fields Extraction/criteria.txt`](Experiments/Dataset%20Fields%20Extraction/criteria.txt). Confidence intervals and error analyses are in the paper. French has the lowest translation scores (median BLEU 32.3, chrF++ 55.8). Translations are adequate for retrieval and understanding but are not a substitute for the authentic decision text.

## Evaluation

Four external competition law practitioners blindly scored responses to three case-specific and three theoretical questions. Responses were identically formatted and shuffled. Each score (0–5) averages five criteria: coherence of legal logic, correct legal taxonomy, illustration of concepts, quality of references, and in-line citations.

| System | Cases, mean (SD) | Theory, mean (SD) |
|---|---|---|
| **Maat** | **3.84 (0.73)** | **3.93 (0.58)** |
| GPT-5.6 | 3.38 (1.21) | 2.49 (0.59) |
| Claude Sonnet 5 | 3.17 (1.09) | 2.52 (1.00) |
| MaatRAG (database search only) | 2.70 (1.98) | n/a |
| Perplexity Sonar | n/a | 3.00 (0.96) |
| LawChat | 1.09 (1.25) | 0.83 (0.24) |
| SaulLM-7B | 0.96 (0.98) | 0.78 (0.42) |

Each system has 12 scores per category (4 reviewers × 3 questions).

- **Theoretical questions:** Maat's advantage held against all five baselines (Holm-adjusted p = 0.002–0.014). General-purpose models matched Maat on legal reasoning but scored near zero on references and in-line citations. Bare Sonar had sources but weaker legal analysis. Maat combines both.
- **Case questions:** Maat's advantage was consistent over LawChat and SaulLM-7B (adjusted p = 0.002). Its lead over GPT-5.6, Claude, and MaatRAG was directional only (adjusted p = 0.19–0.33).
- **Agentic ablation:** MaatRAG matched Maat when the case was in the database. For a decision not yet indexed, MaatRAG wrongly said the case did not exist. Maat's agent fell back to web search and answered correctly (scores 0.05 vs 3.24). This rests on a single question.
- **Reviewer agreement:** Kendall's W was 0.77 (cases) and 0.83 (theory).

The paper's main conclusion is that as foundation models converge on legal reasoning, the durable value of a domain-specific system lies in grounding answers in authoritative, verifiable sources.

## Resource utilization

Average per query over 14 questions:

| Tool chain | Cost (USD) | Runtime |
|---|---|---|
| Theoretical answer | ~$0.44 | ~189 s |
| Database search | ~$0.03 | ~135–137 s |
| Database search, then web search | ~$0.09 | ~135–137 s |

`gpt-4o` accounts for about 98% of the theoretical-answer cost. Swapping it for `gpt-4o-mini` is estimated to cut that to about $0.04. Details are in [`Experiments/Resource Utilization/`](Experiments/Resource%20Utilization/cost_and_time_by_route.xlsx).

## Limitations

- The evaluation is small (4 reviewers, 3 questions per category), so results describe this evaluation set and cannot be generalized with statistical confidence.
- Maat's answers have a distinctive structure that reviewers may have recognized despite identical formatting.
- Sectors are classified only at the NACE section level, which may be too coarse for precise sector search.
- The database covers the EU, Germany, France, Spain, and Italy. Extension to U.S. and selected African jurisdictions is planned.

## Citation

```bibtex
@article{mounir2026maat,
  title  = {Maat: The Agentic Legal Research Assistant for Competition Protection},
  author = {Mounir, Basant and Madkour, Farida and Abdelaziz, Amira and Sami, Asmaa}
}
```

\* Equal contribution.
