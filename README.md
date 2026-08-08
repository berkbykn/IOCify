# IOCify
IOC enrichment &amp; threat intelligence pipeline for modern SOC

# Idea

REST APIs, an XSOAR playbook integration, and a React-based SOC investigation console.

                  ┌──────────────────────────────────────────────────────────┐
                  │                      Ingestion                           │
                  │  (SIEM Alerts / React UI / API / XSOAR Playbook Trigger) │
                  └─────────────────────────────┬────────────────────────────┘
                                                │
                                                ▼
                  ┌──────────────────────────────────────────────────────────┐
                  │                   FastAPI Backend Gateway                │
                  │          (Input Validation, Normalization, Auth)         │
                  └─────────────────────────────┬────────────────────────────┘
                                                │
                   ┌────────────────────────────┼────────────────────────────┐
                   ▼                            ▼                            ▼
        ┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
        │  Network Providers  │      │  Reputation Engine  │      │ Sandbox & Malware   │
        │ • AbuseIPDB         │      │ • VirusTotal (v3)   │      │ • ANY.RUN           │
        │ • RDAP / WHOIS      │      │ • AlienVault OTX    │      │ • Hybrid Analysis   │
        │ • Shodan / Censys   │      │ • Trend Micro ERS   │      │ • MalwareBazaar     │
        └──────────┬──────────┘      └──────────┬──────────┘      └──────────┬──────────┘
                   │                            │                            │
                   └────────────────────────────┼────────────────────────────┘
                                                │ Async Fetch (httpx)
                                                ▼
                  ┌──────────────────────────────────────────────────────────┐
                  │                  Aggregation & Normalization             │
                  │       • Deduplication & Canonical Taxonomy Mapping       │
                  │       • Composite Risk Score Engine (0 - 100)            │
                  │       • Threat Actor & Campaign Attributions             │
                  └─────────────────────────────┬────────────────────────────┘
                                                │
                   ┌────────────────────────────┴────────────────────────────┐
                   ▼                                                         ▼
  ┌─────────────────────────────────┐                       ┌─────────────────────────────────┐
  │   React + TypeScript Dashboard  │                       │   Cortex XSOAR Content Pack     │
  │  (Graph View & Pivot Console)   │                       │  (Auto-Triage & Containment)    │
  └─────────────────────────────────┘                       └─────────────────────────────────┘
