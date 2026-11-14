# Architecture: Municipal SOC Triage

```mermaid
flowchart LR
    A["Flow batch<br/>UNSW-NB15"] --> B["validate_batch<br/>schema, ranges,<br/>known categories<br/><b>P1</b>"]
    B -- "malformed /<br/>unseen category" --> H1["Hold batch:<br/>validation_review"]
    B -- "clean" --> C["drift_check<br/>KS + total variation<br/><b>P2</b>"]
    C -- "drift ≥ 0.30 or<br/>batch < 200" --> H2["Halt: drift_halt"]
    C -- "pass" --> D["score_flows<br/>Random Forest @ 0.65<br/><b>P3</b>"]
    S["Service flag<br/>DNS, POP3<br/>context only<br/><b>P2</b>"] -.-> D
    D -- "< 0.65" --> N["no_alert"]
    D -- "≥ 0.65" --> G["Bounded LLM agent<br/>1 read-only tool<br/><b>P6</b>"]
    G --> K["check_ticket<br/>guardrail<br/><b>P5</b>"]
    K -- "violation" --> X["draft_rejected"]
    K -- "clean" --> HU{{"Analyst<br/>approval gate"}}
    HU -- "approved" --> T["approved_simulation<br/>no external action"]
    L[("Audit log: every stage<br/>outputs/triage.jsonl")]
    classDef human fill:#fde68a,stroke:#b45309,color:#000;
    classDef stop fill:#fecaca,stroke:#b91c1c,color:#000;
    classDef log fill:#e5e7eb,stroke:#6b7280,color:#000;
    class HU human;
    class H1,H2,X stop;
    class L log;
```

![Architecture diagram](architecture.png)

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
