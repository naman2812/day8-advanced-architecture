# Day 8 Assignment Submission Cheat-Sheet

---

## Deliverable A: Pillar Choices (Day 8 Base - 2 to 3 sentences)

For this enterprise knowledge assistant, we selected **Retrieval-Augmented Generation (RAG)** and **Continuous Evaluation & Guardrails** as our two primary pillars. RAG is required because enterprise documentation updates dynamically and demands deterministic grounding without costly retraining. Continuous Evaluation & Guardrails are essential because user-facing accuracy requires real-time hallucination prevention and automated rollback triggers to maintain strict regulatory compliance.

---

## Deliverable B: Critical Failure Point & Mitigation (Day 8 Advanced - 2 to 3 sentences)

The most critical failure point is an **ungrounded hallucination bypassing the evaluation guardrail**, which directly damages institutional credibility and user trust in mission-critical applications. To mitigate this, the architecture enforces a deterministic citation-grounding validation layer where every generated statement must bind to explicit retrieved chunk IDs verified via Natural Language Inference (NLI) and regex pattern matching before rendering. Any draft failing this strict threshold is intercepted before rendering and replaced by a deterministic, pre-approved safe response: *"I cannot verify this information based on the available data."*

---

## GitHub Submission Link

```
https://github.com/naman2812/day8-advanced-architecture
```
