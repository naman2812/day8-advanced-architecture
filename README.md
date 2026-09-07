# Advanced RAG System Architecture: Reliability, Scaling & Guardrails
> **Day 8 Assignment Deliverable (Foundational & Advanced)**: Resilient Multi-Stage RAG Pipeline with System Boundaries, Circuit Breakers, Async Reranking, and Continuous Evaluation Rollback.

---

## 📌 Deliverable Summaries

### Deliverable A: 2–3 Sentences Explaining Pillar Choices (Foundational Day 8)
> **For this enterprise knowledge assistant, we selected Retrieval-Augmented Generation (RAG) and Continuous Evaluation & Guardrails as our two primary pillars.** RAG is required because our enterprise documentation updates dynamically and demands deterministic grounding without costly retraining. Continuous Evaluation & Guardrails are essential because user-facing accuracy requires real-time hallucination prevention and automated rollback triggers to maintain strict regulatory compliance.

### Deliverable B: 2–3 Sentences on Most Critical Failure Point and Mitigation (Advanced Day 8)
> **The most critical failure point is an ungrounded hallucination bypassing the evaluation guardrail, which directly damages institutional credibility and user trust.** To mitigate this, the architecture enforces a deterministic citation-grounding validation layer where generated statements must link to explicit retrieved chunk IDs verified via Natural Language Inference (NLI) and regex pattern matching. Any output failing this strict threshold is intercepted before rendering and replaced by a deterministic, pre-approved safe response (`"I cannot verify this information based on the available data."`).

---

## 🏛️ Complete System Architecture Diagram

```mermaid
graph TD
    %% Node Styling Definitions
    classDef client fill:#1E293B,stroke:#475569,stroke-width:2px,color:#F8FAFC;
    classDef core fill:#1D4ED8,stroke:#3B82F6,stroke-width:2px,color:#FFFFFF;
    classDef fallback fill:#B91C1C,stroke:#EF4444,stroke-width:2px,color:#FFFFFF,stroke-dasharray: 4 4;
    classDef eval fill:#B45309,stroke:#F59E0B,stroke-width:2px,color:#FFFFFF;
    classDef telemetry fill:#581C87,stroke:#A855F7,stroke-width:2px,color:#FFFFFF;
    classDef cache fill:#047857,stroke:#10B981,stroke-width:2px,color:#FFFFFF;
    classDef note fill:#374151,stroke:#9CA3AF,stroke-width:1px,color:#F9FAFB,stroke-dasharray: 3 3;

    %% Client Layer
    subgraph Client_Layer["User Layer"]
        A["User Request"]:::client
        L["Synthesized Output Delivered"]:::client
    end

    %% Internal Boundary (Your Infrastructure)
    subgraph Internal_Boundary["Internal System Boundary - Your Infrastructure"]
        B["API Gateway & Semantic Cache"]:::core
        B_Cache{"Semantic Cache Lookup"}:::core
        B_Hit["Deliver Cached Response"]:::cache
        C{"Query Analyzer & Rewriter"}:::core
        
        D[("Internal Vector DB / Hybrid Retriever")]:::core
        
        F["Async Queue: Reranker Jobs"]:::core
        F1["Cross-Encoder Reranker Workers"]:::core
        
        G{"Context Window Monitor"}:::core
        H["Fallback 2: Map-Reduce Summarization"]:::fallback
        
        J{"Eval Gate: Faithfulness Guardrail"}:::eval
        K["Fallback 3: Deterministic Safe Response"]:::fallback

        %% NOTE: Where the Eval Collector sits
        EvalCollector["NOTE: Eval Collector (Sits inline post-generation)"]:::note
        TelemetryDB[("Telemetry & Metrics DB")]:::telemetry
    end

    %% External Boundary (Third-Party Services)
    subgraph External_Boundary["Third-Party Managed Services Boundary"]
        E["Fallback 1: External Search Agent Tool (SerpAPI / Tavily)"]:::fallback
        I["Generator LLM API (OpenAI / Anthropic / Vertex AI)"]:::core
        AlertService["PagerDuty / Opsgenie Incident Response"]:::telemetry
        CICD["CI/CD Orchestrator (GitHub Actions / Argo Rollouts)"]:::telemetry
    end

    %% Workflow Connections
    A --> B
    B --> B_Cache
    B_Cache -- "Cache Hit (Cosine >= 0.95)" --> B_Hit --> L
    B_Cache -- "Cache Miss" --> C
    C --> D

    %% Failure Point 1 & Fallback
    D -- "Failure 1: Low Relevance Score (< 0.70)" --> E
    D -- "High Relevance (>= 0.70)" --> F
    E -- "Inject Real-Time Web Context" --> F

    F --> F1 --> G

    %% Failure Point 2 & Fallback
    G -- "Failure 2: Token Limit Exceeded (> 80%)" --> H
    G -- "Optimal Token Count (<= 80%)" --> I
    H -- "Compressed Context" --> I

    I --> J
    J -- "Failure 3: Hallucination / Low Grounding" --> K --> L
    J -- "Pass: Faithfulness & Grounding Verified" --> L

    %% Eval Collector & Continuous Loop
    J --> EvalCollector
    EvalCollector -. "Stream Evaluation Telemetry" .-> TelemetryDB
    
    TelemetryDB -. "Refusal Spike Alert (> 15%)" .-> AlertService
    TelemetryDB -. "Auto-Rollback Trigger: Faithfulness Drop (< 0.85)" .-> CICD
    CICD -. "Rollback to Golden Prompt/Model" .-> I
```

---

## 🏛️ System Boundary Breakdown (Yours vs. Third-Party)

| Environment | Component Name | Role & Responsibility |
|---|---|---|
| **Your System (Internal)** | **API Gateway & Semantic Cache** | Edge termination, rate limiting, and Redis similarity cache ($\ge 0.95$). |
| **Your System (Internal)** | **Query Analyzer & Rewriter** | Query decomposition, entity extraction, and vector query rewriting. |
| **Your System (Internal)** | **Vector DB & Hybrid Retriever** | Internal vector similarity search (Dense + BM25 sparse search). |
| **Your System (Internal)** | **Async Reranker Queue & Workers** | Decoupled GPU worker pool executing Cross-Encoder reranking. |
| **Your System (Internal)** | **Context Window Monitor** | Pre-inference token counter protecting against context window blowouts. |
| **Your System (Internal)** | **Eval Gate & Collector** | Inline NLI evaluation judge verifying citation grounding and faithfulness. |
| **Your System (Internal)** | **Telemetry & Metrics DB** | Prometheus / ClickHouse time-series store tracking refusal & quality drift. |
| **Third-Party (External)** | **External Search Tool** | External agent (SerpAPI / Tavily) fetching real-time web context. |
| **Third-Party (External)** | **Generator LLM Provider** | Hosted model endpoint (e.g. OpenAI GPT-4o / Anthropic Claude / Vertex AI). |
| **Third-Party (External)** | **PagerDuty / Alerting** | On-call incident response triggering when refusal anomalies spike. |
| **Third-Party (External)** | **CI/CD Deployment Engine** | Automated deployment pipeline (GitHub Actions / Argo) managing rollback webhooks. |

---

## 🛡️ 3 Identified Failure Points & Explicit Fallback Behaviors

| # | Failure Point | Detection Trigger | Explicit Fallback Behavior |
|---|---|---|---|
| **1** | **Retriever Failure (Low Relevance / Empty Set)** | Vector similarity scores return chunks below `< 0.70`, or empty chunk sets. | **Web Search Fallback:** System branches to an external tool agent (SerpAPI / Tavily) to retrieve authoritative web documents, embed them on-the-fly, and inject them into the reranker without terminating the session. |
| **2** | **Context Window Overflow** | Accumulated tokens (history + system prompt + retrieved chunks) exceed $80\%$ of model context limit. | **Map-Reduce Token Compression:** Middleware intercepts payload and runs hierarchical Map-Reduce summarization via a fast lightweight model, compressing text by 60–75% while preserving chunk IDs and facts. |
| **3** | **Eval Gate Rejection (Hallucination)** | Automated NLI Faithfulness Judge detects ungrounded assertions or hallucinated claims ($< 0.85$). | **Deterministic Safe Response:** System discards the draft response and serves a pre-approved safe fallback message: `"I cannot verify this information based on the available data."` |

---

## ⚡ Scaling Consideration: The Reranker Bottleneck

* **Component Bottlenecking First:** **Cross-Encoder Reranker**.
* **Why it bottlenecks:** Unlike Bi-Encoders that compute independent embeddings, Cross-Encoders pass query and document pairs jointly through full cross-attention ($\mathcal{O}(K \cdot L^2)$). Under high concurrency (500+ QPS), scoring 50 candidate chunks per request saturates GPU VRAM and blocks the asynchronous event loop.
* **Mitigation:**
  1. **Decouple the Reranker:** Offload scoring to an asynchronous Redis queue with auto-scaling GPU workers (KEDA).
  2. **Semantic Caching:** Deploy a Redis semantic cache at the gateway to return cached responses in $< 15\text{ms}$ for queries with cosine similarity $\ge 0.95$.

---

## 📊 Eval Integration: Alerts & Autonomous Rollbacks

1. **Where the Eval Collector Sits:** The **Eval Collector** sits inline directly between the Generator LLM and the final output gate. It evaluates the generated draft against retrieved context chunks before the response reaches the user.
2. **Alert Trigger (Goodhart’s Law Detection):** If the **Refusal Rate** spikes by $> 15\%$ while faithfulness remains near 1.0, an automated **PagerDuty P1 alert** notifies engineers that guardrails have become over-conservative.
3. **Auto-Rollback Trigger (Quality Regression Guard):** If average Faithfulness drops below `0.85` across 100 requests following a deployment, an automated webhook notifies GitHub Actions / Argo Rollouts to immediately roll back traffic to the previous **Golden Release**.

---

## 🚀 Repository Files

- [README.md](file:///C:/Users/Naman/Desktop/day8_advanced_architecture/README.md)
- [ARCHITECTURE.md](file:///C:/Users/Naman/Desktop/day8_advanced_architecture/ARCHITECTURE.md)
- [SUBMISSION.md](file:///C:/Users/Naman/Desktop/day8_advanced_architecture/SUBMISSION.md)
- [diagram.mmd](file:///C:/Users/Naman/Desktop/day8_advanced_architecture/diagram.mmd)
- [index.html](file:///C:/Users/Naman/Desktop/day8_advanced_architecture/index.html)
