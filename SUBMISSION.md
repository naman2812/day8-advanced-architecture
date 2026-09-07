# Day 8 Assignment Submission Guide

Use this document to submit your deliverable in the guided project workspace.

---

## 📋 Deliverable 1: 2–3 Sentences on the Most Critical Failure Point and Mitigation

> **The most critical failure point is an ungrounded hallucination bypassing the evaluation guardrail, which directly damages institutional credibility and user trust.** To mitigate this, the architecture enforces a deterministic citation-grounding validation layer where generated statements must link to explicit retrieved chunk IDs verified via Natural Language Inference (NLI) and regex pattern matching. Any output failing this strict threshold is intercepted before rendering and replaced by a deterministic, pre-approved safe response (`"I cannot verify this information based on the available data."`).

---

## 📊 Deliverable 2: Updated Mermaid Architecture Diagram

```mermaid
graph TD
    %% Styling Definitions
    classDef client fill:#1E293B,stroke:#475569,stroke-width:2px,color:#F8FAFC;
    classDef gateway fill:#0F766E,stroke:#14B8A6,stroke-width:2px,color:#FFFFFF;
    classDef core fill:#1D4ED8,stroke:#3B82F6,stroke-width:2px,color:#FFFFFF;
    classDef fallback fill:#B91C1C,stroke:#EF4444,stroke-width:2px,color:#FFFFFF,stroke-dasharray: 4 4;
    classDef eval fill:#B45309,stroke:#F59E0B,stroke-width:2px,color:#FFFFFF;
    classDef telemetry fill:#581C87,stroke:#A855F7,stroke-width:2px,color:#FFFFFF;
    classDef cache fill:#047857,stroke:#10B981,stroke-width:2px,color:#FFFFFF;

    A[User Request]:::client --> B[API Gateway / Web Interface]:::gateway
    B --> B_Cache{Semantic Cache Lookup}:::gateway
    
    B_Cache -- "Cache Hit (Cosine >= 0.95)" --> B_Hit[Deliver Cached Response]:::cache
    B_Cache -- "Cache Miss" --> C{Query Analyzer & Rewriter}:::core
    
    C --> D[Vector DB Retriever]:::core
    
    %% Failure Path 1 & Fallback
    D -- "Failure 1: Low Relevance Score (< 0.70)" --> E[Fallback: Execute Web Search Tool]:::fallback
    D -- "High Relevance (>= 0.70)" --> F[Async Queue: Cross-Encoder Reranker]:::core
    E --> F
    
    F --> G{Context Monitor}:::core
    
    %% Failure Path 2 & Fallback
    G -- "Failure 2: Token Limit Exceeded (> 80%)" --> H[Fallback: Trigger Map-Reduce Summarization]:::fallback
    G -- "Optimal Tokens (<= 80%)" --> I[Generator LLM]:::core
    H --> I
    
    I --> J{Eval Gate: Faithfulness Guardrail}:::eval
    
    %% Failure Path 3 & Fallback
    J -- "Failure 3: Hallucination / Low Grounding" --> K[Fallback: Deterministic Safe Response]:::fallback
    J -- "Pass: Grounded & Faithfulness Verified" --> L[Synthesized Output Delivered]:::client

    %% Eval & Telemetry Loop
    J -.- M[(Telemetry & Metrics DB)]:::telemetry
    M -. "Alert Trigger: Refusal Spike (> 15%)" .-> N[PagerDuty P1 Incident]:::telemetry
    M -. "Auto-Rollback Trigger: Faithfulness Drop (< 0.85)" .-> O[CI/CD Auto-Rollback Webhook]:::telemetry
    O -. "Rollback to Golden Version" .-> I
```

---

## 🛠️ Step-by-Step GitHub Submission Instructions

To push this repository to GitHub and submit your link:

1. **Create a new repository** on GitHub (e.g. `day8-advanced-architecture` or `rag-advanced-architecture`).
2. Run the following terminal commands in the project folder (`C:\Users\Naman\Desktop\day8_advanced_architecture`):
   ```bash
   git init
   git add .
   git commit -m "feat: complete Day 8 advanced architecture with fallbacks, scaling, and rollback mechanics"
   git branch -M main
   git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/day8-advanced-architecture.git
   git push -u origin main
   ```
3. Copy the URL of your new GitHub repository:
   `https://github.com/<YOUR_GITHUB_USERNAME>/day8-advanced-architecture`
4. Paste the URL into the guided project workspace submission field to earn your **5 points**!
