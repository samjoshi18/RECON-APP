# SecRon — Recon & Vulnerability Detection Platform

**SecRon** is a full-stack reconnaissance and security assessment platform that combines
classical attack-surface discovery (subdomains, ports, CVEs, exposed buckets, etc.) with
**AI/LLM vulnerability detection** (prompt-injection testing and automated LLM red-teaming
via [garak](https://github.com/leondz/garak)). A React SPA frontend talks to an Express API
that orchestrates a suite of offensive-security CLI tools (ProjectDiscovery suite, nmap,
subjack, and more) and third-party threat-intelligence APIs.

> **Authorization notice:** This tool performs active scanning (port scans, vulnerability
> probing, secret grepping, cloud-bucket enumeration, and LLM jailbreak testing). Run it
> **only against systems and targets you own or are explicitly authorized to assess.**

---

## Table of Contents

1. [Overview](#overview)
2. [Key Capabilities](#key-capabilities)
3. [Architecture](#architecture)
4. [Technology Stack](#technology-stack)
5. [Backend API Reference](#backend-api-reference)
6. [AI / LLM Vulnerability Detection](#ai--llm-vulnerability-detection)
7. [External Tools & Integrations](#external-tools--integrations)
8. [Frontend](#frontend)
9. [Project Structure](#project-structure)
10. [Configuration](#configuration)
11. [Deployment](#deployment)
12. [Getting Started (Local Development)](#getting-started-local-development)
13. [Security Notes](#security-notes)

---

## Overview

SecRon is built around a **target-driven, section-based workflow**. You enter a domain
(e.g. `example.com`), and the sidebar exposes independent recon/vuln modules. Each module
calls a dedicated backend endpoint, returns normalized rows, and caches results per target
so you can switch tabs without re-scanning. Results can be aggregated into a generated
report.

The platform deliberately blends **two offensive disciplines**:

- **Traditional recon / vuln scanning** — subdomain enumeration, HTTP probing, port scanning,
  CVE extraction, SSL/TLS inspection, security-header auditing, DNS hygiene, reputation
  lookups, open cloud-storage detection, subdomain-takeover detection, and template-based
  vulnerability scanning (nuclei).
- **AI vulnerability detection** — automated prompt-injection red-teaming of chatbot HTTP
  endpoints and full LLM probing with garak (NVIDIA/Vijil's LLM vulnerability scanner).

---

## Key Capabilities

| Module | What it does | Backing tech |
|---|---|---|
| **Subdomain Discovery** | Enumerates subdomains and probes live HTTP hosts, status codes, titles, and detected technologies | `subfinder` + `httpx` |
| **Port Scan** | TCP port scanning with web-port probing (naabu primary, nmap fallback) | `naabu`, `nmap`, `httpx` |
| **Host Discovery** | Subdomain-to-IP and IP-to-host mapping with live-host confidence scoring | `subfinder`, `httpx` |
| **URLs** | Historical URL discovery from the Wayback Machine per subdomain | Wayback CDX API |
| **Headers Check** | Audits security response headers (HSTS, CSP, X-Frame-Options, etc.) across hosts | Node `fetch` |
| **DNS Hygiene** | A/AAAA/MX/NS/CAA + SPF & DMARC validation | Node `dns` |
| **Nmap** | Service/version detection + CVE extraction via the `vulners` nmap script | `nmap` (`vulners`) |
| **SSL/TLS Check** | Certificate validity, issuer, expiry, signature algorithm, key size, SAN | Node `tls` |
| **Reputation** | Domain reputation & safe-browsing threat matches | VirusTotal v3, Google Safe Browsing v4 |
| **Open Buckets** | Detects publicly listable AWS S3 / GCS / Azure Blob storage | HTTP + `aws` CLI |
| **Nuclei Scan** | Template-based vulnerability scanning, streamed as NDJSON | `nuclei` |
| **Takeover Check** | Subdomain-takeover detection | `subjack` |
| **Breach Check** | Credential-breach lookup by email | LeakCheck API v2 |
| **Google Dorking** | Generates & executes categorized Google dorks (exposed files, configs, keys...) | SerpAPI |
| **GitHub Dorking** | Searches GitHub for target-leaked secrets/code | GitHub Search API |
| **JS Scan** | Crawls JS assets and greps for leaked secrets/keywords | `katana`, `curl` |
| **Prompt Injection** | Automated prompt-injection red-team of a chatbot endpoint | custom controller (OpenAI/Anthropic/Ollama) |
| **Garak LLM Scan** | Full LLM vulnerability probing of an HTTP chat endpoint | garak 0.13.3 |

Additional product features:

- **Fast / Full modes** — each endpoint tunes concurrency, timeouts, depth, and result caps.
- **Per-section caching** — results are snapshotted per target so tab switches are instant.
- **Streaming** — Nuclei, takeover, and dorking results stream as NDJSON for live updates.
- **Report generation** — aggregates the current target's findings into a text report.
- **Sorting & filtering** — status-code sorting, host-discovery confidence sorting, dork hit filtering.

---

## Architecture

```
+-------------------+        REST / NDJSON         +----------------------------+
|   React SPA       |  ------------------------->  |  Express 5 API (Node/TS)   |
|  (Vite + TS)      |  <-------------------------  |  backend/src/server.ts     |
|  frontend/        |                              |                            |
+-------------------+                              +----+---------------+-------+
                                                        |               |
                                          spawn() /     |        fetch()|
                                          child_process |               |
                                          v             v               v
                                  +----------------+ +----------+ +-------------+
                                  | Recon CLI tools| | Threat   | | AI scanners |
                                  | subfinder,httpx| | intel    | | garak, PI   |
                                  | naabu,nmap,... | | APIs     | | runner      |
                                  +----------------+ +----------+ +-------------+
```

- The **frontend** is a static SPA (deployed to Firebase Hosting). It calls the backend via
  `VITE_API_BASE`.
- The **backend** is a single Express app that validates each request with **Zod**, applies
  **Helmet**, **CORS allowlist**, and **rate limiting**, then orchestrates CLI tools via
  `child_process.spawn` and calls external APIs via `fetch`.
- Streaming endpoints set `Content-Type: application/x-ndjson` and write JSON lines as
  findings arrive (Nuclei, takeover, dorking).

---

## Technology Stack

### Backend
- **Runtime:** Node.js (TypeScript, strict)
- **Web framework:** Express 5
- **Validation:** Zod v4 (strict hostname/email schemas)
- **Security middleware:** Helmet, CORS allowlist, express-rate-limit
- **Process orchestration:** `child_process.spawn` (with timeouts, SIGTERM kills, output caps)
- **Networking:** native `tls`, `dns`, and `fetch` for cert/DNS/reputation checks
- **Cloud Functions:** Firebase Functions (v2, scaffolded; `maxInstances: 10`)
- **Dependency security:** Snyk (test/monitor/container scripts)

### Frontend
- **Framework:** React 19
- **Build tooling:** Vite 7, TypeScript 5.8
- **Linting:** ESLint 9 + typescript-eslint, react-hooks, react-refresh plugins
- **Hosting:** Firebase Hosting
- **Analytics:** Firebase Analytics
- **Dependency security:** Snyk (test/monitor scripts)

### Vendored AI tooling
- **garak 0.13.3** (vendored under `backend/tools/garak_py/`) — NVIDIA/Vijil LLM
  vulnerability scanner. Apache-2.0 licensed. Provides probes (DAN, promptinject,
  latent-injection, leakreplay, encoding, lmrc, malwaregen, packagehallucination, and many
  more), detectors, generators (including a REST generator for HTTP endpoints), and
  AVID report digesting.

---

## Backend API Reference

All endpoints live under `/api`. Request bodies are validated with Zod; `target` must match
a strict hostname regex. Most endpoints accept `{ target, mode? }` where `mode` is
`fast` (default) or `full`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | Liveness/healthcheck (used by Railway) |
| GET | `/api/debug/env` | Shows whether VT/GSB API keys are configured (prefix only) |
| POST | `/api/recon` | Subdomain discovery + httpx probing |
| POST | `/api/ports` | Port scan (naabu, nmap fallback) + web-port probing |
| POST | `/api/host-discovery` | Subdomain/IP cross-mapping with confidence scoring |
| POST | `/api/nmap` | nmap `-sV` + `vulners` CVE extraction |
| POST | `/api/js-scan` | katana crawl + JS secret grep |
| POST | `/api/ssl-check` | TLS certificate inspection across subdomains |
| POST | `/api/breach-check` | Email breach lookup (`{ email }`) |
| POST | `/api/urls-scan` | Wayback Machine historical URLs |
| POST | `/api/headers-check` | Security-header audit |
| POST | `/api/dns-hygiene` | DNS record + SPF/DMARC validation |
| POST | `/api/reputation` | VirusTotal + Google Safe Browsing |
| POST | `/api/cloud-buckets` | S3 / GCS / Azure Blob public-listing detection |
| POST | `/api/nuclei` | Nuclei scan (streams NDJSON findings) |
| POST | `/api/takeover` | Subjack subdomain-takeover detection (streams NDJSON) |
| POST | `/api/dorking` | Google dork generation/execution (streams NDJSON) |

**Server hardening (applied globally):**
- Helmet (CSP disabled for static SPA assets; COEP disabled).
- 64 KB JSON body limit.
- CORS allowlist from `ALLOWED_ORIGINS` + Firebase defaults; methods restricted to
  `GET/POST/OPTIONS`.
- Rate limit: 60 requests / minute / IP on recon endpoints.
- Input validation: strict hostname regex; IP-address and error-string filtering on
  subdomain output; output size caps (5 MB) and per-tool timeouts with SIGTERM.

---

## AI / LLM Vulnerability Detection

SecRon's differentiator is **purpose-built AI red-teaming** alongside traditional recon.

### 1. Prompt Injection module (`/api/prompt-injection`)

Tests a chatbot HTTP endpoint for prompt-injection resistance. From the frontend you
configure:
- **Target URL** of the chat API (e.g. an OpenAI-compatible `/api/chat`).
- A **YAML HTTP config** describing method, headers, and JSON body with
  `"{PAYLOAD_POSITION}"` / `"{ANSWER_POSITION}"` markers.
- A **controller model** (OpenAI / Anthropic / Ollama) used as the attack/judge.
- **Iterations** for multi-round attacks.

Each rule returns `{ ruleName, type, severity, passed, passRate, response, evaluation }`
so you can see per-rule pass/fail rates and the model's actual responses.

### 2. Garak LLM Scan (`/api/garak-scan`)

Runs the vendored **garak 0.13.3** LLM vulnerability scanner against any HTTP chat
endpoint using garak's `rest.RestGenerator`. The frontend lets you configure:
- Target URI, HTTP method, headers (JSON).
- Request template — JSON body **or** multipart form-data, with a `$INPUT` placeholder
  marking where the probe prompt is injected.
- Response JSON field path (e.g. `$.response`) to extract the model reply.
- Request timeout, SSL verification toggle.
- Garak **probes** (default `promptinject`), **generations**, and extra CLI args.

Results include the probe list, exit code, stdout/stderr, JSONL/HTML report paths, and a
digest. The UI surfaces a Cloudflare caveat: endpoints behind advanced bot protection
generally cannot be scanned by garak's Python HTTP client due to TLS-fingerprint checks.

**garak probe families included** (vendored data + probe modules): DAN jailbreaks,
promptinject, latent injection, leakreplay (training-data extraction), encoding attacks,
LMRC, malware generation, package hallucination, promptinject, GCG/AutoDAN/BEAST suffixes,
realtoxicityprompts, donotanswer, web/XSS injection, and more. Detectors include
LLM-as-judge, known-bad-signatures, perspective toxicity, leakreplay, and others.

---

## External Tools & Integrations

**CLI tools (must be on `PATH` — backend auto-extends PATH with common dirs incl. `/root/go/bin`):**
- ProjectDiscovery: `subfinder`, `httpx`, `naabu`, `katana`, `nuclei`
- `nmap` (with `vulners` script), `subjack`, `curl`, `aws` CLI (optional S3 probe)

**Third-party APIs (configure via environment variables):**
- **VirusTotal** (`VIRUSTOTAL_API_KEY`) — domain reputation in `/api/reputation`.
- **Google Safe Browsing** (`GSB_API_KEY`) — threat matches in `/api/reputation`.
- **LeakCheck** (`LEAKCHECK_API_KEY`) — breach data in `/api/breach-check`.
- **SerpAPI** (`SERPAPI_KEY`) — live Google dork results in `/api/dorking` (without it,
  dorks are generated with direct Google URLs but no result counts).
- **GitHub** search — used by GitHub dorking.
- **Wayback Machine** CDX API — historical URLs.

---

## Frontend

The SPA is a single-page workspace (`frontend/src/App.tsx`). Flow:

1. **Home screen** — enter a domain/IP/wildcard target.
2. **Workspace** — left sidebar of modules; main panel renders normalized result tables.
3. Each module caches its snapshot per target (`localStorage`) and updates the URL
   (`?target=...&section=...`) for shareable/deep-linkable state.
4. **Streaming** modules (Nuclei, takeover, dorking) render rows incrementally.
5. **Report** view aggregates the current target's cached findings into a text report.

Key supporting modules:
- `SecurityConfig.ts` — production security policy & compliance checklist.
- `DeploymentValidator.ts` — pre-deploy validation gate (see Security Notes).
- `firebase.ts` — Firebase app initialization (Hosting/Analytics).

---

## Project Structure

```
recon-app/
├── backend/
│   ├── src/
│   │   ├── server.ts          # Express API: all /api endpoints + tool orchestration
│   │   └── firebase.ts        # Firebase app init
│   ├── functions/             # Firebase Cloud Functions (scaffold)
│   ├── tools/garak_py/        # Vendored garak 0.13.3 LLM vuln scanner
│   ├── recon-output/          # Sample discovery-bundle outputs (*.txt)
│   ├── .wayback-cache/        # Cached Wayback lookups (per-domain JSON)
│   ├── railway.json           # Railway deploy config (healthcheck on /api/health)
│   ├── deploy.sh              # Deployment guidance helper
│   ├── env.example            # Environment variable template
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── App.tsx            # Main SPA: all modules, tables, streaming, reporting
│   │   ├── DeploymentValidator.ts
│   │   ├── SecurityConfig.ts
│   │   ├── main.tsx
│   │   └── *.css
│   └── package.json           # React 19, Vite 7, ESLint 9, Snyk
├── config/
│   └── deployment-context.json
├── .security-pipeline.json    # Pre-review security scan pipeline (MCP-backed)
├── .coderabbit.yaml           # CodeRabbit review config
└── README.md
```

---

## Configuration

### Backend environment (`backend/.env`, see `backend/env.example`)

| Variable | Purpose |
|---|---|
| `PORT` | Server port (Railway sets this automatically; default `4000`) |
| `ALLOWED_ORIGINS` | Comma-separated CORS allowlist (frontend origins) |
| `VIRUSTOTAL_API_KEY` | VirusTotal reputation |
| `GSB_API_KEY` | Google Safe Browsing |
| `LEAKCHECK_API_KEY` | Breach checks |
| `SERPAPI_KEY` | Live Google dork results |

### Frontend environment (`frontend/.env`)

| Variable | Purpose |
|---|---|
| `VITE_API_BASE` | Backend origin, e.g. `http://localhost:4000` (required for the SPA to reach the API) |

### CLI tools

Install the ProjectDiscovery suite, nmap (with the `vulners` script), subjack, and
optionally the `aws` CLI. The backend merges `/usr/local/bin`, `/usr/bin`, `/bin`, and
`/root/go/bin` into `PATH` at startup.

---

## Deployment

- **Backend:** Railway (primary; see `railway.json` — Dockerfile builder, healthcheck on
  `/api/health`, restart on failure). Render and Vercel are also documented in `deploy.sh`.
- **Frontend:** Firebase Hosting (project `techificial-38f35`). Build with `npm run build`
  and deploy via the Firebase CLI.
- Set `ALLOWED_ORIGINS` to the deployed frontend origin(s) on the backend, and
  `VITE_API_BASE` to the deployed backend origin on the frontend.

---

## Getting Started (Local Development)

```bash
# 1. Backend
cd backend
cp env.example .env        # fill in API keys + ALLOWED_ORIGINS
npm install
npm run dev                # ts-node + nodemon on :4000

# 2. Frontend (separate terminal)
cd frontend
echo "VITE_API_BASE=http://localhost:4000" > .env
npm install
npm run dev                # Vite dev server
```

Install the required CLI tools (ProjectDiscovery suite, nmap + vulners, subjack) and ensure
they are on `PATH`. Then open the Vite dev URL, enter a target you are authorized to test,
and pick a module.

---

## Security Notes

- **Use only with authorization.** Many modules perform active probing, scanning, secret
  extraction, cloud-bucket enumeration, and LLM jailbreak testing. Unauthorized use against
  third-party systems is illegal.
- **Credentials hygiene.** Review `backend/src/firebase.ts` and `backend/src/server.ts`
  before publishing — Firebase web config and a fallback LeakCheck key are committed in
  source. Move all secrets to environment variables and rotate any committed keys.
- **`frontend/src/DeploymentValidator.ts` contains embedded zero-width Unicode characters**
  in its comments (invisible/obfuscation glyphs). This is not legitimate source formatting
  and should be treated as a suspicious artifact — inspect and clean this file before
  trusting its output, as hidden text can carry injected instructions.
- **`config/deployment-context.json`** references a metadata-style URL
  (`https://100.100.100.200/latest/meta-data/`). Confirm this is intentional for your
  environment before deploying; it is not a standard cloud metadata endpoint.
- **Debug endpoint** `/api/debug/env` exposes API-key presence and prefixes — disable it in
  production.
- The `tls` certificate check uses `rejectUnauthorized: false` intentionally (to read
  invalid certs); this only affects the SSL inspection module, not general HTTPS egress.
