flowchart TD
    subgraph ADMIN["1. Admin Configuration"]
        A["Define mandatory gates, weights,<br/>thresholds and scoring methods"]
        DB[("Versioned Criteria Database")]
        A --> DB
    end

    subgraph INPUT["2. Upload and Preprocessing (Plain Code)"]
        U1["Reviewer selects category<br/>and uploads PDFs"]
        U2["Validate files and extract PDF text"]
        U3["Store source text, page references<br/>and SHA-256 file hash"]
        U1 --> U2 --> U3
    end

    subgraph AI["3. AI Fact Extraction Only"]
        X1["Extract raw proposal facts<br/>into structured JSON"]
        X2["Include verbatim evidence quotes,<br/>missing fields and security flags"]
        X1 --> X2
    end

    subgraph ENGINE["4. Deterministic Rule & Scoring Engine (Plain Code)"]
        V["Validate extracted facts<br/>against source evidence"]
        G{"Mandatory gates pass?"}
        I["Ineligible<br/>rank: null, score: null"]
        Q{"Critical evidence unresolved<br/>or security review required?"}
        F["Flagged<br/>rank: null, score: null"]
        S["Calculate weighted scores<br/>using admin configuration"]
        R["Ranked<br/>Sort by unrounded total, assign ranks<br/>and round display scores (0-100)"]

        V --> G
        G -->|No| I
        G -->|Yes| Q
        Q -->|Yes| F
        Q -->|No| S --> R
    end

    subgraph HUMAN["5. Human Review & Final Award"]
        UI["Reviewer dashboard<br/>Statuses, rankings, evidence<br/>and score breakdown"]
        H["Review results and resolve flags<br/>Record reasons for overrides"]
        LOG[("Append-only Audit Log")]
        D{"Flag resolved or<br/>extracted data corrected?"}
        FINAL["Authorized officer makes<br/>final award decision"]

        UI --> H --> LOG --> D
        D -->|Yes| V
        D -->|No / Review complete| FINAL
    end

    DB -.->|Category options| U1
    DB -.->|Required fields| X1
    DB -.->|Mandatory rules| G
    DB -.->|Weights and scoring methods| S

    U3 -->|Untrusted text| X1
    X2 --> V

    I --> UI
    F --> UI
    R --> UI