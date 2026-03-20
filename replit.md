# BDL Command Center

## Overview

Dynasty fantasy football management tool. Integrates with Sleeper leagues, scrapes KeepTradeCut (KTC) market values, and applies a custom valuation engine to rank players across multiple league formats (1QB, Superflex, TE Premium, IDP-heavy).

## Stack

- **Monorepo tool**: pnpm workspaces (`packageManager: pnpm@9.15.9`)
- **Node.js version**: 24
- **TypeScript version**: 5.9
- **Frontend**: React + Vite (artifact: `bdl-command-center`)
- **Backend API**: Express 5 (artifact: `api-server`)
- **API contract**: OpenAPI 3.1 spec in `lib/api-spec`, codegen via Orval
- **Shared libs**: `@workspace/api-zod`, `@workspace/api-client-react`, `@workspace/db`
- **DB**: PostgreSQL + Drizzle ORM (not yet used in production flows)

## Structure

```text
BDL-Command-Center-master/
├── artifacts/
│   ├── api-server/                # Express API: KTC scraper, IDP rankings, coverage
│   │   ├── src/
│   │   │   ├── index.ts           # Entry: reads PORT, starts Express
│   │   │   ├── app.ts             # CORS, JSON parsing, mounts /api routes
│   │   │   ├── routes/            # health, ktcValues, ktcCoverage, idpRankings, nflverse
│   │   │   └── lib/               # ktcScraper, ktcCache, idpRankings, scheduler, teamService
│   │   │       └── nflverse/      # Football-intelligence layer (see below)
│   │   └── data/                  # ktc_values.json, idp_seed.json, nflverse_*.json (caches)
│   ├── bdl-command-center/        # React + Vite frontend
│   │   └── src/
│   │       ├── App.tsx            # Main app, Sleeper league detection
│   │       ├── services/          # Core valuation engine
│   │       │   ├── valuation.ts         # v4 engine — rank-anchored VORP, no SF double-count
│   │       │   ├── consensus.ts         # Dynasty consensus rank provider
│   │       │   ├── picks.ts             # Draft pick valuation
│   │       │   ├── leagueContext.ts
│   │       │   ├── rookies.ts
│   │       │   ├── tradeCalculator.ts   # Trade pool builder, nflverse context, verdict
│   │       │   ├── tradeIntelligence.ts # Front-office intelligence layer (P1-P4)
│   │       │   ├── providers/           # ktcProvider, idpProvider, fantasyPros, playerMetadata
│   │       │   └── platforms/           # sleeper, espn, yahoo, nfl, manual
│   │       └── components/
│   └── mockup-sandbox/            # Vite component preview server (canvas mockups)
├── lib/
│   ├── api-spec/                  # OpenAPI 3.1 spec + Orval codegen config
│   ├── api-client-react/          # Generated React Query hooks
│   ├── api-zod/                   # Generated Zod schemas
│   └── db/                        # Drizzle ORM schema + DB connection
├── scripts/
│   ├── proxy.mjs                  # Routing proxy: :22597 → /api → :8080, /* → :5173
│   └── src/
│       ├── validateValuation.ts   # Valuation engine validation report (Step 7+8 sanity checks)
│       └── hello.ts
├── pnpm-workspace.yaml            # catalog pins, overrides, minimumReleaseAge
├── tsconfig.base.json
├── tsconfig.json                  # Root project references (libs only)
└── package.json                   # packageManager: pnpm@9.15.9
```

## Valuation Engine (`valuation.ts` v4)

The core math engine in `artifacts/bdl-command-center/src/services/valuation.ts`. Key design principles:

- **Blend model**: per-position KTC/VORP weights (e.g. QB 1QB: 80/20, RB: 60/40, WR: 75/25, TE: 70/30)
- **VORP fallback**: when KTC is unavailable, `vorpRankScaler()` anchors rank 1→1.55×, rank 200+→0.60×
- **No SF double-count**: `sfValue` from KTC already encodes the superflex premium; SF bonus only applies in VORP-only mode
- **Clamped adjustments**: all non-age multipliers clamped to [0.82, 1.22] combined; individual caps: consensusAdj ±6%, scoringAdj ±15%, scarcityAdj ±18%, rosterNeedAdj ±5%
- **Age curve**: position-specific nudge only (not primary sort key)
- **Format-aware**: reads `FormatContext` (PPR, TE premium, superflex, IDP, roster sizes) from Sleeper league settings

Run the validation report: `pnpm --filter @workspace/scripts run validate`

## TypeScript

- **Typecheck from root**: `pnpm run typecheck` — builds libs first (project references), then checks artifacts
- Lib packages (`lib/*`) are composite and emit `.d.ts` only
- Artifact packages check with `tsc --noEmit`

## Routing Infrastructure

The project lives inside `BDL-Command-Center-master/` (a subdirectory of the workspace root). The root `.replit` maps `localPort = 22597 → externalPort = 80`, meaning all external traffic arrives on container port 22597.

A lightweight routing proxy (`scripts/proxy.mjs`) runs as the "Routing Proxy" workflow and listens on port 22597. It routes:
- `/api` and `/api/*` → port 8080 (API server)
- Everything else → port 5173 (Vite frontend)

This proxy is required because the Replit shared-proxy process does not automatically start for projects whose `[[artifacts]]` are declared in a subdirectory `.replit` rather than the workspace-root `.replit` (which cannot be edited).

The Vite config (`artifacts/bdl-command-center/vite.config.ts`) also has a dev-server proxy for `/api → :8080` so the frontend works correctly when accessed directly at port 5173.

**Do not remove the Routing Proxy workflow.** Without it, all external access returns 502.

## Running

- **Frontend**: `pnpm --filter @workspace/bdl-command-center run dev`
- **API server**: `pnpm --filter @workspace/api-server run dev`
- **Routing proxy**: `node BDL-Command-Center-master/scripts/proxy.mjs` (runs on port 22597)
- **API port**: reads `PORT` env var (default 8080)
- **KTC scraper**: runs every 6h via scheduler; cache at `artifacts/api-server/data/ktc_values.json`
- **IDP seed**: `artifacts/api-server/data/idp_seed.json`

## Player Pool & Ranked Valuation (`api-server`)

### Source data
| Source | File | Scope |
|--------|------|-------|
| DynastyProcess CSV | `data/dynasty_process.csv` | Offense, 661 players, columns `value_1qb`/`value_2qb` |
| FantasyPros IDP | scraped live via `idpProvider.ts` | Defense (~100 players), DL/LB/DB |

### Blend model (`playerPool.ts`)
- Offense: 60% DP `value_1qb` + 40% KTC for 1QB value; 55% DP `value_2qb` + 45% KTC for sfValue
- Only blends when both sources are nonzero; otherwise takes whichever is available
- IDP: FP IDP values pass through unchanged (no KTC for defense)

### Format presets (`leagueFormats.ts`)
Five presets matching real BDL league formats:
| Key | Label | Notes |
|-----|-------|-------|
| `1qb` | Standard 1QB | uses `value_1qb` blend |
| `sf` | Superflex | uses sfValue blend; QBs elevated |
| `teprem` | TE-Premium SF | SF + 18% TE scoring bonus |
| `idp` | IDP-Heavy SF | SF + IDP enabled (full FP IDP values) |
| `bdl` | BDL League | SF + IDP + 0.5 PPR (4% TE discount) |

Format adjustments applied only:
- `+18%` TE in TE-prem leagues
- `−4%` TE in 0.5 PPR leagues
- `0.05×` structural penalty for IDP positions in non-IDP leagues
- All scarcity multipliers are 1.0 (source data already encodes scarcity)

### Ranked endpoints (`routes/valuationRanked.ts`)
- `GET /api/player-pool/ranked?format=<key>&limit=N` — full ranked list with `overall` + `byPosition` breakdowns
- `GET /api/player-pool/ranked/compare?formats=a,b,c&name=partial` — cross-format value table
- `POST /api/player-pool/lineup-impact` — lineup audit: given a roster + lineup, returns delta-value per player

## API Routes (all under `/api`)

- `GET /api/healthz` — health check
- `GET /api/ktc-values` — current KTC player values
- `GET /api/ktc-coverage` — KTC match rate vs Sleeper player pool
- `GET /api/idp-rankings` — IDP seed rankings
- `GET /api/player-pool/ranked` — ranked player list by format (1qb/sf/teprem/idp/bdl)
- `GET /api/player-pool/ranked/compare` — cross-format value comparison
- `POST /api/player-pool/lineup-impact` — lineup audit / delta values
- `GET /api/news` — backend-cached NFL news from 5 RSS sources with intelligence fields
- `GET /api/news/sources` — source health check with per-source article counts
- `GET /api/news/players/:name` — articles mentioning a specific player
- `GET /api/news/teams/:team` — articles mentioning a specific team
- `GET /api/team-report/:leagueId/:rosterId` — front-office intelligence report for a Sleeper roster

### nflverse routes (`/api/nflverse/…`)
- `GET /api/nflverse/health` — cache status for all 7 nflverse data sources
- `GET /api/nflverse/player/:id?by=name|gsisId` — single-player lookup with meta+depth+injury+signals
- `GET /api/nflverse/signals?position=&team=&limit=` — filtered player signal list
- `GET /api/nflverse/depth-chart/:team` — full depth chart for one NFL team
- `GET /api/nflverse/injuries?position=&team=&status=` — current injury report
- `GET /api/nflverse/schedule/:team` — schedule + W/L record for one team
- `GET /api/nflverse/metadata/coverage` — coverage stats (totalPlayers, withDepth, withInjury, etc.)
- `POST /api/nflverse/refresh` — manually clear in-memory signal cache

## News Service (`api-server`)

### Architecture
- `lib/newsSources.ts` — Feed config (5 sources across 3 tiers) with tier and colours
- `lib/newsService.ts` — RSS fetcher, XML parser, classifier, relevance scorer, deduplicator, tagger
- `routes/news.ts` — GET /api/news + sub-routes handler

### Sources (as of March 2026)
| ID | Name | Tier | Notes |
|----|------|------|-------|
| `rotowire` | Rotowire | impact | Player-level injury/status reports |
| `cbssports` | CBS Sports | analysis | 36 articles, NFL analysis and transactions |
| `nfltraderumors` | NFL Trade Rumors | analysis | 10 articles, transaction-focused |
| `espn` | ESPN NFL | general | 23 articles, broad NFL coverage |
| `yahoo` | Yahoo Sports | general | 25 articles, broad NFL coverage |

**Note**: FantasyPros RSS permanently broken (returns 5-byte "Array" body). Replaced with CBS Sports + NFLTradeRumors.

### Intelligence pipeline
- `classifyArticle()` — rule-based keyword match to 7 article types
- `scoreRelevance()` — 0-100 score from source tier, player tags, article type, recency, keywords
- `deduplicateAndCluster()` — Jaccard similarity + tag overlap dedup within 8-hour window
- `tagArticle()` — full-name + unambiguous last-name matching against 760-player index; AMBIGUOUS_LAST_NAMES guard

### Normalised article shape
```
{ id, title, summary, url, source, sourceId, sourceColor, sourceTier,
  publishedAt, imageUrl, playerTags, teamTags,
  articleType, relevanceScore, relevanceTier, clusterId, relatedCount }
```

### Query params: `?source=`, `?type=`, `?tier=`, `?player=`, `?team=`, `?limit=`

## nflverse Intelligence Layer (`api-server/lib/nflverse/`)

Seven modules pulled from the nflverse CDN (GitHub releases), all async-cached to `data/nflverse_*.json`:

| Module | CDN source | Cache TTL |
|--------|-----------|-----------|
| `nflverseMetadata.ts` | `players/players.csv` (19,901 QB/RB/WR/TE/DL/LB/DB) | 24h |
| `nflverseDepthCharts.ts` | `depth_charts/depth_charts_<year>.csv` | 6h |
| `nflverseInjuries.ts` | `injuries/injuries_<year>.csv` | 2h |
| `nflverseStats.ts` | `player_stats/player_stats_<prior year>.csv` | 12h |
| `nflverseSchedules.ts` | `schedules/games.csv` | 12h |
| `nflverseSignals.ts` | Synthesised from the above five | 6h |
| `nflverseClient.ts` | Base CSV fetcher with `parseCsvAsync()` (yields every 2000 rows) | — |

**Key implementation notes:**
- All large-file reads use `parseCsvAsync()` to avoid event-loop blocking
- Correct CDN column names: `latest_team`, `years_of_experience`, `college_name`, `headshot`
- Sleeper ID cross-reference not available (nflverse `players.csv` doesn't include `sleeper_id` column); team enrichment is name+position based
- Signal lookup in teamService uses `normaliseForKtc(name)-position` key with name-only fallback

### PlayerSignals shape
```
{ gsisId, sleeperId, displayName, position, team,
  playerRoleConfidence, depthCompetitionScore, injuryRiskFlag,
  practiceStatus, recentProductionIndex, usageTrend,
  availabilityScore, depthRank, depthPosition,
  reportStatus, reportInjury, weeksWithData,
  fantasyPtsPerGame, avgTargets, avgCarries }
```

## Team Intelligence Service (`api-server`)

### Architecture
- `lib/teamService.ts` — Full front-office report generator; no external AI
- `routes/teamReport.ts` — GET /api/team-report/:leagueId/:rosterId handler

### Pipeline
1. Fetch Sleeper roster from `https://api.sleeper.app/v1/league/{id}/rosters`
2. Load KTC map, Sleeper player DB, and nflverse signals in parallel
3. Build signal name-index (`normaliseForKtc(name)-position` key + name-only fallback)
4. Match each player to KTC values; formula fallback for unmatched; attach nflverse signal
5. Build position group strength scores (0-100); apply -15pt penalty per injured starter (availabilityScore < 50, depthRank === 1)
6. Determine teamStatus via contenderScore + avgAge + pickWealth rules
7. Build recommendedActions — includes injury alerts for injured depth-rank-1 starters
8. Cross-reference news articles by player name for urgentNews
9. Resolve team display name from Sleeper users API

### Team report response shape
```
{ leagueId, rosterId, teamName, teamStatus,
  positionGroupStrength: { QB, RB, WR, TE, DL, LB, DB },  // each includes injuredStarters count
  weakSpots, coreAssets, picks, urgentNews,
  recommendedActions, metrics, generatedAt }
```
Where `metrics = { rosterValue, topTenValue, contenderScore, rebuildScore, draftCapitalScore, avgAgeTopTen, picksCount, totalPlayers }`

## UI Design System (`bdl-command-center`)

### Layout primitives (`components/ui/layout.tsx`)
All screens use shared primitives from `layout.tsx` — do not write inline page structure.

| Component | Purpose |
|-----------|---------|
| `ScreenRoot` | Outer flex-column container for every screen |
| `ScreenScroll` | Scrollable content area inside ScreenRoot |
| `ContentPad` | Standard 10px/12px padding wrapper |
| `PageHeader` | Standardized title block (icon, title, subtitle, right slot) |
| `PageContextRow` | Subtle metadata row — dots-separated pills |
| `FilterStrip` | Horizontal scrollable filter bar |
| `FilterPill` | Individual filter pill |
| `SearchBox` | Standard search input |
| `SectionLabel` | Small-caps section heading |
| `SectionFrame` | Elevated card container for content sections |
| `ItemCard` | Standard list item card (highlight, accent border variants) |
| `SurfaceCard` | Panel/settings summary box |
| `EmptyState` | Simple centered empty state |
| `EmptyPanel` | Rich empty/unavailable state with icon, description, action hint |
| `LoadState` | Centered loading indicator |
| `InlineAlert` | Inline warning/info box |
| `RankBadge` | Circular rank badge with tier-based color |
| `TeamSelectorStrip` | Horizontal scrollable team picker row |
| `ActivityFeedItem` | Activity feed card with colored left border |
| `RowDivider` | Horizontal rule |

### Shell
- `NavigationShell.tsx` — Header + team context band + bottom nav + more-menu overlay
  - Header: app icon + league name + season/platform + current-screen badge
  - Team context band: appears below header when a team is selected (`teamName && selectedTeam`)
  - Bottom nav: Command / Rankings / Trade Lab / Roster / More (5 items)

### Screen conventions
Every screen: `PageHeader` → optional `PageContextRow` → `FilterStrip`/`SearchBox` → `ScreenScroll` content
Transactions screen uses `ActivityFeedItem` for the feed layout.
Draft Board shows `EmptyPanel` when rookie provider is not connected.

## Installing Dependencies

Requires pnpm 9+ (uses `catalog:` protocol). Use `npx pnpm@9 install` if the nix-store pnpm binary is too old.
