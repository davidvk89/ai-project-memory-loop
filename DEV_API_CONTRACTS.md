# Current API Contracts And Trust Boundaries

## Purpose

This document describes the current API contracts and trust boundaries in the local Roleplaying Companion / Bannerlord prototype. It documents what exists now. It does not propose or implement fixes, schemas, middleware, data migration, prompt changes, UI changes, or future architecture.

The current app is a local Node server with a static browser frontend, local JSON campaign storage, and a backend-only LLM gateway. The browser must call the Node backend, not LM Studio directly.

## Endpoint Summary

| Endpoint | Method | Current/Legacy | Main Caller | Commits Data | Advances Time | Creates Snapshot | Calls LLM |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `/api/chronicle` | GET | Current | `public/app.js` polling | No | No | No | No |
| `/api/llm/generate-scene` | POST | Current | `public/app.js` player input submit | Yes, only when `ok: true` | Yes, only on successful commit | Yes, only on successful commit | Yes |
| `/api/rollback` | POST | Current | `public/app.js` rollback button | Yes, by restoring previous snapshot | Restores prior time | Consumes/removes snapshot | No |
| `/api/turn` | POST | Legacy but active | No current browser caller found | Yes | Usually yes, path-dependent | Yes | No |

## GET /api/chronicle

### Caller

- Called by `public/app.js` in `refreshChronicle()`.
- Runs immediately on page load and then every `state.pollMs` milliseconds.

### Request

- Method: `GET`
- Path: `/api/chronicle`
- Body: none.

### Success Response

Status: `200`

Returned by `summarizeCampaign(loadCampaign())` in `src/server.js`.

Representative shape:

```json
{
  "revision": "string",
  "state": {
    "campaignName": "string",
    "time": "Night, Summer 18, 1085",
    "location": "Akkalat",
    "treasury": 7000
  },
  "chronicle": [
    {
      "id": "string",
      "type": "llm_scene_turn",
      "title": "Local LLM Scene Turn",
      "time": "Night, Summer 18, 1085",
      "location": "Akkalat",
      "narration": "string",
      "involvedCharacters": [],
      "playerInput": "string",
      "choices": [],
      "continuityNotes": [],
      "memoryCandidates": [],
      "namedEntitiesUsed": null,
      "uncertainties": [],
      "validationWarnings": [],
      "model": null,
      "provider": null
    }
  ],
  "characters": [],
  "openThreads": [],
  "pendingEnforcement": [],
  "enforcementRecords": [],
  "rollback": {
    "available": true,
    "count": 1,
    "latest": {}
  }
}
```

### Failure Response

- General server catch returns status `500` with:

```json
{
  "error": "string"
}
```

Possible causes include malformed/missing campaign JSON, filesystem read errors, or summary logic errors.

### Reads

- `campaign/state.json`
- `campaign/characters/*.json`
- `campaign/threads/*.json`
- `campaign/messages/*.json`
- `campaign/events/*.json`
- `campaign/enforcement/*.json`
- `campaign/snapshots/manifest.json`

Read owner:
- `src/store.js` via `loadCampaign()`, `getSnapshots()`, `getCampaignRevision()`.

### Writes

- None.

### Validation

- No request validation needed beyond route/method.
- No schema validation of campaign JSON before summarizing.
- Chronicle sorting uses `compareFormattedTimes()` and falls back to string id comparison.

### Player-Facing Data

The endpoint returns more than Chronicle prose. Player-facing fields currently include:

- `state.campaignName`
- `state.time`
- `state.location`
- `state.treasury`
- `chronicle[].time`
- `chronicle[].location`
- `chronicle[].narration` or `chronicle[].scene` if present in the browser-side renderer
- `chronicle[].playerInput`
- `characters[].name`
- `characters[].rank`
- `characters[].clan`
- `characters[].faction`
- `characters[].status`
- `characters[].relationToPlayer`
- `characters[].lastSeen`
- `characters[].speechStyle`
- `openThreads[].title`
- `openThreads[].summary`
- `pendingEnforcement[].requiredAction`
- `pendingEnforcement[].reason`
- `enforcementRecords[].type`
- `enforcementRecords[].status`
- `enforcementRecords[].statement`
- `enforcementRecords[].requiredAction`
- `enforcementRecords[].reason`
- `enforcementRecords[].createdAt`
- `rollback.available`
- `rollback.count`

### Internal / Debug / Support Data

The response may also include support or technical data:

- `revision`
- `chronicle[].type`
- `chronicle[].title`
- `chronicle[].choices`
- `chronicle[].continuityNotes`
- `chronicle[].memoryCandidates`
- `chronicle[].namedEntitiesUsed`
- `chronicle[].uncertainties`
- `chronicle[].validationWarnings`
- `chronicle[].model`
- `chronicle[].provider`
- `rollback.latest`
- IDs across chronicle, characters, threads, and enforcement records.

### Enforcement Fields

`pendingEnforcement` is preserved for legacy compatibility. It is built from old Bannerlord-shaped records that are not `status: "enforced"` and retains the old `requiredAction` / `reason` projection used by the existing sidebar and legacy flows.

`enforcementRecords` is the clean Enforcement v0 feed projection. It contains clean v0 records only and is not active-scope filtered. Active, resolved, and retired clean records may appear here. Deprecated Bannerlord-shaped legacy records, invalid clean records, unknown extra fields, `suggestedBannerlordAction`, `bannerlordActionTaken`, and legacy string `authority` do not appear in this projection.

The local LLM generation path reads clean Enforcement v0 records through `src/enforcementContextController.js`, but it does not insert `enforcementRecords` wholesale into `campaignPacket.context`. LLM context receives only compact binding constraints from clean records that are `status: "active"` and whose typed scopes exactly match the current active scope set.

Current frontend rendering intentionally does not display most Chronicle technical metadata as in-world prose, but the endpoint still returns it.

### Trust Boundary Notes

- Campaign archive records become browser-visible data here.
- Local JSON files are treated as trusted server-side data.
- Technical/debug/support fields cross from backend to browser.
- Browser rendering decides which returned fields are player-facing.

### Risks

- The endpoint exposes mixed player-facing and technical data in one response.
- If campaign JSON is malformed, the endpoint can fail with a generic `500`.
- Chronicle records are heterogeneous; consumers must not assume every event has the same fields.
- Current summary includes choices and metadata even though choices are next-action possibilities rather than happened history.

## POST /api/llm/generate-scene

### Caller

- Called by `public/app.js` in `requestScene(playerAction)`.
- Triggered by submitting `#turn-input` through Enter or Send.
- Choice buttons copy a choice into `#turn-input`; they do not directly call the endpoint.

### Request

Method: `POST`

Path: `/api/llm/generate-scene`

Expected body:

```json
{
  "playerAction": "string"
}
```

Current frontend request body is built in `public/app.js` by `requestScene(playerAction)`. The browser no longer sends an authoritative `campaignPacket`.

`src/server.js` validates `playerAction`, loads campaign state/history, builds a State Boundary, runs the temporal input guard, and then builds a temporary server-side `campaignPacket` using `src/campaignPacketBuilder.js`. This builder preserves the current temporary Bannerlord context, uses `campaign.state.location` as `campaignPacket.currentLocation`, includes formatted `campaign.state.currentTime` in the LLM-facing context, adds a compact recency-only slice of up to 3 recent committed event records, adds compact Actors Context from `campaign/actors` plus legacy `campaign/characters` records, adds compact Open Threads context selected by `src/threadContextController.js`, adds compact Units Context from valid `campaign/units/*.json` records, and adds `sideChannels.messages.hasAvailable` as a binary mailbox side-channel. Message contents are not inserted into `campaignPacket.context`.

### Success Response

The LLM route returns status `200` with:

```json
{
  "ok": true,
  "data": {
    "scene": "string",
    "choices": ["string", "string", "string"],
    "continuityNotes": ["string"],
    "memoryCandidates": ["string"],
    "namedEntitiesUsed": {
      "locations": ["string"],
      "majorCharacters": ["string"],
      "minorCharacters": ["string"],
      "clans": ["string"],
      "factions": ["string"]
    },
    "unsupportedInventions": [],
    "uncertainties": ["string"]
  },
  "validation": {
    "passed": true,
    "hardErrors": [],
    "warnings": ["string"]
  },
  "meta": {
    "provider": "lmstudio",
    "model": "qwen/qwen3-14b",
    "latencyMs": 0,
    "inputChars": 0,
    "outputChars": 0,
    "usage": null
  },
  "turn": {
    "eventId": "string",
    "timeBefore": "Evening, Summer 18, 1085",
    "timeAfter": "Night, Summer 18, 1085",
    "timeAdvancedBlocks": 1
  },
  "campaign": {}
}
```

Important: `turn` and `campaign` are added by `src/server.js` after `src/routes/llmRoutes.js` returns `ok: true` and `commitValidatedLlmTurn()` runs.

### Failure Response

Invalid campaign packet or failed model validation returns status `422`:

```json
{
  "ok": false,
  "error": "Scene output failed validation.",
  "validation": {
    "passed": false,
    "hardErrors": ["string"],
    "warnings": ["string"]
  },
  "rawPreview": "string",
  "meta": {
    "provider": "lmstudio",
    "model": "qwen/qwen3-14b",
    "latencyMs": 0,
    "inputChars": 0,
    "outputChars": 0,
    "usage": null
  }
}
```

Provider/LLM connection errors return status `503` with a similar body:

```json
{
  "ok": false,
  "error": "string",
  "validation": {
    "passed": false,
    "hardErrors": ["string"],
    "warnings": []
  },
  "rawPreview": "",
  "meta": {}
}
```

Malformed JSON, missing `playerAction`, non-string `playerAction`, empty `playerAction`, overlong `playerAction`, explicit calendar-date assertions, or time jumps over seven days return status `400`:

```json
{
  "ok": false,
  "error": "Invalid playerAction.",
  "validation": {
    "passed": false,
    "hardErrors": ["string"],
    "warnings": []
  },
  "rawPreview": "",
  "meta": null
}
```

### Reads

Generation/validation:
- `.env` through `src/llm/llmClient.js`
- `campaign/state.json` is read so the server can build the temporary `campaignPacket` with the current campaign time and location.

Commit after success:
- `campaign/state.json`
- `campaign/characters/*.json`
- `campaign/threads/*.json`
- `campaign/messages/*.json`
- `campaign/events/*.json`
- `campaign/enforcement/*.json`
- `campaign/snapshots/manifest.json`

### Writes

Only when `result.body?.ok` is true and `commitValidatedLlmTurn()` accepts the result:

- Creates/overwrites one rollback snapshot in `campaign/snapshots/`.
- Appends one event JSON to `campaign/events/`.
- Updates `campaign/state.json`.
- Updates `campaign/snapshots/manifest.json`.

It does not currently update:
- `campaign/characters/`
- `campaign/threads/`
- `campaign/messages/`
- `campaign/enforcement/`

### Validation

Request body validation in `src/server.js`:

- Request body must be a JSON object.
- `playerAction` is required.
- `playerAction` must be a string.
- Trimmed `playerAction` must not be empty.
- Trimmed `playerAction` must be 4,000 characters or fewer.
- Malformed JSON for this route returns a structured `400`.
- Temporal guard rejects explicit Bannerlord calendar-date assertions and time jumps over `stateBoundary.maxTimePassedDays`.
- Rejected temporal input does not reach `campaignPacketBuilder`, the LLM, or commit.

Server-built packet validation:

- `campaignPacket` must be an object.
- `currentLocation`, `context`, and `playerAction` must be non-empty strings.
- `relevantLocations`, `allowedLocations`, `relevantMajorCharacters`, `allowedMajorCharacters`, `allowedClans`, and `allowedFactions` must be arrays of strings.
- `currentLocation` must be included in `allowedLocations`.

LLM output validation:

- Raw provider content must parse as JSON.
- Parsed output must match the expected schema shape.
- Top-level unexpected fields are rejected.
- Required top-level fields are required.
- `choices` must contain exactly 3 strings.
- `namedEntitiesUsed` must include locations, majorCharacters, minorCharacters, clans, and factions arrays.
- `scene` must be narrative prose, not serialized JSON or schema-shaped metadata.
- Entity validation checks named locations, major characters, clans, and factions against allowed/relevant packet fields.
- Minor characters that do not collide with major full names are warnings.
- Banned content regex patterns are hard errors.
- Invented-canon regex patterns are hard errors.
- Non-empty `unsupportedInventions` is a hard error.
- Non-empty `uncertainties`, empty `memoryCandidates`, sparse `continuityNotes`, and short scene prose are warnings, not hard blockers.

### Commit Behavior

Commit is performed in `src/server.js` after the route handler returns:

1. `src/server.js` parses and validates `{ playerAction }`.
2. `src/server.js` loads campaign state and builds a State Boundary.
3. `validateTemporalPlayerAction()` checks player input against the State Boundary.
4. `src/server.js` builds a temporary server-side `campaignPacket`.
5. `validateCampaignPacket(campaignPacket)` confirms the generated packet still satisfies the existing LLM route contract.
6. `handleGenerateScene({ campaignPacket })` returns a result.
7. `src/server.js` checks `if (result.body?.ok)`.
8. `commitValidatedLlmTurn({ campaignPacket, llmBody: result.body })` is called.
9. `commitValidatedLlmTurn()` checks `llmBody?.ok && llmBody?.validation?.passed`.
10. Snapshot is created with reason `before_llm_scene_turn`.
11. Current campaign is loaded.
12. Time advances by `estimateBlocks(campaignPacket.playerAction)`.
13. Location is set to `campaignPacket.currentLocation` if present.
14. A `llm_scene_turn` event is appended.
15. Campaign state is saved.
16. The response gets `turn` metadata and a fresh `campaign` summary.

This is where validated LLM output becomes campaign truth.

### Failure Behavior

- If playerAction validation fails, no packet is built, no LLM call is made, and no campaign data is committed.
- If temporal input validation fails, no packet is built, no LLM call is made, and no campaign data is committed.
- If server-side packet building or packet validation fails, no LLM call is made and no campaign data is committed.
- If provider call fails, no campaign data is committed.
- If JSON parsing/schema validation fails after repair attempt, no campaign data is committed.
- If content/entity validation fails, no campaign data is committed.
- If `commitValidatedLlmTurn()` receives a non-passed body, it returns `null` and does not commit.

### Player-Facing Data

- `data.scene`
- `data.choices`
- `validation.warnings` as compact validation/debug notes
- `campaign` summary after commit
- `turn.timeBefore`, `turn.timeAfter` may be useful status/support data.

### Internal / Debug / Support Data

- `meta.provider`
- `meta.model`
- `meta.latencyMs`
- `meta.inputChars`
- `meta.outputChars`
- `meta.usage`
- `validation.hardErrors`
- `validation.warnings`
- `rawPreview` on failure
- `namedEntitiesUsed`
- `continuityNotes`
- `memoryCandidates`
- `unsupportedInventions`
- `uncertainties`
- `turn.eventId`

### Trust Boundary Notes

- Browser input enters the backend as `playerAction`.
- The `campaignPacket` is now server-built before prompt construction and entity validation.
- The server-built packet is temporary and still uses hardcoded v1 Bannerlord context and entity lists.
- User input enters prompt construction in `promptBuilder.js`.
- LLM output enters validation in `llmRoutes.js`.
- Validated LLM output enters campaign truth through `llmTurnAdapter.js`.
- Technical metadata returns to the browser in both success and failure cases.

### Risks

- The browser no longer sends `campaignPacket`, but the server-built packet still uses temporary hardcoded context and entity lists.
- Current local LLM flow does not yet select full campaign memory from the filesystem before prompting.
- `campaignPacket.currentLocation` now comes from `campaign.state.location`, then can be written back during commit.
- `estimateBlocks()` is keyword-based and uses raw player action text.
- `rawPreview` may return part of failed model output to the browser.
- Success responses include both player-facing scene data and technical/debug metadata.
- Malformed JSON for this route is now structured as `400`, but unrelated routes still use generic server catch behavior.

## POST /api/rollback

### Caller

- Called by `public/app.js` in `rollbackLastTurn()`.
- Triggered by the Rollback Last Turn button after browser confirmation.

### Request

- Method: `POST`
- Path: `/api/rollback`
- Body currently sent by frontend:

```json
{}
```

No snapshot id is accepted by the current route. It restores the latest available snapshot.

### Success Response

Status: `200`

```json
{
  "restored": {
    "id": "snapshot_...",
    "createdAt": "2026-06-28T13:21:42.655Z",
    "reason": "before_llm_scene_turn",
    "input": "string"
  },
  "campaign": {}
}
```

`campaign` is the current summary after restore.

### Failure Response

No snapshot:

Status: `400`

```json
{
  "error": "No rollback snapshot is available."
}
```

Other errors, such as a missing snapshot file listed in the manifest:

Status: `500`

```json
{
  "error": "Snapshot file is missing: snapshot_id"
}
```

### Reads

- `campaign/snapshots/manifest.json`
- Selected `campaign/snapshots/snapshot_*.json`
- Restored campaign collections inside the snapshot
- Current campaign after restore for summary response

### Writes

Through `src/store.js`:

- Overwrites `campaign/state.json`
- Clears and rewrites `campaign/characters/*.json`
- Clears and rewrites `campaign/threads/*.json`
- Clears and rewrites `campaign/messages/*.json`
- Clears and rewrites `campaign/events/*.json`
- Clears and rewrites `campaign/enforcement/*.json`
- Deletes restored snapshot file
- Updates `campaign/snapshots/manifest.json`

### Validation

- No request-body validation beyond route/method.
- No validation of snapshot campaign shape before restore.
- `restoreSnapshot()` verifies that a manifest target exists and that the snapshot file exists.

### Restore Behavior

- Restores the latest snapshot if no id is supplied internally.
- Current route does not expose snapshot id selection.
- Snapshot restore overwrites live campaign data.
- Snapshot is consumed/deleted after restore.

### Trust Boundary Notes

- Runtime snapshot data overwrites live campaign truth.
- Snapshot data is treated as trusted local server-side data.
- Browser can trigger restore, but does not provide the restored data.

### Risks

- Restoring a malformed or stale snapshot can overwrite all live collections.
- No schema validation is performed on snapshot contents before writing.
- The restore operation clears and rewrites whole folders.
- The UI confirmation is frontend-only UX; backend does not require a confirmation token.

## POST /api/turn Legacy Route

### Caller

- Legacy route in `src/server.js`.
- Current `public/app.js` does not call it.
- It may still be called manually or by older tooling.

### Request

Method: `POST`

Path: `/api/turn`

Expected body:

```json
{
  "input": "string"
}
```

### Success Response

Status: `200`

```json
{
  "result": {
    "header": "Night, Summer 1, 1085 -- Akkalat",
    "narration": "string",
    "totalTurnTokenEstimate": null
  },
  "campaign": {}
}
```

### Failure Response

Empty input:

Status: `400`

```json
{
  "error": "Input is required."
}
```

Malformed JSON or runtime errors:

Status: `500`

```json
{
  "error": "string"
}
```

### Reads

- `campaign/state.json`
- `campaign/characters/*.json`
- `campaign/threads/*.json`
- `campaign/messages/*.json`
- `campaign/events/*.json`
- `campaign/enforcement/*.json`
- `campaign/snapshots/manifest.json`

### Writes

Path-dependent. `src/engine.js` may write:

- `campaign/snapshots/*`
- `campaign/state.json`
- `campaign/events/*.json`
- `campaign/characters/*.json`
- `campaign/threads/*.json`
- `campaign/messages/*.json`
- `campaign/enforcement/*.json`
- `campaign/html/index.html`
- Workspace `outputs/bannerlord_companion_dashboard.html` through `src/html.js`

### Validation

- Route checks only that `input` is non-empty after trimming.
- `processTurn()` contains hardcoded branch checks for exact choices or keyword matches.
- No formal request schema.
- No LLM validation because this route does not call the LLM.

### Commit Behavior

`processTurn(input)` in `src/engine.js`:

- Creates a snapshot with reason `before_player_input`.
- Loads campaign data.
- Checks multiple hardcoded legacy resolvers:
  - Manteos fine payment choice
  - explicit time skip
  - Player one / Mesui choice one
  - forced castle hall event
  - castle hall choice five
  - Player one / Mesui meeting
- If no special resolver matches:
  - estimates time blocks from input
  - advances campaign time
  - may update location
  - delivers due messages
  - rolls a semi-random event
  - writes a `gm_turn` event
  - applies character memory updates
  - may create enforcement
  - saves campaign state
  - regenerates the legacy dashboard

### Trust Boundary Notes

- Browser or external input can enter legacy campaign mutation with only non-empty validation.
- Legacy logic treats input text as a driver for branch selection, time advancement, and character updates.
- This route can directly alter campaign truth without LLM validation.

### Risks

- Legacy but active route can still mutate campaign data.
- Large `src/engine.js` mixes routing-adjacent game logic, story content, persistence, memory updates, enforcement, and dashboard generation.
- Hardcoded example scenes can affect campaign canon if the route is called.
- Route response shape differs from `/api/llm/generate-scene`.

## LLM Route / Validation Notes

The `/api/llm/generate-scene` flow has two distinct phases:

1. Generation and validation in `src/routes/llmRoutes.js` and `src/llm/*`.
2. Server-side commit in `src/server.js` and `src/llmTurnAdapter.js`.

Flow:

1. Browser submits `{ playerAction }`.
2. `src/server.js` parses JSON and validates `playerAction`.
3. `src/server.js` loads campaign state and builds a State Boundary.
4. `src/server.js` rejects unsafe temporal input before packet construction.
5. `src/server.js` builds a temporary server-side `campaignPacket`.
6. `src/server.js` validates the generated packet with `validateCampaignPacket()`.
7. `src/server.js` calls `handleGenerateScene({ campaignPacket })`.
8. `src/routes/llmRoutes.js` validates the campaign packet shape again before prompt construction.
9. `promptBuilder.js` builds system/user prompts.
10. `llmClient.js` calls the configured OpenAI-compatible provider.
11. `llmRoutes.js` parses raw model output as JSON.
12. `validateSceneOutput.js` checks schema, content, unsupported inventions, and warnings.
13. `entityValidation.js` checks named entities against packet allow/relevance lists.
14. If validation passes, response body has `ok: true`.
15. `src/server.js` commits via `commitValidatedLlmTurn()`.
16. If validation fails, response body has `ok: false` and no commit is performed.

The model output becomes trusted enough to commit only after:

- provider response exists,
- content parses as JSON,
- schema validation passes,
- entity/content validation passes,
- `unsupportedInventions` is empty,
- `validation.passed` is true,
- `commitValidatedLlmTurn()` confirms `llmBody.ok` and `llmBody.validation.passed`.

## LLM Stack File Roles

| File | Input received | Output produced | Validates / assumes | Handles user input | Handles campaign memory | Handles LLM output | Can block bad result | Data returned |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `src/server.js` | Parsed `{ playerAction }`; server-loaded campaign state/history | State Boundary; server-built temporary `campaignPacket`; final HTTP response | Validates `playerAction`, malformed JSON, temporal guard, generated packet contract, commit gate | Yes, directly from browser request | Reads campaign state time/location and passes event/actor/thread/unit/message side-channel history for packet construction | Receives validated route result | Yes | Player-facing scene plus validation/meta/campaign summary |
| `src/stateBoundary.js` | `campaignState` | State Boundary object | Validates minimal state/time/location and legacy/current status fields | No | Uses campaign state only | No | Yes, by throwing on broken state | Internal boundary only |
| `src/temporalInputGuard.js` | `playerAction`, State Boundary | `{ ok: true }` or structured rejection | Blocks explicit calendar assertions and time jumps over `maxTimePassedDays` | Yes | Uses State Boundary guardrails only | No | Yes | Internal validation result plus user-facing rejection message |
| `src/campaignPacketBuilder.js` | `campaignState`, recent `events`, merged actor/legacy character records, selected open threads, valid unit records, clean enforcement records, message side-channel inputs, `playerAction` | Temporary v1 `campaignPacket` | Requires current campaign location and `currentTime`; assumes temporary hardcoded Bannerlord context/entities | Yes, inserts `playerAction` into packet | Uses `campaignState.location`, `campaignState.currentTime`, compact recent-events context, compact actor context, Thread Context Controller v0 output, clean scope-matched Enforcement v0 binding constraints, compact Unit v0 context, and binary message side-channel state; no full memory retrieval | No | Yes, by throwing if state/time/location invalid | Internal packet only |
| `src/enforcementContextController.js` | `campaign/enforcement/*.json`, `campaign.state.activeScopes` if present | Clean `enforcementRecords` feed projection and compact active binding constraints | Validates clean Enforcement v0 schema, classifies deprecated legacy records, extracts active scopes, exact-matches typed scopes | No | Reads enforcement records and campaign active scopes only | No | Yes, by excluding deprecated/invalid/nonmatching/inactive records from LLM context | Clean API projection plus internal context text |
| `src/routes/llmRoutes.js` | Server-built `campaignPacket`; raw provider results | HTTP-like `{ statusCode, body }` result | Validates packet shape, JSON parsing, schema errors, final scene validation | Yes, through `campaignPacket.playerAction` | Only whatever is inside server-built temporary `campaignPacket` | Yes | Yes | Player-facing scene plus validation/meta/rawPreview |
| `src/llm/llmClient.js` | Prompt messages, response format, repair flag | `{ raw, meta }` from provider | Validates required `.env`; provider HTTP status; provider JSON; `choices[0].message.content` string | Indirectly, as prompt text | No | Receives raw provider payload | Yes, by throwing provider/shape errors | Internal raw model text and technical meta |
| `src/llm/promptBuilder.js` | `campaignPacket`, optional repair instruction | `{ systemPrompt, userPrompt }` | Assumes packet fields exist after route validation | Yes, inserts `playerAction` into user prompt | Only packet-provided context and entities | No | No | Internal prompt strings |
| `src/llm/sceneSchema.js` | None at runtime except imports | `sceneJsonSchema`, `sceneResponseFormat` | Defines required output contract | No | No | Indirectly constrains provider output | Yes, through provider response format and later schema validation | Internal schema/config |
| `src/llm/validateSceneOutput.js` | Parsed model output and packet | `{ passed, hardErrors, warnings }` | Validates object shape, arrays, required fields, banned patterns, invented canon patterns, unsupported inventions | Indirectly via model output and packet | Only packet data | Yes | Yes | Validation result |
| `src/llm/entityValidation.js` | Parsed model output and packet | `{ hardErrors, warnings }` | Checks named entities against allowed/relevant packet arrays | Indirectly | Only packet data | Yes | Yes | Validation details |
| `src/llm/contentRules.js` | None; imported constants and helper | Rule arrays and `combinedSceneText()` | Defines prompt rules and regex hard-error patterns | No | No | Combines output text for validators | Yes, through validators using patterns | Internal rules/helper |

## Trust Boundary Map

1. Browser input entering backend.
   - Boundary: `#turn-input` to `POST /api/llm/generate-scene`, and legacy body `input` to `/api/turn`.
   - Current control: frontend trims empty input for UX; backend validates `playerAction` for the LLM route and non-empty input for legacy route.
   - Gap: raw player intent still enters prompt construction; legacy route has minimal validation.

2. Server-built temporary `campaignPacket` entering the LLM route.
   - Boundary: `src/server.js` / `src/campaignPacketBuilder.js` to `src/routes/llmRoutes.js`.
   - Current control: generated packet is checked with `validateCampaignPacket()`.
   - Gap: packet still uses temporary hardcoded context and entity lists; full memory retrieval remains future work.

3. User input entering prompt or context construction.
   - Boundary: `campaignPacket.playerAction` to `promptBuilder.js`; legacy `input` to `context.js` in some scripted paths.
   - Current control: non-empty string checks.
   - Gap: no prompt-injection filtering or intent validation beyond output validation.

4. Campaign memory being selected for prompt/context.
   - Current LLM route: server reads campaign state, recent events, actors, threads, units, message side-channel state, and clean scope-matched Enforcement v0 binding constraints.
   - Legacy route: `src/context.js` selects characters, threads, recent events, and enforcement based on provided options.
   - Gap: future server-side retrieval will need richer authoritative selection rules. Enforcement v0 exact scope matching deliberately does not infer hierarchy or authority.

5. LLM output entering validation.
   - Boundary: `llmClient.js` raw provider content to `llmRoutes.js`.
   - Current control: JSON parse, schema shape, entity/content validation.
   - Gap: validators cannot prove all narrative facts are true, only enforce current packet and pattern constraints.

6. Validated LLM output entering campaign truth.
   - Boundary: `result.body.ok === true` to `commitValidatedLlmTurn()`.
   - Current control: `commitValidatedLlmTurn()` checks `llmBody.ok` and `llmBody.validation.passed`.
   - Gap: committed record still includes technical metadata and future choices in storage.

7. Campaign archive records becoming player-facing Chronicle output.
   - Boundary: `campaign/events/*.json` to `/api/chronicle` to `public/app.js`.
   - Current control: frontend Chronicle renderer displays only heading/input/scene prose.
   - Gap: API still returns technical metadata to the browser.

8. Rollback snapshot data overwriting live campaign data.
   - Boundary: `campaign/snapshots/snapshot_*.json` to live `campaign/*`.
   - Current control: snapshot must exist.
   - Gap: no schema validation before overwrite; restore rewrites whole collections.

9. Legacy `/api/turn` changing campaign data.
   - Boundary: external POST body to `processTurn()`.
   - Current control: non-empty input only.
   - Gap: legacy route remains active and can mutate state, characters, events, threads, messages, enforcement, snapshots, and generated dashboard.

10. Technical/debug/support metadata being returned to the browser.
    - Boundary: backend responses to `public/app.js`.
    - Current control: frontend chooses what to render prominently.
    - Gap: API includes model/provider/meta, validation data, rawPreview on failures, entity lists, and stored event support fields.

## Validation Map

| Area | Current validation | Missing or weak validation |
| --- | --- | --- |
| JSON request parsing | `/api/llm/generate-scene` has structured malformed JSON handling; other route blocks use `JSON.parse()` with generic catch | Other routes do not have structured 400 for malformed JSON |
| `/api/chronicle` request | Route/method only | No campaign data shape validation before response |
| `/api/llm/generate-scene` request | Body object plus string, non-empty, max 4,000 character `playerAction` | No prompt-injection filtering or semantic intent validation |
| `/api/llm/generate-scene` temporal guard | Blocks explicit calendar-date assertions and time jumps over 7 days before packet construction | Conservative pattern matching only; not a full natural-language parser or time-skip planner |
| `/api/llm/generate-scene` packet | Server-built packet object, required string fields, required string arrays, currentLocation in allowedLocations | Temporary packet uses only campaign state location, not full server-side campaign memory |
| Prompt construction | Assumes validated packet | No prompt-injection filtering; relies on output validation |
| Provider response | HTTP status, response JSON, `choices[0].message.content` string | No provider-specific deeper semantic validation |
| Model JSON | Parse as JSON; repair once on schema failure | Repair is limited; failed raw output preview can return to browser |
| Scene schema | Required fields, no extra top-level fields, array/string checks, exactly 3 choices | Does not validate narrative truth |
| Entity validation | Allowed/relevant locations and major characters, allowed clans/factions, minor name collision | Relies on server-built allowed/relevant packet lists; actor names/aliases are derived only from server-side actor/legacy character records |
| Content validation | Banned content regex, invented-canon regex, unsupported inventions empty | Regex patterns are partial and cannot catch every unsupported fact |
| Commit gate | `ok` and `validation.passed` checked before commit | Commit trusts packet location/action for time and location update |
| Rollback | Snapshot target/file must exist | No snapshot schema validation before restore |
| Legacy `/api/turn` | Non-empty input | No formal schema or trust checks |

## Commit Behavior Map

| Endpoint | Snapshot | State | Events | Characters | Threads | Messages | Enforcement | LLM |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `GET /api/chronicle` | No | Read only | Read only | Read only | Read only | Read only | Read only | No |
| `POST /api/llm/generate-scene` success | Creates one-turn snapshot | Advances time; may update location | Appends `llm_scene_turn` | No | No | No | No | Yes |
| `POST /api/llm/generate-scene` failure | No | No | No | No | No | No | No | Attempted or skipped depending on failure point |
| `POST /api/rollback` success | Consumes/removes snapshot | Restores | Restores | Restores | Restores | Restores | Restores | No |
| `POST /api/turn` legacy | Creates one-turn snapshot | Path-dependent updates | Appends events | May update | May create | May deliver | May create/resolve | No |

## Player-Facing vs Internal Data

Player-facing data:

- Active scene prose.
- Generated choices.
- Chronicle heading, player input, and scene/narration.
- Campaign time, location, treasury.
- Known characters summary.
- Open thread titles/summaries.
- Pending enforcement requirements.
- Hard validation errors when generation fails.
- Compact warning indicator/details.

Internal/debug/support data:

- Provider/model/latency/input/output usage metadata.
- Validation warnings and hard error arrays.
- `rawPreview` on failed model output.
- `namedEntitiesUsed`.
- `continuityNotes`.
- `memoryCandidates`.
- `unsupportedInventions`.
- `uncertainties`.
- `turn.eventId`.
- Rollback latest snapshot metadata.
- Event `type`, technical titles, and IDs.

Current boundary:

- `/api/chronicle` and `/api/llm/generate-scene` can return both player-facing and internal/support data.
- `public/app.js` decides what is shown prominently.
- Debug/support data is not currently separated into a different endpoint.

## Legacy Flow Notes

- `/api/turn` is legacy but active.
- It calls `processTurn()` in `src/engine.js`.
- It does not call the local LLM.
- It uses hardcoded scene/choice/time-skip logic.
- It can create snapshots, advance time, write events, update character memories/relationships, deliver messages, create threads, create or resolve enforcement, update state, and regenerate the legacy dashboard.
- It uses `src/context.js` for compact context/token estimates in some legacy choice-resolution branches.
- It has much weaker validation than `/api/llm/generate-scene`.
- Removing or disabling it is out of scope for this documentation task.

## Known Risks

- Browser no longer sends `campaignPacket`, but the server-built temporary packet still uses hardcoded v1 context and entity lists.
- Current LLM route writes to campaign memory but only reads campaign state time/location for prompt context; it does not yet build its prompt from full campaign memory.
- Temporal input guard is conservative and not a full parser, scheduler, or long time-skip simulator.
- Raw player input reaches prompts.
- Output validation is necessary but not a full truth engine.
- Successful LLM scene records store choices and technical metadata alongside Chronicle content.
- `/api/chronicle` returns summarized campaign view, not only Chronicle prose.
- `/api/rollback` can overwrite all live campaign collections from a snapshot without schema validation.
- Legacy `/api/turn` remains active and can mutate campaign data with only minimal request validation.
- Technical/debug/support metadata crosses to the browser.
- Generated prose or Chronicle entries may echo the player's submitted action too directly.

## Recommended Next Task

Recommended next task brief: CP-0006 - document the current local LLM scene-generation sequence as a step-by-step flow diagram, including where campaign-packet construction should eventually move from frontend hardcoded context to backend memory retrieval. This should remain documentation-only unless explicitly promoted to an implementation pass.
