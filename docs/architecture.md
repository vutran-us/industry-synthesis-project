# Architecture: Municipal SOC Triage

```mermaid
flowchart TD
    A["Flow batch<br/>(UNSW-NB15 records)"] --> B["validate_batch<br/>schema, ranges, known categories<br/><b>P1</b>"]
    B -- "malformed or unseen category" --> H1["Hold batch:<br/>validation_review"]
    B -- "clean" --> C["drift_check<br/>KS + total variation vs fit reference<br/><b>P2</b>"]
    C -- "distance ≥ 0.30 or batch < 200" --> H2["Halt: drift_halt<br/>alert analyst"]
    C -- "pass" --> D["score_flows<br/>Random Forest, threshold 0.65<br/><b>P3</b>"]
    S["Service flag (DNS, POP3)<br/>context only, not a vote<br/><b>P2</b>"] -.-> D
    D -- "score < 0.65" --> N["no_alert"]
    D -- "score ≥ 0.65" --> R["review_candidate"]
    R --> G["Bounded LLM agent (opt-in)<br/>one read-only tool: read_flow_evidence<br/><b>P6</b>"]
    G --> K["check_ticket<br/>evidence + claim guardrail<br/><b>P5</b>"]
    K -- "violation" --> X["draft_rejected"]
    K -- "clean" --> P["pending_approval"]
    P --> HU{{"Human analyst<br/>approval gate"}}
    HU -- "approved" --> T["approved_simulation<br/>(no external action)"]

    B & C & D & G & K & HU -.-> L[("Audit log<br/>outputs/triage.jsonl<br/>partition + source ID")]

    classDef human fill:#fde68a,stroke:#b45309,color:#000;
    classDef stop fill:#fecaca,stroke:#b91c1c,color:#000;
    class HU human;
    class H1,H2,X stop;
```

## System boundaries
| The system does | The system does not |
|---|---|
| Validate, drift-check, score and rank flows | Block traffic, change firewall rules, or take personnel action |
| Draft a review ticket from flow ID and score only | Confirm incidents or name CVEs, hosts or users |
| Log every stage for reconstruction | Release anything without explicit analyst approval |

## Prior-project integration
- **P1:** validation gate and reproducibility (checksums, seed, pinned environment)
- **P2:** chi-square association test, service-rate flag, drift gate
- **P3:** Random Forest, validation-selected threshold, subgroup error audit
- **P5:** guardrail motivated by fabricated generative output
- **P6:** bounded tool-calling agent with a stage cap and human approval
