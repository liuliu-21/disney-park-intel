# 🏰 Disney Park Intelligence

English | [中文](README.md)

An AI itinerary planner built for Shanghai Disneyland.

I started this project because park guides are plentiful, but a ranked list of attractions is not very useful once you are inside the park. What people actually need is an executable plan that can account for live queues, walking distance, height restrictions, dining reservations, Premier Access, and personal preferences at the same time.

👉 [Try the live app](https://disney-park-intel.vercel.app)

## What it does

- Builds a day plan from group heights, preferences, pace, and Premier Access choices
- Orders attractions using live waits, historical snapshots, and walking cost
- Treats dining reservations, shows, and must-do attractions as hard anchors
- Revises a plan through natural language while streaming the Agent's tool activity
- Includes attraction, restaurant, shop, and photo-spot pages, with offline caching

## Where most of the work went

The hardest part was not getting the interface to run. It was proving that a plausible-looking itinerary was actually correct.

Earlier versions contained several silent failures: external provider IDs did not map to internal attractions; Chinese reviews were split on whitespace, which made almost every retrieval score zero; and walking cost was repeatedly measured from the starting point, causing routes to jump across the park. None of these bugs crashed the application. They simply produced bad advice.

I added deterministic evaluations and boundary tests, rewrote the Chinese BM25 tokenizer and scorer, centralized provider-ID mapping, and reworked the scheduler. That is the part of this project I care about most: it is not only an LLM interface, but an attempt to build a decision system that is testable, explainable, and honest when its data is weak.

## How it works

```text
preferences + time / height / reservation constraints
                         │
                         ▼
              filter and score candidates
                         │
                         ▼
       live wait > historical forecast > static baseline
                         │
                         ▼
          greedy routing + hard anchors + gap filling
                         │
                         ▼
                  short Claude explanation
```

The LLM does not invent the schedule. Time, height, Premier Access intervals, and reservation anchors are handled by deterministic code. Claude selects tools, interprets user intent, and explains the result. The Agent runs for at most five rounds and streams text and tool progress over SSE.

```text
cost = waitWeight × effectiveWait
     + walkWeight × walkMinutes
     + energyWeight × thrillScore × 5
```

The `efficient`, `balanced`, and `easy` modes change the relative cost of waiting, walking, and physical intensity.

## Data and fallback behavior

| Capability | Source | Fallback |
|---|---|---|
| Live wait times | themeparks.wiki; 20 of 24 attractions mapped | Static baseline with `fallback: true` |
| Wait forecast | Versioned historical snapshots | With fewer than 8 samples, extrapolate from the current snapshot and report low confidence |
| Attraction reviews | Offline collection of public Xiaohongshu posts | Uncovered targets use clearly labelled hand-written samples |
| AI scoring | Structured Claude output | Local rules when credentials are absent or the call fails |
| Conversation state | Process memory; optional Upstash Redis | Without Redis, state is lost on a cold start |

The repository currently contains 24 attractions, 11 restaurants, 29 shops, 44 photo locations, 280 real Xiaohongshu posts covering 14 attractions, and 1,300 versioned wait-time snapshots.

I do not present those numbers as user counts. The project does not yet have verified active-user or retention data.

### Known limitations

- Xiaohongshu has no star rating. The stored `rating` is a popularity proxy, not a satisfaction score.
- Dictionary sentiment performs poorly on informal language, hashtags, and emoji; about 75% of collected posts are classified as neutral.
- Real restaurant reviews have not been collected. Those targets expose a fallback marker.

## Reproducible results

```bash
npm test
```

There are 170 tests covering provider IDs, wait parsing, height boundaries, Premier Access, route constraints, Chinese retrieval, session persistence, rate limits, validation, and SSE framing. Three live-network checks are skipped by default.

```bash
RATE_LIMIT_LLM=100000 npm run dev
python3 scripts/eval_itinerary.py
```

**100/100 itinerary scenarios pass** across normal, time, height, Premier Access, anchor, and route-mode categories. A deterministic script performs the scoring without calling an LLM.

```bash
npm run eval:retrieval
```

| P@1 | P@3 | Recall@3 | MRR | nDCG@5 |
|---:|---:|---:|---:|---:|
| 0.944 | 0.556 | 0.678 | 0.963 | 0.790 |

These results use 14 hand-written examples and 18 queries labelled by one person. They are useful for comparing retrieval changes, but **they are not evidence of production retrieval quality**. `eval_tool_accuracy.py` has not been run, and the repository does not claim a result that does not exist.

## Run locally

```bash
git clone https://github.com/liuliu-21/disney-park-intel.git
cd disney-park-intel
npm install
cp .env.local.example .env.local
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). The core application works without Anthropic credentials: scoring falls back to local rules, while the AI assistant returns a descriptive 503 response.

> If you do not use `ANTHROPIC_API_KEY`, remove the line instead of leaving an empty value. An empty value still masks other valid credentials.

## Stack and repository map

- Next.js 14 App Router, TypeScript, Tailwind CSS, and Zustand
- Anthropic Tool Use API with SSE streaming
- BM25 with Chinese character bigrams
- Vitest, GitHub Actions, Vercel, and optional Upstash Redis

```text
src/app/                 pages and API routes
src/app/api/agent/       Agent loop, tool definitions, and execution
src/lib/routing.ts       routing and constraint handling
src/lib/wait-*.ts        live wait service and historical forecast
src/lib/vector-store.ts  BM25 retrieval
data/                    reviews and wait snapshots
scripts/                 collection, validation, and evaluation tools
```

## Authorship and attribution

The `JINGYU LIU` and `Myla0619` identities in the Git history belong to the same author. The remaining snapshot commits were generated by GitHub Actions configured by that author.

External APIs, Disney attraction information, and public Xiaohongshu content were not created by this project. The project implements their collection, mapping, fallback behavior, retrieval, and evaluation.
