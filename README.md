# IOCify
IOC enrichment &amp; threat intelligence pipeline for modern SOC

# 2. Idea
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
                   ┌────────────────────────────┼────────────────────────────┐
                   ▼                                                         ▼
            ┌──────────────────────────────────┐          ┌──────────────────────────────────┐
            |    React + TypeScript Dashboard  |          |    Cortex XSOAR Content Pack     |
            |    (Graph View & Pivot Console)  |          |    (Auto-Triage & Containment)   |
            └──────────────────────────────────┘          └──────────────────────────────────┘

# 3. Problem statement
Modern SOC triage suffers from context fragmentation: analysts spend a large share of mean time to respond (MTTR) manually querying disparate TI platforms, scraping WHOIS records, checking reputation scores, and correlating malware families by hand — for every single observable, on every single alert. IOC Scope collapses that into one API call and one score.

# 4. What this repository contains
Layer	Technology	Purpose
Enrichment engine & API	Python 3.11 + FastAPI	Ingests observables, fans out to TI providers, computes composite scores
Upstream integrations	Python (httpx, async)	Adapters for each TI/reputation/sandbox provider
SOC automation	Cortex XSOAR content pack (Python)	Reputation commands + an auto-triage playbook
Investigation console	React + TypeScript	Analyst-facing UI for searching indicators and reading results

This maps directly onto the stack requested for the project:

✅ Primary scripting — Python: all enrichment, parsing, and TI/SIEM integration logic lives in backend/app/.
✅ API automation — Python httpx: backend/app/utils/http_client.py is the shared async client (retry, backoff, concurrency limiting) used by every provider adapter.
✅ Automation/orchestration — Cortex XSOAR + Python: xsoar/IOCScopePack/ contains the integration (IOCScope.py/.yml) and an enrichment→triage playbook.
✅ Backend UI — FastAPI: backend/app/main.py exposes the enrichment engine as a documented REST API (/docs, /redoc).
✅ SOC dashboard — React + TypeScript: frontend/ is the investigation console that calls the FastAPI backend and renders composite scores, artifacts, and provider-level detail.

# 4. Indicator coverage & data sources
| Indicator Type | Providers Queried             | What's Extracted                                                                     |
|----------------|-------------------------------|--------------------------------------------------------------------------------------|
| 1              | IPv4 / IPv6                   | AbuseIPDB, Shodan, RDAP, VirusTotal, Trend Micro ERS, BlacklistMaster                |
| 2              | File Hashes (MD5/SHA1/SHA256) | VirusTotal, ANY.RUN, Hybrid Analysis, MalwareBazaar                                  |
| 3              | Domains / URLs                | RDAP/WHOIS, VirusTotal, AlienVault OTX, URLScan.io, Trend Micro ERS, BlacklistMaster |
| 4              | Emails                        | EmailRep.io, Have I Been Pwned                                                       |

Every provider adapter degrades gracefully: a missing API key or a timeout produces 'ok: false' on that one source rather than failing the whole enrichment.
