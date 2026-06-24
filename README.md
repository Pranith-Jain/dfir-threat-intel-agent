# DFIR & ThreatIntel Autonomous Investigator Agent

An autonomous DFIR (Digital Forensics and Incident Response) and threat intelligence investigator agent built on **Cloudflare Workers** with **Durable Objects** for stateful, multi-step investigations.

## What It Does

Describe what to investigate in natural language — the agent autonomously:

1. **Classifies** the query type (IP, domain, hash, CVE, actor, phishing, ransomware)
2. **Plans** which intelligence tools to call using an LLM
3. **Executes** tools in parallel (98 threat intel tools)
4. **Observes** results and decides what to investigate next
5. **Repeats** for up to 8 reasoning steps
6. **Synthesizes** a structured intelligence report with citations

## Architecture

```
User Query
    │
    ├── POST /api/v1/agent/investigate
    │
    ▼
┌─────────────────────────────────────────────┐
│  InvestigatorAgentDO (Durable Object)       │
│  ┌─────────────────────────────────────┐    │
│  │  Alarm-driven state machine         │    │
│  │                                     │    │
│  │  PLAN → ACT → OBSERVE → repeat      │    │
│  │  (max 8 steps, 10s tool timeout)    │    │
│  │              │                      │    │
│  │              ▼                      │    │
│  │        SYNTHESIZE report            │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  Steps persisted to D1 agent_sessions       │
│  Real-time progress via SSE streaming       │
└─────────────────────────────────────────────┘
```

### Key Components

| File | Purpose |
|------|---------|
| `src/lib/agent/types.ts` | Core type definitions (AgentState, AgentStep, AgentTool, etc.) |
| `src/lib/agent/tools.ts` | Tool registry — 98 threat intel tools wrapping platform API endpoints |
| `src/lib/agent/prompts.ts` | LLM system prompts for planner, observer, and synthesizer roles |
| `src/lib/agent/planner.ts` | LLM-powered planning: decides which tools to call next |
| `src/lib/agent/observer.ts` | Summarizes tool results after each step |
| `src/lib/agent/synthesizer.ts` | Final report generation from investigation history |
| `src/routes/agent.ts` | Hono API routes (investigate, poll, stream, sessions) |
| `worker/durable-objects/investigator-agent.ts` | Durable Object — alarm-driven agent loop |
| `migrations/0016_agent_sessions.sql` | D1 schema for session persistence |
| `src/pages/dfir/AgentInvestigator.tsx` | React frontend with real-time SSE streaming |

## Agent Loop

Each investigation step follows this cycle:

### 1. PLAN
The LLM (Groq primary, Workers AI fallback) receives the query, investigation history, and available tools. It decides which 1-3 tools to call and why.

### 2. ACT
Tool calls execute in parallel against the platform's API via the SELF service binding (in-process, no DNS round-trip). Each tool has a 10-second timeout.

### 3. OBSERVE
The LLM summarizes what was found, extracts key facts, and identifies gaps.

### 4. DECIDE
If the agent has enough information (or hit the step budget), it synthesizes. Otherwise, it plans the next step.

### 5. SYNTHESIZE
A final LLM pass produces a structured intelligence report with:
- TL;DR executive summary
- Key findings with confidence tags
- Detailed analysis with evidence citations
- MITRE ATT&CK mapping
- Actionable recommendations

## Available Tools

| Category | Tools |
|----------|-------|
| **IOC & Reputation** | `check_ioc`, `get_live_iocs`, `get_trending_iocs`, `get_ioc_lifecycle`, `correlate_iocs`, `ioc_watchlist_add/list/alerts/stats` |
| **CVE & Vulns** | `lookup_cve`, `poc_scan`, `cve_poc_map`, `cve_health`, `soc_cve_report` |
| **Domain & Network** | `lookup_domain`, `lookup_asn`, `lookup_ip_geo`, `get_domain_history`, `pivot_domain`, `search_registrant`, `get_domain_certs`, `watch_domain_ct`, `passive_dns_query/reverse/overlap` |
| **Actors & Malware** | `enrich_actor`, `search_malpedia`, `search_malware`, `search_triage` |
| **Intel Feeds** | `get_live_iocs`, `get_ransomware_activity`, `get_supply_chain_attacks`, `get_cert_in_advisories`, `get_detections`, `get_threat_pulse`, `get_today_briefing`, `cyber_news` |
| **Breach** | `check_breach` |
| **MITRE** | `lookup_mitre`, `generate_yara_rule`, `validate_yara_rule` |
| **Web & Phishing** | `scan_website`, `analyze_phishing_url`, `analyze_phishing_email`, `wayback_lookup`, `google_dorks` |
| **Crypto** | `trace_crypto_address` |
| **Reports** | `parse_threat_report`, `analyze_report`, `extract_ttps`, `extract_fivew`, `extract_iocs_from_image`, `get_cross_report_graph` |
| **Investigation** | `notebook_list/create/get/add_entry/update/delete`, `ws_list/create/get/add_subject/add_connection/add_finding/exposure/export_stix/render_graph/workflow_advance/workflow_summary` |
| **Telegram** | `tg_boolean_search`, `tg_timeline`, `tg_saved_searches_list/create/delete` |
| **HudsonRock** | `hr_search_email/domain/username/ip`, `hr_domain_overview`, `hr_assets_discovery`, `hr_third_party_risk`, `hr_infection_analysis`, `hr_account` |

## API Endpoints

```
POST /api/v1/agent/investigate    Start a new investigation
GET  /api/v1/agent/:id            Poll investigation state
GET  /api/v1/agent/:id/stream     SSE stream of step events
GET  /api/v1/agent/sessions       List recent investigations
```

### Example

```bash
# Start an investigation
curl -X POST https://your-domain/api/v1/agent/investigate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"query": "Investigate APT28 recent activity"}'

# Returns: {"id": "uuid", "queryType": "actor", "maxSteps": 8, "status": "running"}

# Stream progress
curl -N https://your-domain/api/v1/agent/uuid/stream

# Poll final state
curl https://your-domain/api/v1/agent/uuid
```

## Infrastructure

- **Runtime**: Cloudflare Workers
- **State**: Durable Objects (SQLite-backed, alarm-driven)
- **Database**: D1 (agent sessions, step history)
- **LLM**: Groq (llama-4-scout-17b) primary, Workers AI (llama-3.3-70b) fallback
- **Cache**: KV + Cache API (1h edge cache for intel feeds)
- **Streaming**: Server-Sent Events for real-time progress

## LLM Strategy

- **Groq** (primary): Free tier, fast, good at structured JSON output
- **Workers AI** (fallback): `@cf/meta/llama-3.3-70b-instruct-fp8-fast` then `@cf/meta/llama-3.1-8b-instruct`
- **Rate limit handling**: Fail-fast on 429, never retry within the same account
- **Cost control**: Max 8 steps per investigation, 10s tool timeout, deterministic fallbacks

## Design Decisions

1. **Durable Object for state** — Survives Worker restarts, supports long-running investigations via alarm-driven scheduling (same pattern as the report builder)
2. **Reuse existing API tools** — No new tool definitions; the agent wraps the same endpoints as the MCP server
3. **Capped step budget** — Controls cost and latency; configurable per investigation
4. **SSE streaming** — Frontend sees each step as it happens without WebSocket complexity
5. **Deterministic fallbacks** — If the LLM is unavailable, the observer falls back to rule-based summaries

## License

MIT License — see [LICENSE](./LICENSE).

## Part of

This agent is part of the [DFIR & ThreatIntel Portfolio Platform](https://pranithjain.qzz.io) — a comprehensive cybersecurity analysis platform with 98+ MCP tools and 230+ total tools across DFIR and Threat Intelligence domains.
