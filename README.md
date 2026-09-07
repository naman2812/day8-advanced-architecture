# Advanced RAG System Architecture: Reliability, Scaling & Guardrails

> **Day 8 Assignment Deliverable (Foundational + Advanced)** — Resilient Multi-Stage RAG Pipeline with clear System Boundaries, 3 Circuit Breakers, Decoupled GPU Reranking, and Continuous Evaluation Rollback.

---

## Deliverable A — Pillar Choices (Day 8 Foundational)

For this enterprise knowledge assistant, we selected **Retrieval-Augmented Generation (RAG)** and **Continuous Evaluation & Guardrails** as our two primary pillars. RAG is required because enterprise documentation updates dynamically and demands deterministic grounding without costly retraining. Continuous Evaluation & Guardrails are essential because user-facing accuracy requires real-time hallucination prevention and automated rollback triggers to maintain strict regulatory compliance.

---

## Deliverable B — Most Critical Failure Point & Mitigation (Day 8 Advanced)

The most critical failure point is an **ungrounded hallucination bypassing the evaluation guardrail**, which directly damages institutional credibility and user trust in mission-critical applications. To mitigate this, the architecture enforces a deterministic citation-grounding validation layer where every generated statement must bind to explicit retrieved chunk IDs verified via Natural Language Inference (NLI) and regex pattern matching before rendering. Any draft failing this strict threshold is intercepted before rendering and replaced by a deterministic, pre-approved safe response: *"I cannot verify this information based on the available data."*

---

## Full System Architecture Diagram

> All node labels with parentheses are wrapped in double quotes as required by GitHub Mermaid.

```mermaid
graph TD
    classDef client fill:#1E293B,stroke:#475569,stroke-width:2px,color:#F8FAFC;
    classDef core fill:#1D4ED8,stroke:#3B82F6,stroke-width:2px,color:#FFFFFF;
    classDef fallback fill:#B91C1C,stroke:#EF4444,stroke-width:2px,color:#FFFFFF,stroke-dasharray: 4 4;
    classDef eval fill:#B45309,stroke:#F59E0B,stroke-width:2px,color:#FFFFFF;
    classDef telemetry fill:#581C87,stroke:#A855F7,stroke-width:2px,color:#FFFFFF;
    classDef cache fill:#047857,stroke:#10B981,stroke-width:2px,color:#FFFFFF;
    classDef note fill:#374151,stroke:#9CA3AF,stroke-width:1px,color:#F9FAFB,stroke-dasharray: 3 3;

    subgraph UserLayer["User Layer"]
        A["User Request"]:::client
        L["Synthesized Output Delivered"]:::client
    end

    subgraph InternalBoundary["Internal System Boundary - Your Infrastructure"]
        B["API Gateway and Semantic Cache"]:::core
        B_Cache{"Semantic Cache Lookup"}:::core
        B_Hit["Deliver Cached Response"]:::cache
        C{"Query Analyzer and Rewriter"}:::core
        D[("Internal Vector DB - Hybrid Retriever")]:::core
        F["Async Queue - Reranker Jobs"]:::core
        F1["Cross-Encoder Reranker Workers"]:::core
        G{"Context Window Monitor"}:::core
        H["Fallback 2 - Map-Reduce Summarization"]:::fallback
        J{"Eval Gate - Faithfulness Guardrail"}:::eval
        K["Fallback 3 - Deterministic Safe Response"]:::fallback
        EvalCollector["NOTE: Eval Collector sits inline post-generation"]:::note
        TelemetryDB[("Telemetry and Metrics DB")]:::telemetry
    end

    subgraph ExternalBoundary["Third-Party Managed Services Boundary"]
        E["Fallback 1 - External Web Search Agent"]:::fallback
        I["Generator LLM API - OpenAI / Anthropic / Vertex AI"]:::core
        AlertSvc["PagerDuty / Opsgenie Incident Response"]:::telemetry
        CICD["CI/CD Orchestrator - GitHub Actions / Argo Rollouts"]:::telemetry
    end

    A --> B
    B --> B_Cache
    B_Cache -- "Cache Hit: Cosine >= 0.95" --> B_Hit --> L
    B_Cache -- "Cache Miss" --> C
    C --> D

    D -- "Failure 1: Low Relevance Score < 0.70" --> E
    D -- "High Relevance >= 0.70" --> F
    E -- "Inject Real-Time Web Context" --> F

    F --> F1 --> G

    G -- "Failure 2: Token Limit Exceeded > 80 pct" --> H
    G -- "Optimal Token Count <= 80 pct" --> I
    H -- "Compressed Context" --> I

    I --> J
    J -- "Failure 3: Hallucination / Low Grounding" --> K --> L
    J -- "Pass: Faithfulness Verified" --> L

    J --> EvalCollector
    EvalCollector -. "Stream Evaluation Telemetry" .-> TelemetryDB

    TelemetryDB -. "Alert: Refusal Spike > 15 pct" .-> AlertSvc
    TelemetryDB -. "Auto-Rollback: Faithfulness Drop < 0.85" .-> CICD
    CICD -. "Rollback to Golden Prompt or Model" .-> I
```

---

## System Boundary Breakdown

| Environment | Component | Role |
|---|---|---|
| **Internal (Yours)** | API Gateway & Semantic Cache | Edge termination, rate limiting, Redis similarity cache |
| **Internal (Yours)** | Query Analyzer & Rewriter | Query decomposition and vector query rewriting |
| **Internal (Yours)** | Vector DB / Hybrid Retriever | Dense ANN + BM25 sparse similarity search |
| **Internal (Yours)** | Async Reranker Queue & Workers | Decoupled GPU worker pool for Cross-Encoder reranking |
| **Internal (Yours)** | Context Window Monitor | Pre-inference token counter blocking context overflow |
| **Internal (Yours)** | Eval Gate & Eval Collector | Inline NLI faithfulness scoring before user delivery |
| **Internal (Yours)** | Telemetry & Metrics DB | Prometheus / ClickHouse tracking refusal & quality drift |
| **Third-Party (External)** | External Web Search Agent | SerpAPI / Tavily fetching real-time fallback context |
| **Third-Party (External)** | Generator LLM API | Hosted endpoint: OpenAI / Anthropic Claude / Vertex AI |
| **Third-Party (External)** | PagerDuty / Opsgenie | On-call incident alerting for anomaly spikes |
| **Third-Party (External)** | CI/CD Orchestrator | GitHub Actions / Argo Rollouts managing rollback webhooks |

---

## 3 Identified Failure Points with Explicit Fallback Behaviors

### Failure Point 1 — Retriever Failure (Low Relevance / Empty Set)
- **Detection:** Vector similarity scores below `0.70`, or empty result set.
- **Impact without fallback:** Model hallucinates with high confidence from irrelevant chunks.
- **Fallback:** Route query to an **External Web Search Agent** (SerpAPI / Tavily). Dynamically embed fetched web documents and inject into the reranker queue — session continues without interruption.

### Failure Point 2 — Context Window Overflow
- **Detection:** Accumulated tokens (system prompt + history + chunks) exceed `80%` of model context limit.
- **Impact without fallback:** HTTP 400 API crash or silent truncation of critical instructions.
- **Fallback:** Middleware triggers **Map-Reduce Summarization** via a fast lightweight model (Gemini Flash / Claude Haiku). Chunks are compressed 60–75% through Map (per-chunk propositions) then Reduce (consolidated summary), preserving chunk IDs.

### Failure Point 3 — Eval Gate Rejection (Hallucination)
- **Detection:** NLI Faithfulness Judge scores the draft below `0.85` — ungrounded claims detected.
- **Impact without fallback:** False or fabricated information reaches the user, destroying trust.
- **Fallback:** Draft is discarded. User receives the deterministic safe response: *"I cannot verify this information based on the available data."* Failed pair is queued for RLHF analysis.

---

## Scaling Consideration — The Reranker Bottleneck

**First bottleneck: Cross-Encoder Reranker**

Unlike Bi-Encoders (pre-computed vectors, sub-ms ANN lookup), a Cross-Encoder passes query + document pairs through full bidirectional cross-attention — O(K * L^2) complexity. At 500+ QPS with 50 candidates per request, GPU VRAM saturates and the async event loop blocks.

**Mitigation:**
1. **Decouple via Async Queue** — Offload reranking to a Redis/Celery worker cluster with KEDA autoscaling on GPU nodes based on queue depth.
2. **Semantic Front-End Cache** — Redis / GPTCache in front of the API Gateway. Queries with cosine similarity >= 0.95 against cached queries return in < 15ms with zero GPU cost.
3. **Two-Stage Cascade** — BM25 pre-filters 100 candidates to top 15 before the Cross-Encoder runs.

---

## Eval Integration — How Metrics Trigger Alerts and Rollbacks

**Where the Eval Collector sits:** Inline between the Generator LLM and the final output gate. Every response is scored *before* reaching the user.

**Metrics tracked (async telemetry stream to Prometheus / ClickHouse):**
- Faithfulness Score (NLI-grounded claim ratio)
- Answer Relevance (cosine similarity of query vs. response)
- Refusal Rate (proportion of safe-response fallbacks triggered)
- P99 Latency and Token Consumption

**Alert Trigger — Goodhart's Law Detection:**
- Condition: Refusal Rate moving average spikes > 15% over a 15-minute window while Faithfulness stays near 1.0 (system is being over-conservative).
- Action: PagerDuty P1 incident fires, on-call engineers recalibrate guardrail thresholds.

**Auto-Rollback Trigger — Quality Regression Guard:**
- Condition: Rolling 100-request Faithfulness average drops below `0.85`, or hallucination rate exceeds `5%`, following a deployment.
- Action: Telemetry analyzer fires an authenticated webhook to the CI/CD orchestrator (GitHub Actions / Argo Rollouts). Zero-downtime rollback to the last Golden Model / Prompt Template version. Failed run is auto-exported to an evaluation regression dataset.
