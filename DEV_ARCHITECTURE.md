# Developer Architecture Notes

## Current Codebase Map

This project is currently a Bannerlord-focused prototype for a broader future Roleplaying Companion product. Module support is not implemented yet. The current app is a small Node.js server with a static browser frontend, local JSON campaign storage, and a provider-agnostic local LLM scene-generation path.

### Project Root

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `package.json` | configuration / entry point | Defines the preferred project-root startup script: `npm start`. | Dependency-free at the moment. Keep this minimal unless runtime code adds real external dependencies. |
| `package-lock.json` | configuration | Lockfile matching the dependency-free package setup. | Should stay simple while dependencies remain empty. |
| `.env` | configuration / LLM | Local runtime LLM settings. | May contain local secrets or machine-specific values. Do not expose values to frontend code. |
| `.env.example` | configuration / documentation | Example LLM provider settings for LM Studio. | Keep in sync with required variables in `src/llm/llmClient.js`. |
| `README.md` | documentation | Older project overview and test instructions. | Some commands reference bundled Codex runtimes; `RUNNING.md` is now the preferred human startup guide. |
| `RUNNING.md` | documentation / workflow | Windows/PowerShell startup instructions. | Current preferred run workflow lives here: start LM Studio, run `npm start`, open `http://127.0.0.1:5177/`. |
| `DEV_ARCHITECTURE.md` | documentation | Current developer codebase map. | This file is documentation only and should be updated when responsibilities move. |
| `server.out.log`, `server.err.log` | runtime | Local server output/error logs from previous runs. | Runtime artifacts. Do not treat as source of truth for app behavior. |

### Current Entry Points

| Entry point | Path / command | Category | Notes |
| --- | --- | --- | --- |
| Project startup entry point | `npm start` | workflow | Runs from the project root. |
| Node/server entry point | `src/server.js` via `node src/server.js` | backend | Starts the HTTP server on `127.0.0.1:5177` by default. |
| Frontend/browser entry point | `public/index.html` | frontend / UI | Served by the Node backend. Opening this directly with `file:///` bypasses backend APIs and is not the real app workflow. |
| Main browser script | `public/app.js` | frontend / UI / Chronicle | Owns polling, player input submission, active scene rendering, choices, validation display, Chronicle rendering, and rollback button wiring. |
| Main stylesheet | `public/styles.css` | frontend / UI | Bannerlord-inspired shell and responsive layout. |
| Main storage entry point | `src/store.js` | storage / data | Reads and writes JSON files under `campaign/`; also owns one-turn rollback snapshots. |
| Current LLM route entry point | `POST /api/llm/generate-scene` in `src/server.js` and `src/routes/llmRoutes.js` | backend / LLM | Browser calls the Node backend only; backend calls the configured provider. |
| Chronicle API entry point | `GET /api/chronicle` in `src/server.js` | backend / Chronicle | Returns summarized state, recent events, records, and rollback availability. |
| Rollback API entry point | `POST /api/rollback` in `src/server.js` | backend / storage | Restores the latest snapshot created before a player input / LLM scene turn. |
| Older scripted turn API | `POST /api/turn` in `src/server.js` and `src/engine.js` | backend / legacy game loop | Still present, but the current frontend submits to `/api/llm/generate-scene`. |

### Frontend Files

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `public/index.html` | frontend / UI | Static DOM shell for the companion ledger. Defines topbar, active scene, choices container, input form, Chronicle panel, side records, and validation card. | Keep element IDs stable unless `public/app.js` is updated at the same time. |
| `public/app.js` | frontend / UI / Chronicle / API client | Main browser logic. Posts player actions to `/api/llm/generate-scene`, renders scenes and choices, polls `/api/chronicle`, renders Chronicle entries, and calls rollback. | Important / Risk Candidate. Mixes API client, UI renderer, polling, Chronicle display, enforcement display, and interaction handling. The frontend should continue sending player input only, not authoritative campaign context. |
| `public/styles.css` | frontend / UI | Bannerlord-inspired dark ledger layout, panel styling, choice grid, scroll behavior, and responsive layout. | Important / Risk Candidate. Layout changes can easily affect playability, scrolling, and input reachability. |

### Backend And Server Files

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `src/server.js` | backend / entry point / API | Creates the HTTP server, serves static frontend files, implements `/api/chronicle`, `/api/turn`, `/api/llm/generate-scene`, and `/api/rollback`, and summarizes campaign state for the frontend. | Important / Risk Candidate. Central routing file and bridge between UI, storage, old scripted engine, local LLM route, Chronicle summary, and rollback. |
| `src/calendar.js` | backend / time | Bannerlord time model: seasons, days, time blocks, formatting, parsing, comparison, and rough action duration estimation. | Important / Risk Candidate. Time rules affect Chronicle ordering, time advancement, rollback expectations, and campaign consistency. |
| `src/store.js` | storage / data / rollback | Loads campaign JSON collections, writes records, saves state, computes campaign revision, creates and restores one-turn snapshots. | Important / Risk Candidate. Direct filesystem persistence layer; mistakes can corrupt campaign memory or rollback behavior. |
| `src/llmTurnAdapter.js` | backend / LLM / Chronicle / storage | Commits successful validated LLM scene turns to campaign events, advances time, creates rollback snapshots, and saves state. | Important / Risk Candidate. This is the current bridge from generated scenes into canon/history. Failed validation should not reach this layer. |
| `src/engine.js` | backend / legacy game loop / Chronicle | Older scripted GM loop with hardcoded story branches, random events, enforcement, character memory updates, time skips, and dashboard regeneration. | Important / Risk Candidate. Largest file; mixes many responsibilities and hardcoded campaign/story data. Current frontend does not use it for normal LLM scene generation, but `/api/turn` still exposes it. |
| `src/context.js` | backend / prompt context / legacy | Builds compact context packets and token estimates for older scripted choice-resolution flows. | Important / Risk Candidate. Contains instruction text about narrator behavior and third-person player references for legacy context packets. |
| `src/html.js` | backend / generated HTML / legacy dashboard | Renders an older static dashboard into `campaign/html/index.html` and `outputs/bannerlord_companion_dashboard.html`. | Runtime-adjacent legacy path. The current live app is `public/index.html`; this generated dashboard may be stale or secondary. |
| `src/run_turn.js` | backend / CLI test utility | Runs the older scripted turn loop from the command line. | Useful for old tests, not the current browser LLM workflow. |
| `src/enforcementContextController.js` | backend / enforcement / context | Owns clean Enforcement v0 validation, deprecated legacy classification, active scope extraction, exact scope matching, clean feed projection, and compact LLM binding-constraint formatting. | Enforcement v0 is a scoped normative-record context source. Its clean feed projection and matched active constraints are separate outputs and should not be collapsed into one selection. |

### LLM, Prompt, And Validation Files

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `src/routes/llmRoutes.js` | backend / LLM route | Validates incoming campaign packets, builds prompts, calls provider, retries once for schema repair, validates output, and returns success/failure response bodies. | Important / Risk Candidate. This is the main local LLM scene-generation control flow and validation gate. |
| `src/llm/llmClient.js` | LLM / provider / configuration | Loads `.env`, validates required LLM environment variables, and calls an OpenAI-compatible `/chat/completions` endpoint. | Important / Risk Candidate. Must remain backend-only; do not expose provider URLs or keys in frontend code. |
| `src/llm/promptBuilder.js` | prompt / LLM | Builds system and user prompts from the campaign packet and content/entity/memory rules. | Important / Risk Candidate. Prompt behavior directly affects narrator output and the known prose/player-input echo issue. Do not change during non-prompt tasks. |
| `src/llm/contentRules.js` | prompt / validation | Defines content, entity, and memory rules plus banned/invented-canon regex patterns. | Important / Risk Candidate. Changes here alter both instructions and hard validation behavior. |
| `src/llm/sceneSchema.js` | LLM / validation | Defines required structured JSON output shape and `response_format` for the provider call. | Important / Risk Candidate. Frontend and storage expect this shape: scene, choices, continuity notes, memory candidates, entities, unsupported inventions, uncertainties. |
| `src/llm/validateSceneOutput.js` | validation | Validates schema shape, entity constraints, banned content patterns, unsupported inventions, and warning conditions. | Important / Risk Candidate. Determines whether generated output may become canon and be saved. |
| `src/llm/entityValidation.js` | validation / canon | Checks named entities against allowed and relevant campaign-packet entities. | Important / Risk Candidate. This is central to preventing unsupported location/character invention. |

### Storage And Data Folders

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `campaign/state.json` | data / storage | Current campaign state: campaign name, current time, location, treasury, random seed, and current scene. | Important / Risk Candidate. Directly affects displayed campaign state and time advancement. |
| `campaign/characters/` | data / storage / memory | Character dossiers for known characters such as Manteos, Mesui, Mela, and Player one. | Important / Risk Candidate. Character memory and relationships are persistent and should be treated as campaign canon. |
| `campaign/events/` | data / Chronicle / storage | Event and Chronicle records, including older scripted events and local LLM scene turns. | Important / Risk Candidate. This is the historical campaign record. Generated choices and metadata may be stored, but player-facing Chronicle rendering should show only what happened. |
| `campaign/threads/` | data / storage | Open or closed political/story threads. | Important / Risk Candidate. Drives side-panel records and older random-event pressure. |
| `campaign/messages/` | data / storage | Pending/delivered message records for the older game loop. | Used by `src/engine.js`; may be less active in current LLM flow. |
| `campaign/enforcement/` | data / storage | Enforcement records. Existing records are deprecated Bannerlord-shaped manual synchronization data; clean Enforcement v0 records are scoped laws, customs, obligations, and restrictions. | Important / Risk Candidate. Preserve legacy records, but clean v0 paths must skip deprecated Bannerlord fields and use exact typed-scope matching for LLM constraints. |
| `campaign/snapshots/` | runtime / rollback / storage | One-turn rollback snapshot files and `manifest.json`. | Runtime data. Snapshot behavior is intentionally limited to one turn. |
| `campaign/html/index.html` | runtime / generated HTML | Static dashboard generated by `src/html.js`. | Generated/legacy artifact, not the current live browser app. |

### Test And Utility Files

| Path | Category | Apparent responsibility | Notes before editing |
| --- | --- | --- | --- |
| `test/test-lmstudio-scene.js` | test / LLM | Manual test for the local LLM scene route using a sample campaign packet. | Calls the route module directly. Requires LM Studio/model availability for a full pass. |

### Current Application Flow

1. User starts the app from the project root with `npm start`.
2. `package.json` runs `node src/server.js`.
3. `src/server.js` serves `public/index.html`, `public/styles.css`, and `public/app.js`.
4. Browser loads `public/app.js`.
5. Browser polls `GET /api/chronicle` for current state and recent Chronicle records.
6. Player submits text in `#turn-input`.
7. `public/app.js` posts only `{ playerAction }` to `POST /api/llm/generate-scene`.
8. `src/server.js` loads campaign data and calls `src/campaignPacketBuilder.js` to build the temporary server-side `campaignPacket`.
9. `src/campaignPacketBuilder.js` assembles state, recent events, actors, threads, scoped enforcement constraints, units, message side-channel state, and temporary Bannerlord context.
10. `src/routes/llmRoutes.js` builds prompts, calls the configured provider through `src/llm/llmClient.js`, and validates the structured scene output.
11. On success, `src/server.js` calls `src/llmTurnAdapter.js`.
12. `src/llmTurnAdapter.js` creates a rollback snapshot, advances Bannerlord time, appends a Chronicle/event record under `campaign/events/`, and saves `campaign/state.json`.
13. Browser renders the active scene and choices, then refreshes Chronicle/state from the returned campaign summary.
14. On validation failure, the UI shows errors and the failed output is not committed to Chronicle.

### Chronicle Responsibilities

| Area | Path | Responsibility | Notes |
| --- | --- | --- | --- |
| Chronicle persistence | `src/llmTurnAdapter.js`, `src/store.js`, `campaign/events/` | Successful validated LLM turns are saved as event records. | Failed validation outputs should not be saved. |
| Chronicle API summary | `src/server.js` | Loads recent events, sorts by campaign time, and returns the summary to the browser. | Summary currently includes metadata that may be useful for debug/context, but player-facing rendering should stay in-world. |
| Chronicle rendering | `public/app.js` | Renders visible Chronicle entries in the browser. | Should display what happened: time/location, player input, scene prose. Choices are future possibilities, not Chronicle history. |
| Legacy generated Chronicle | `src/html.js` | Renders older static HTML dashboard Chronicle entries. | Not the live app's current rendering path. |

### Important / Risk Candidate Files

- `src/engine.js` - large file mixing legacy game loop, hardcoded scenes, time skips, character memory mutation, enforcement, random events, and Chronicle writes.
- `public/app.js` - mixes frontend state, API calls, temporary campaign packet construction, active scene rendering, choices, Chronicle rendering, polling, and rollback interaction.
- `src/server.js` - central backend route file; bridges static serving, LLM route, Chronicle summary, rollback, and legacy `/api/turn`.
- `src/store.js` - direct JSON persistence and rollback restoration; mistakes here can affect all campaign memory.
- `src/llmTurnAdapter.js` - commits accepted LLM output into canon/history and advances time.
- `src/routes/llmRoutes.js` - main local LLM flow and validation gate.
- `src/llm/promptBuilder.js` - prompt construction; changes can alter narrator voice and output quality.
- `src/llm/contentRules.js` - shared prompt and validation rules; changes can affect both generation and rejection behavior.
- `src/llm/validateSceneOutput.js` - hard validation and warning logic for generated scenes.
- `src/llm/entityValidation.js` - canon/entity control for locations, characters, clans, and factions.
- `src/llm/sceneSchema.js` - output contract shared by model, backend, and frontend expectations.
- `src/enforcementContextController.js` - separates clean Enforcement v0 projection from active scope-matched LLM binding constraints.
- `public/styles.css` - dense layout and scrolling behavior; changes can quickly affect playability.
- `campaign/state.json` and `campaign/events/` - live campaign canon and history.
- `campaign/characters/` - persistent character memory and relationship data.

### Known Issues Not Addressed

- Generated prose or Chronicle entries may echo the player's submitted action too directly.

### Notes For Future Development

- The broader product direction is a generic Roleplaying Companion, but this codebase is still Bannerlord-specific.
- Do not assume the older scripted `/api/turn` flow and the newer local LLM flow are equivalent; they write similar data but use different logic.
- The backend currently builds a temporary v1 campaign packet with some hardcoded Bannerlord context plus focused server-side retrieval helpers. Replacing the remaining hardcoded context with real memory retrieval should be a deliberate future pass.
- The app currently uses local JSON files as storage. No database or framework is present.
- The browser should continue to call only the Node backend. It should not call LM Studio or any cloud provider directly.
