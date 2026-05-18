# MikiBot — Production Blueprint

MikiBot is an AI-powered Valorant performance analysis and coaching platform built for **rank-up outcomes**, not generic chat. This repository defines a production-ready architecture, data model, AI orchestration, and implementation roadmap.

---

## 1) Full Application Architecture

### High-level system

```text
[Next.js Web App]
  ├─ Auth (Clerk)
  ├─ Dashboard + Reports UI
  ├─ Calls Backend API (FastAPI)
  └─ Optional Realtime (SSE/WebSocket)

[FastAPI Backend]
  ├─ Auth verification middleware (Clerk JWT)
  ├─ Riot ingestion services
  ├─ Match parser + stat feature engineering
  ├─ Rank/role benchmark engine
  ├─ Coaching orchestration engine
  ├─ OpenAI client (GPT-5.5)
  ├─ Report validator + confidence scoring
  └─ REST API + background workers

[PostgreSQL]
  ├─ User/account domain
  ├─ Match raw + normalized stats
  ├─ Benchmarks + snapshots
  ├─ Coaching reports + action plans
  └─ Progress tracking + longitudinal trends

[Job Queue + Scheduler]
  ├─ Periodic sync (riot pull)
  ├─ Recompute aggregates
  ├─ Generate reports
  └─ Alerting + retries + dead-letter

[Observability]
  ├─ Structured logs
  ├─ Metrics (latency, token cost, success rate)
  └─ Tracing + error monitoring
```

### Core services
- **Ingestion Service:** pulls Riot data and normalizes entities.
- **Feature Service:** computes role/rank/map/phase features.
- **Benchmark Service:** compares users against rank-role distributions.
- **Coaching Service:** builds tactical narrative and drills from evidence.
- **Progress Service:** tracks trendline and intervention impact.

---

## 2) Proposed Folder Structure

```text
mikibot/
├─ apps/
│  ├─ web/                         # Next.js + Tailwind + TS
│  │  ├─ src/
│  │  │  ├─ app/
│  │  │  ├─ components/
│  │  │  ├─ features/
│  │  │  │  ├─ dashboard/
│  │  │  │  ├─ matches/
│  │  │  │  ├─ coaching/
│  │  │  │  └─ progression/
│  │  │  ├─ lib/
│  │  │  └─ styles/
│  │  └─ package.json
│  └─ api/                         # FastAPI
│     ├─ app/
│     │  ├─ main.py
│     │  ├─ api/
│     │  │  ├─ v1/
│     │  │  │  ├─ routes_auth.py
│     │  │  │  ├─ routes_players.py
│     │  │  │  ├─ routes_matches.py
│     │  │  │  ├─ routes_reports.py
│     │  │  │  └─ routes_benchmarks.py
│     │  ├─ core/
│     │  │  ├─ config.py
│     │  │  ├─ logging.py
│     │  │  └─ security.py
│     │  ├─ db/
│     │  │  ├─ models/
│     │  │  ├─ session.py
│     │  │  └─ migrations/
│     │  ├─ services/
│     │  │  ├─ riot/
│     │  │  ├─ parsing/
│     │  │  ├─ features/
│     │  │  ├─ benchmarks/
│     │  │  ├─ coaching/
│     │  │  └─ llm/
│     │  ├─ workers/
│     │  └─ schemas/
│     └─ pyproject.toml
├─ packages/
│  ├─ shared-types/
│  ├─ ui-kit/
│  └─ analytics-sdk/
├─ infra/
│  ├─ docker/
│  ├─ terraform/
│  └─ monitoring/
└─ docs/
   ├─ architecture.md
   ├─ prompt-system.md
   ├─ api-spec.md
   └─ runbooks/
```

---

## 3) Backend Design (FastAPI)

### Key modules
- `riot_client`: secure Riot API access, rate limit backoff, idempotent pulls.
- `match_parser`: round-level transformations (economy, utility, entries, first deaths, clutches).
- `feature_builder`: produces model-ready tactical features.
- `benchmark_engine`: percentile and z-score by rank, role, map, patch window.
- `coaching_engine`: weakness prioritization + tactical recommendation synthesis.
- `report_guardrails`: prevents unsupported claims and flags missing data.

### Async processing
- Use **Celery/RQ + Redis** for long jobs:
  - historical ingest
  - benchmark recalculation
  - report generation
- API returns job IDs; frontend polls or uses SSE.

---

## 4) Frontend Design (Next.js)

### Core pages
- `/dashboard`: KPI tiles (ACS trend, HS%, first death rate, rank delta)
- `/matches`: match timeline + map filters + role filters
- `/coaching/[reportId]`: full tactical report with confidence tags
- `/improvement`: routines + completion tracking
- `/agents`: agent pool and role fit analysis

### UX patterns
- Dark tactical UI, compact grids, sparkline trend cards.
- “Evidence chips” beside every coaching claim.
- Red/amber/green impact labels based on rank-up ROI.

---

## 5) Database Schema (PostgreSQL)

### Core tables
- `users(id, clerk_id, riot_puuid, region, created_at)`
- `player_profiles(user_id, current_rank, rr, preferred_roles[], preferred_agents[])`
- `matches(id, user_id, match_id, map, queue, started_at, won, patch_version)`
- `round_events(match_id, round_no, side, econ_state, first_blood, first_death, clutch_attempt, clutch_win)`
- `player_match_stats(match_id, user_id, agent, role, acs, adr, kda, hs_pct, kast, fk, fd, trade_pct, util_damage, util_assists)`
- `attack_defense_splits(match_id, user_id, attack_acs, defense_acs, attack_fd_rate, defense_fd_rate)`
- `benchmarks(rank, role, map, metric, p25, p50, p75, p90, sample_size, patch_window)`
- `coaching_reports(id, user_id, report_type, generated_at, confidence, payload_json)`
- `improvement_plans(id, user_id, start_date, end_date, plan_json, adherence_score)`
- `progress_snapshots(user_id, snapshot_date, rank, rr, rolling_win_rate, rolling_acs, rolling_hs_pct)`

---

## 6) API Design

### REST endpoints (v1)
- `POST /auth/sync`
- `POST /riot/connect`
- `POST /riot/sync` (async)
- `GET /players/me/overview`
- `GET /players/me/matches?limit=&map=&role=`
- `GET /players/me/agents`
- `GET /players/me/benchmarks`
- `POST /reports/generate` (async)
- `GET /reports/{report_id}`
- `GET /improvement-plan/current`
- `POST /improvement-plan/checkin`

### Response contract
Each coaching report block includes:
- `claim`
- `evidence[]` (metric + value + benchmark + delta)
- `confidence` (`low|medium|high`)
- `recommended_actions[]`

---

## 7) Riot API Integration Plan

1. Store encrypted Riot identifiers and region.
2. Ingest latest N matches on sync trigger.
3. Deduplicate by `match_id + user_id`.
4. Persist raw JSON for audit + replay.
5. Parse to normalized tactical events.
6. Run feature extraction pipeline.
7. Backfill benchmark cohorts nightly.
8. Respect rate limits with token bucket + exponential backoff.

---

## 8) AI Orchestration Flow

1. Load user context + last 30/60/100 matches.
2. Compute metrics and deltas vs benchmark.
3. Run weakness scoring model.
4. Build structured “evidence packet.”
5. Call LLM with strict system prompt + JSON schema output.
6. Validate each claim maps to evidence.
7. Reject/repair unsupported text.
8. Persist coaching report and plan.

---

## 9) Production-Grade System Prompt

Use this as the **system prompt**:

> You are MikiBot, an elite Valorant performance analyst and tactical coaching engine.
> 
> Core identity:
> - Professional esports analyst
> - Rank-up optimization coach
> - Tactical, direct, brutally honest
> - No motivational fluff, no fake positivity, no emotional manipulation
> 
> Non-negotiable behavior:
> 1) Never invent data.
> 2) Never assume missing stats.
> 3) Never present uncertain patterns as facts.
> 4) Every coaching claim must be tied to provided evidence.
> 5) If evidence is insufficient, explicitly say so and lower confidence.
> 
> Output requirements:
> - Tactical Summary
> - Biggest Weaknesses
> - Rank-Specific Analysis
> - Role-Based Analysis
> - Match Pattern Detection
> - Tactical Adjustments
> - Training Routine
> - Immediate Priorities
> - Long-Term Focus
> 
> Analysis rules:
> - Always consider: rank context, MMR trend, win/loss consistency, agent pool depth, HS%, ACS trends, entry success, first death rate, clutch conversion, economy efficiency, attack vs defense impact, utility effectiveness, map-specific performance, role performance, and consistency.
> - Adapt advice by rank band:
>   - Iron–Silver: mechanics/fundamentals/consistency
>   - Gold–Diamond: positioning/utility timing/decision quality
>   - Ascendant–Radiant: macro tempo/map pressure/info denial
> - Adapt advice by role (Duelist, Controller, Initiator, Sentinel).
> 
> Style rules:
> - Specific, tactical, concise.
> - No generic advice (e.g., “improve aim and comms”).
> - Explain causal links (behavior → metric outcome → round impact).
> - Prioritize actions by expected rank-up impact.
> 
> Missing-data policy:
> - State exact missing fields.
> - Provide provisional coaching only where supported.
> - Ask for additional data needed to increase confidence.

---

## 10) LLM Workflow

- **Model:** GPT-5.5 for report generation.
- **Input:** structured evidence JSON, rank/role context, recent trend summary.
- **Output:** strict JSON schema (`report_sections`, `priorities`, `routine`, `confidence_notes`).
- **Post-processing:** evidence-link validator + profanity/abuse filters + UI markdown renderer.

---

## 11) Memory System Design

Three-layer memory:
1. **Session memory:** current request context.
2. **Short-term performance memory (30-day):** volatile trend deltas.
3. **Long-term progression memory:** persistent milestones and recurring weaknesses.

Store “intervention outcomes” (e.g., reduced first-death rate after 2 weeks) to personalize future plans.

---

## 12) Role-Analysis Engine

### Duelist
- entry timing score
- space-creation index
- first-blood conversion to round win
- trade distance risk index

### Controller
- smoke value per round
- denial timing quality
- retake utility preservation
- post-plant stall efficiency

### Initiator
- recon-to-kill conversion
- flash team value
- utility-chain timing score
- support proximity quality

### Sentinel
- anchor hold rate
- flank trap uptime value
- anti-lurk detection rate
- rotate discipline score

---

## 13) Rank Benchmarking Logic

- Segment by `rank x role x map x patch_window`.
- Compute percentile bands and robust z-scores.
- Flag weaknesses where:
  - metric < p35 and
  - trend direction negative over 20 matches and
  - metric has high round-win correlation.
- Create impact score:
  - `impact = deficiency * win_corr * consistency_penalty`.

Use impact score to rank priorities.

---

## 14) Coaching Generation Pipeline

1. Data ingest
2. Feature computation
3. Benchmark comparison
4. Weakness ranking
5. Tactical hypothesis drafting
6. LLM report generation
7. Evidence validation
8. Routine generation
9. Progress target assignment

Each routine item must include:
- objective
- daily/weekly volume
- success metric
- review checkpoint

---

## 15) Deployment Instructions

### Frontend (Vercel)
- Connect Git repo
- Set env vars: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- Build command: `pnpm build`

### Backend (Railway/Render)
- Deploy FastAPI service
- Env vars: `DATABASE_URL`, `REDIS_URL`, `OPENAI_API_KEY`, `RIOT_API_KEY`, `CLERK_JWT_ISSUER`
- Run migrations during release
- Run API + worker processes separately

### Database
- Managed Postgres with PITR
- Daily logical backups and weekly restore drills

---

## 16) MVP Roadmap (6–8 weeks)

- Week 1: Auth + Riot link + base schema
- Week 2: Match ingest + parser + stat dashboard
- Week 3: Benchmark engine v1
- Week 4: LLM coaching engine + system prompt + guardrails
- Week 5: Improvement plans + routine tracker
- Week 6: QA, observability, cost controls
- Week 7–8: closed beta + iteration

---

## 17) Scaling Roadmap

Phase 2:
- VOD upload + event tagging
- Team analytics (duo/squad)
- Discord summary bot

Phase 3:
- Real-time assistant overlays
- Voice coaching summaries
- Tournament/scrim prep modules

---

## 18) Security Considerations

- JWT verification for every API call
- Row-level access controls by `user_id`
- Encrypt secrets at rest and in transit
- PII minimization and retention limits
- Audit logs for report generation and data access
- Rate limiting + abuse detection

---

## 19) Cost Optimization Strategy

- Tiered generation:
  - lightweight heuristic report for frequent refreshes
  - full GPT-5.5 report on schedule/manual trigger
- Cache benchmark snapshots
- Batch ingest jobs off-peak
- Token budget controls (max context windows, compact evidence packets)
- Monitor cost per active user and per report

---

## 20) Recommended Development Phases

1. **Foundation:** auth, ingest, schema, dashboard.
2. **Intelligence:** benchmarking + weakness detection.
3. **Coaching:** LLM prompts, validation, routines.
4. **Optimization:** observability, reliability, cost.
5. **Expansion:** VOD, Discord, realtime, mobile.

---

## Example Coaching Response Contract (for UI)

```json
{
  "tactical_summary": "...",
  "biggest_weaknesses": [
    {
      "issue": "High first death rate on attack",
      "evidence": ["attack_fd_rate=0.24 vs rank-role p50=0.17"],
      "impact": "high",
      "confidence": "high"
    }
  ],
  "rank_specific_analysis": "...",
  "role_based_analysis": "...",
  "match_pattern_detection": "...",
  "tactical_adjustments": ["..."],
  "training_routine": [{"day":"Mon", "drills":["..."]}],
  "immediate_priorities": ["..."],
  "long_term_focus": ["..."]
}
```

This blueprint is implementation-ready and can be split into epics/stories immediately.
