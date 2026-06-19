```mermaid
graph TD
    subgraph "Public Internet (Untrusted)"
        NGC[("NVIDIA NGC Public Registry<br/>(nvcr.io)")]
    end

    subgraph "Enterprise Security Zone (DMZ)"
        Ingest[("📥 Ingestion Server / CI Pipeline")]
        Scan{("🛡️ Vulnerability Scan<br/>(Trivy/Clair/Snyk)")}
        Policy[("🚦 Policy Gate<br/>(Fail if Critical CVEs)")]
    end

    subgraph "Internal Trusted Network (DGX Cluster)"
        Harbor[("🏢 Private Registry<br/>(Harbor/NGC Private)<br/>Only 'Verified' Images")]
        DGX[("💻 DGX Spark Nodes<br/>(Developers)")]
        K8s[("🛑 Admission Controller<br/>(Kyverno/OPA)<br/>Blocks External Pulls")]
    end

    %% Flow
    NGC -->|1. Pull Base Image| Ingest
    Ingest -->|2. Submit for Scan| Scan
    Scan -->|3. Clean?| Policy
    Scan -->|3. Vulnerable?| Reject[("❌ Reject & Alert")]
    Policy -->|4. Push Verified Image| Harbor
    Policy -->|4. Block| Reject
    
    DGX -->|5. Pull ONLY from Internal| Harbor
    K8s -.->|Enforce: No nvcr.io| DGX
    
    %% Styling
    style NGC fill:#f9f,stroke:#333,stroke-width:2px
    style Harbor fill:#9f9,stroke:#333,stroke-width:2px
    style DGX fill:#bbf,stroke:#333,stroke-width:2px
    style Scan fill:#ff9,stroke:#333,stroke-width:2px
    style Reject fill:#f99,stroke:#333,stroke-width:2px   
