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
| Layer | Technology | Purpose |
|---|---|---|
| Enrichment engine & API | **Python 3.11 + FastAPI** | Ingests observables, fans out to TI providers, computes composite scores |
| Upstream integrations | **Python (httpx, async)** | Adapters for each TI/reputation/sandbox provider |
| SOC automation | **Cortex XSOAR content pack (Python)** | Reputation commands + an auto-triage playbook |
| Investigation console | **React + TypeScript** | Analyst-facing UI for searching indicators and reading results |

This maps directly onto the stack requested for the project:

- ✅ **Primary scripting — Python**: all enrichment, parsing, and TI/SIEM integration logic lives in `backend/app/`.
- ✅ **API automation — Python httpx**: `backend/app/utils/http_client.py` is the shared async client (retry, backoff, concurrency limiting) used by every provider adapter.
- ✅ **Automation/orchestration — Cortex XSOAR + Python**: `xsoar/IOCScopePack/` contains the integration (`IOCScope.py`/`.yml`) and an enrichment→triage playbook.
- ✅ **Backend UI — FastAPI**: `backend/app/main.py` exposes the enrichment engine as a documented REST API (`/docs`, `/redoc`).
- ✅ **SOC dashboard — React + TypeScript**: `frontend/` is the investigation console that calls the FastAPI backend and renders composite scores, artifacts, and provider-level detail.

# 4. Indicator coverage & data sources
| Indicator Type | Providers Queried | What's Extracted |
|---|---|---|
| IPv4 / IPv6 | AbuseIPDB, Shodan, RDAP, VirusTotal, Trend Micro ERS, BlacklistMaster | ISP/ASN, geolocation, abuse confidence score, open ports, CVEs, VPN/Tor exit node detection |
| File Hashes (MD5/SHA1/SHA256) | VirusTotal, ANY.RUN, Hybrid Analysis, MalwareBazaar | Sandbox execution behavior, MITRE ATT&CK mapping, malware family, first/last seen |
| Domains / URLs | RDAP/WHOIS, VirusTotal, AlienVault OTX, URLScan.io, Trend Micro ERS, BlacklistMaster | Registrar data, domain age (< 30 days = elevated risk), category/reputation, historical scans |
| Emails | EmailRep.io, Have I Been Pwned | Deliverability, breach exposure, credential leak linkage, disposable-provider status |

Every provider adapter degrades gracefully: a missing API key or a timeout produces `ok: false` on that one source rather than failing the whole enrichment.

# 5. Configuration
All secrets are environment-driven — see [`backend/.env.example`](backend/.env.example) for the full list. Every provider key is optional; unset keys cause that provider to return `ok: false` with `error: "missing_api_key"` instead of breaking the request.

| Variable | Purpose |
|---|---|
| `API_AUTH_TOKEN` | Bearer token required on all `/api/v1/*` routes (unset = auth disabled, dev only) |
| `REDIS_URL` | Enrichment result cache backend |
| `VIRUSTOTAL_API_KEY`, `ABUSEIPDB_API_KEY`, `OTX_API_KEY`, `SHODAN_API_KEY`, `URLSCAN_API_KEY`, `EMAILREP_API_KEY`, `HIBP_API_KEY`, `MALWAREBAZAAR_API_KEY`, `ANYRUN_API_KEY`, `HYBRID_ANALYSIS_API_KEY`, `TRENDMICRO_ERS_API_KEY`, `BLACKLISTMASTER_API_KEY` | Provider credentials |
| `RDAP_BASE_URL` | RDAP bootstrap resolver (no key required, defaults to `https://rdap.org`) |

# 6. API reference (summary)

Full interactive reference is auto-generated by FastAPI at `/docs` once the backend is running.

```
POST /api/v1/enrich
Body: { "indicator": "1.2.3.4", "indicator_type": "ipv4" (optional), "force_refresh": false }
→ EnrichmentResponse { composite, key_artifacts, whois, threat_actors, malware, provider_findings, ... }

GET  /api/v1/enrich/{indicator}?force_refresh=false
GET  /api/v1/health
```

# 7. Security considerations

- Keep all provider API keys server-side (`backend/.env`); never expose them to the frontend bundle.
- Set `API_AUTH_TOKEN` before running anywhere beyond localhost.
- Lock `allow_origins` in `backend/app/main.py`'s CORS middleware to your real console domain in production.
- In XSOAR, the integration's API token parameter uses the credentials type, so it's stored in XSOAR's vault rather than plaintext.
