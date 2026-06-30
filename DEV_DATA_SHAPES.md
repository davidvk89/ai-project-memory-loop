# Current Data Shapes

This document maps the current stored JSON/file-system data shapes for the Bannerlord companion prototype. It is descriptive only. It does not design a future data model, normalize existing records, or change application behavior.

The broader product direction is a generic Roleplaying Companion, but the current data is Bannerlord-specific and stored as local JSON under `campaign/`.

## Storage Overview

| Path | Classification | Purpose | Main read/write owners |
| --- | --- | --- | --- |
| `campaign/state.json` | Campaign truth/canon | Current campaign clock, location, treasury, random seed, active threads, and active scene. | Read/write: `src/store.js`; modified by `src/llmTurnAdapter.js` and `src/engine.js`; summarized by `src/server.js`; rendered by `public/app.js`. |
| `campaign/characters/` | Character memory / campaign truth/canon | Persistent dossiers for known characters and the player-character representative. | Read/write: `src/store.js`; modified by `src/engine.js`; read by `src/server.js`, `src/context.js`, `src/html.js`. |
| `campaign/events/` | Player-facing Chronicle/history / campaign truth/canon | Historical event records from imports, scripted turns, choices, time skips, and local LLM scene turns. | Read/write: `src/store.js`; written by `src/llmTurnAdapter.js` and `src/engine.js`; summarized by `src/server.js`; rendered by `public/app.js`. |
| `campaign/threads/` | Open thread/story state | Active or historical political/story threads. | Read/write: `src/store.js`; written by `src/engine.js`; read by `src/server.js`, `src/context.js`, `src/html.js`. |
| `campaign/messages/` | Legacy scripted-flow data | Pending/delivered messages used by the older scripted loop. | Read/write: `src/store.js`; updated by `src/engine.js`; read by `src/html.js`. |
| `campaign/enforcement/` | Enforcement/manual synchronization data | Manual Bannerlord actions needed to keep game state aligned with companion-app canon. | Read/write: `src/store.js`; written/updated by `src/engine.js`; read by `src/server.js`, `src/context.js`, `src/html.js`. |
| `campaign/snapshots/` | Runtime/rollback support | One-turn rollback snapshot and manifest. | Read/write: `src/store.js`; created by `src/llmTurnAdapter.js` and `src/engine.js`; restored by `src/server.js` via `restoreSnapshot()`. |
| `campaign/html/index.html` | Runtime/generated support | Legacy generated static dashboard. | Written by `src/html.js`; not the current live browser app. |

## Data Area: Campaign State

Path: `campaign/state.json`

Purpose: Stores the current campaign-wide state. This is the main campaign truth for time, location, money, random seed, and any currently active scripted scene.

Classification:
- Campaign truth/canon.
- Open scene/story state.
- Bannerlord-specific time and location state.

Representative shape:

```json
{
  "campaignName": "Test Campaign",
  "currentTime": {
    "block": "Night",
    "season": "Summer",
    "day": 18,
    "year": 1085
  },
  "location": "Akkalat",
  "treasury": 7000,
  "activeThreads": ["thread_recruitment_rights"],
  "randomSeed": 399654,
  "currentScene": {
    "id": "scene_bad_trade_season",
    "title": "Bad Trade Season",
    "involvedCharacters": ["player_one"],
    "openedAt": "Sundown, Summer 17, 1085",
    "location": "On the Calradian road",
    "choices": {
      "1": "Return to court and seek political work to repair the loss."
    }
  }
}
```

Apparent required fields:
- `campaignName`
- `currentTime.block`
- `currentTime.season`
- `currentTime.day`
- `currentTime.year`
- `location`
- `treasury`
- `randomSeed`

Apparent optional or flow-specific fields:
- `activeThreads`
- `currentScene`
- `currentScene.choices`

Read/write owner files:
- `src/store.js`: reads/writes the whole state file through `loadCampaign()` and `saveState()`.
- `src/llmTurnAdapter.js`: advances `currentTime`, updates `location` from the submitted campaign packet, saves state.
- `src/engine.js`: legacy scripted flow updates `currentTime`, `location`, `treasury`, `randomSeed`, and `currentScene`.
- `src/server.js`: summarizes state for `/api/chronicle`.
- `public/app.js`: reads summarized state from the API and displays time, location, treasury, and campaign name.

Bannerlord-specific assumptions:
- Time uses Bannerlord-style blocks and seasons: `Dawn`, `Morning`, `Midday`, `Sundown`, `Evening`, `Night`; `Summer`, `Autumn`, `Winter`, `Spring`; 21 days per season.
- `location` stores a Bannerlord place or campaign-scene location.
- `treasury` is in denars.

Notes before editing:
- `currentTime` is expected by `src/calendar.js`.
- `currentScene` currently belongs mostly to legacy scripted scenes, not the new local LLM generation path.
- Changing this shape affects Chronicle sorting, UI display, time advancement, and rollback snapshots.

## Data Area: Characters

Path: `campaign/characters/*.json`

Purpose: Persistent character dossiers. These records carry Bannerlord facts, roleplay interpretation, relationships, recent history, and durable character memory.

Classification:
- Character memory.
- Campaign truth/canon.
- Bannerlord-specific character dossiers.

Representative shape:

```json
{
  "id": "mesui",
  "name": "Mesui",
  "culture": "Khuzait",
  "clan": "Khergit",
  "faction": "Khuzaits",
  "rank": "clan leader",
  "occupation": "Lady",
  "status": "active",
  "age": 52,
  "relationToPlayer": 3,
  "lastSeen": "Castle Hall of Ortysia",
  "source": {
    "type": "bannerlord_encyclopedia_screenshot",
    "capturedFrom": "Home / Heroes / Mesui"
  },
  "firstMet": {
    "time": "Winter 12, 1086",
    "location": "unknown",
    "note": "Bannerlord encyclopedia states Mesui has met Dawud."
  },
  "speechStyle": "hard, direct, grievance-led...",
  "traits": ["ruthless defender of clan rights"],
  "relationship": {
    "trust": 1,
    "respect": 3,
    "resentment": 0,
    "debt": 0
  },
  "knownMotives": ["restore Khergit standing"],
  "bannerlordFacts": {
    "titleLine": "Vassal of the Khuzaits",
    "biography": "Mesui is leader of the Khergit...",
    "knownFriends": ["Chambui"],
    "knownEnemies": ["Caladog"],
    "visibleSkillValues": [165, 68]
  },
  "recentHistory": [
    {
      "time": "Winter 12, 1086",
      "summary": "Mesui has met Dawud."
    }
  ],
  "memories": [
    {
      "id": "mem_mesui_khergit_grievance",
      "weight": "defining",
      "summary": "Mesui believes the Khergit have shed blood...",
      "emotion": "old grievance sharpened into political purpose",
      "publicPosition": "defense of Khergit rights and honor",
      "privateMotive": "force the khanate to acknowledge..."
    }
  ]
}
```

Apparent required fields:
- `id`
- `name`
- `culture`
- `clan`
- `faction`
- `rank`
- `occupation`
- `status`
- `relationToPlayer`
- `lastSeen`
- `speechStyle`
- `traits`
- `relationship`
- `knownMotives`
- `bannerlordFacts`
- `recentHistory`
- `memories`

Apparent optional or nullable fields:
- `age` can be `null` for prototype-created characters.
- `source` and `firstMet` are present in current records, but their contents vary by source.
- `bannerlordFacts.knownFriends`, `knownEnemies`, and `visibleSkillValues` may be empty arrays.

Nested field notes:
- `relationship` currently uses numeric `trust`, `respect`, `resentment`, and `debt`.
- `memories` use `id`, `weight`, `summary`, `emotion`, `publicPosition`, and `privateMotive`.
- `recentHistory` uses `time` and `summary`.

Read/write owner files:
- `src/store.js`: reads all character JSON and writes individual character files through `saveCharacter()`.
- `src/engine.js`: modifies memories, relationships, `lastSeen`, and `recentHistory` during legacy scripted turns.
- `src/context.js`: reads compact character fields and last memories for legacy context packets.
- `src/server.js`: summarizes a smaller character view for the browser side panel.
- `src/html.js`: renders characters into the legacy static dashboard.

Bannerlord-specific assumptions:
- Character records include Bannerlord concepts such as culture, clan, faction, rank, occupation, encyclopedia facts, visible skill values, friends, enemies, and relation score.
- Player character is represented as a character dossier (`player_one`) so obligations and memories can attach to the player as an in-world actor.

Notes before editing:
- These records are central to permanent memory.
- Current local LLM scene generation reads actor/legacy character data through server-side context helpers, but the broader retrieval model is still temporary and incomplete.
- Legacy scripted flows update characters directly.

## Data Area: Events / Chronicle

Path: `campaign/events/*.json`

Purpose: Historical records of what happened in the campaign. Records are mixed-source: imported facts, legacy scripted GM turns, choice resolutions, time skips, enforcement resolutions, and current local LLM scene turns.

Classification:
- Player-facing Chronicle/history.
- Campaign truth/canon.
- Some records also contain runtime/debug metadata.
- Some records belong to legacy scripted-flow data.

### Common Event Fields

Most event records include:
- `id`
- `type`
- `title`
- `time`
- `location`
- `input`
- `involvedCharacters`
- `narration`

Apparent optional fields across event types:
- `createdAt`
- `campaignTime`
- `playerInput`
- `timeAdvancedBlocks`
- `timeBefore`
- `timeAfter`
- `scene`
- `choices`
- `continuityNotes`
- `memoryCandidates`
- `namedEntitiesUsed`
- `uncertainties`
- `validationWarnings`
- `model`
- `provider`
- `selectedChoice`
- `contextPacket`
- `outputTokenEstimate`
- `totalTurnTokenEstimate`
- `arrivals`
- `randomEvent`
- `enforcementResolved`
- `treasuryChange`
- `factsExtracted`

### Local LLM Scene Turn Shape

Written by `src/llmTurnAdapter.js` after successful validation.

```json
{
  "id": "event_20260628132142665",
  "type": "llm_scene_turn",
  "title": "Local LLM Scene Turn",
  "createdAt": "2026-06-28T13:21:42.665Z",
  "time": "Night, Summer 18, 1085",
  "campaignTime": "Night, Summer 18, 1085",
  "location": "Akkalat",
  "input": "Dawud asks Mela what she believes the scouts will find.",
  "playerInput": "Dawud asks Mela what she believes the scouts will find.",
  "timeAdvancedBlocks": 1,
  "timeBefore": "Evening, Summer 18, 1085",
  "timeAfter": "Night, Summer 18, 1085",
  "narration": "Akkalat's council chamber is dimly lit...",
  "scene": "Akkalat's council chamber is dimly lit...",
  "choices": ["Mela suggests they wait..."],
  "continuityNotes": [],
  "memoryCandidates": ["Dawud al-Sahir's tension..."],
  "namedEntitiesUsed": {
    "locations": ["Akkalat"],
    "majorCharacters": ["Dawud al-Sahir", "Mela of Oburit"],
    "minorCharacters": [],
    "clans": ["al-Sahir", "Oburit"],
    "factions": ["Khuzait"]
  },
  "uncertainties": [],
  "validationWarnings": ["continuityNotes are sparse."],
  "model": "qwen/qwen3-14b",
  "provider": "lmstudio"
}
```

Required for current LLM turn records:
- `id`
- `type`
- `createdAt`
- `time` / `campaignTime`
- `location`
- `input` / `playerInput`
- `narration` / `scene`
- `choices`
- `continuityNotes`
- `memoryCandidates`
- `namedEntitiesUsed`
- `uncertainties`
- `validationWarnings`
- `model`
- `provider`

Notes:
- `choices` are possible next actions, not happened history. They may be stored for support/debug/context, but should not be rendered as Chronicle prose.
- `validationWarnings`, `model`, `provider`, and `namedEntitiesUsed` are technical/support metadata, not player-facing Chronicle text.
- `title` is still technical in current stored LLM records.

### Legacy Scripted GM Turn Shape

Written by `src/engine.js`.

```json
{
  "id": "event_20260627150531225",
  "type": "gm_turn",
  "title": "GM Turn",
  "time": "Morning, Summer 1, 1085",
  "location": "Outside Ortysia",
  "input": "We meet Lord Manteos outside Ortysia...",
  "timeAdvancedBlocks": 1,
  "involvedCharacters": ["manteos"],
  "arrivals": ["msg_manteos_steward"],
  "randomEvent": {
    "title": "A Thread Stirs",
    "summary": "The unresolved matter...",
    "relatedThread": "thread_recruitment_rights"
  },
  "narration": "Lord Manteos receives the news..."
}
```

### Legacy Choice Resolution Shape

Written by `src/engine.js`.

```json
{
  "id": "event_20260627153211037",
  "type": "choice_resolution",
  "title": "Audience Ended Before Rivalry Could Harden",
  "time": "Evening, Summer 1, 1085",
  "location": "Castle Hall of Ortysia",
  "input": "5",
  "selectedChoice": 5,
  "timeAdvancedBlocks": 1,
  "involvedCharacters": ["manteos", "mesui"],
  "contextPacket": {
    "tokenEstimate": 3056,
    "characterCount": 12223,
    "wordCount": 1457
  },
  "outputTokenEstimate": 257,
  "totalTurnTokenEstimate": 3313,
  "narration": "The audience ends before either pride..."
}
```

### Encyclopedia Import Shape

Created during screenshot/dossier import tests.

```json
{
  "id": "event_mesui_encyclopedia_import",
  "type": "encyclopedia_import",
  "title": "Mesui Added From Bannerlord Encyclopedia",
  "time": "Midday, Summer 1, 1085",
  "location": "Outside Ortysia",
  "input": "Imported from Bannerlord encyclopedia screenshot.",
  "involvedCharacters": ["mesui"],
  "narration": "The encyclopedia does not introduce Mesui...",
  "factsExtracted": ["Mesui is leader of the Khergit."]
}
```

Read/write owner files:
- `src/store.js`: reads all events and appends event records through `appendRecord("events", event)`.
- `src/llmTurnAdapter.js`: writes current local LLM scene-turn records and advances time/state.
- `src/engine.js`: writes legacy `gm_turn`, `choice_resolution`, `time_skip`, `forced_random_event`, and `enforcement_resolution` events.
- `src/server.js`: sorts and summarizes recent events for `/api/chronicle`.
- `public/app.js`: renders player-facing Chronicle entries from the API summary.
- `src/context.js`: reads recent events for legacy context packets.
- `src/html.js`: renders recent events in the legacy static dashboard.

Bannerlord-specific assumptions:
- Event time is a formatted Bannerlord campaign timestamp.
- Locations, clans, factions, and named characters are Bannerlord/campaign entities.
- Enforcement and treasury changes assume Bannerlord actions such as denar deductions.

Notes before editing:
- Event records are not normalized. Different event types have different fields.
- Current LLM turn records contain both in-world history and technical metadata.
- Existing event files are campaign canon and should not be rewritten casually.

## Data Area: Threads

Path: `campaign/threads/*.json`

Purpose: Tracks open political/story matters and their stakes.

Classification:
- Open thread/story state.
- Campaign truth/canon.
- Some thread records are legacy scripted-flow data because `src/engine.js` creates them.

Representative shape:

```json
{
  "id": "thread_khergit_grievance",
  "title": "The Khergit Grievance",
  "status": "open",
  "importance": "major",
  "summary": "Mesui claims the Khergit have shed blood...",
  "involvedCharacters": ["mesui"],
  "stakes": [
    "Khuzait internal unity",
    "Khergit land claims"
  ]
}
```

Apparent required fields:
- `id`
- `title`
- `status`
- `importance`
- `summary`
- `involvedCharacters`
- `stakes`

Apparent optional fields:
- No optional fields are visible in current records, but future records may add source or timestamps.

Read/write owner files:
- `src/store.js`: reads all thread JSON and appends thread records.
- `src/engine.js`: creates several thread records during legacy scripted turns.
- `src/server.js`: filters `status === "open"` for the browser side panel.
- `src/context.js`: uses open threads in legacy context packets.
- `src/html.js`: renders open threads in the legacy static dashboard.

Bannerlord-specific assumptions:
- Threads are political/campaign matters tied to clans, nobles, locations, and faction stakes.
- Importance values such as `major` and `moderate` are currently free-form strings.

Notes before editing:
- Threads are likely reusable in a future generic Roleplaying Companion, but current content is Bannerlord-specific.
- No explicit created/updated timestamps currently appear in thread records.

## Data Area: Messages

Path: `campaign/messages/*.json`

Purpose: Message records for delayed/due communication in the older scripted turn loop.

Classification:
- Legacy scripted-flow data.
- Runtime/story support data.
- Campaign truth/canon once delivered, because messages can affect events.

Representative shape:

```json
{
  "id": "msg_manteos_steward",
  "from": "Steward of Lord Manteos",
  "to": "player clan",
  "dueAfter": {
    "season": "Summer",
    "day": 1,
    "year": 1085,
    "block": "Morning"
  },
  "status": "delivered",
  "summary": "A steward reminds the clan...",
  "relatedThread": "thread_recruitment_rights",
  "deliveredAt": "Morning, Summer 1, 1085"
}
```

Apparent required fields:
- `id`
- `from`
- `to`
- `dueAfter`
- `status`
- `summary`
- `relatedThread`

Apparent optional fields:
- `deliveredAt` appears after delivery.

Read/write owner files:
- `src/store.js`: reads all messages and writes individual messages through `saveMessage()`.
- `src/engine.js`: finds due messages, marks them delivered, and records arrivals in legacy events.
- `src/html.js`: renders pending messages in the legacy dashboard.

Bannerlord-specific assumptions:
- Message sources and recipients are campaign actors such as stewards, lords, and the player clan.
- `dueAfter` uses the same Bannerlord time object shape as `campaign/state.json`.

Notes before editing:
- Current local LLM scene generation does not appear to consume messages.
- This folder may become useful again for a future messenger/event queue, but it is currently legacy-scripted.

## Data Area: Enforcement

Path: `campaign/enforcement/*.json`

Purpose: Stores enforcement records: campaign consequences that future prose must respect when applicable.

Classification:
- Enforcement/normative campaign truth.
- Campaign truth/canon.
- Mixed legacy and clean v0 data in current implementation.

### Deprecated Legacy Bannerlord Shape

The current committed Bannerlord-shaped records are deprecated legacy data. They are preserved for compatibility with legacy `/api/turn`, rollback, `pendingEnforcement`, `src/context.js`, and `src/html.js`.

Representative shape:

```json
{
  "id": "enforce_event_20260627150531225",
  "status": "enforced",
  "requiredAction": "Resolve Lord Manteos' demanded 5,000 denar fine.",
  "reason": "Manteos claims the clan recruited...",
  "authority": "Lord Manteos, local landholding lord",
  "suggestedBannerlordAction": "If the player accepts the fine, deduct 5,000 denars...",
  "createdAt": "Morning, Summer 1, 1085",
  "resolvedAt": "Midday, Summer 1, 1085",
  "resolution": "The player paid Lord Manteos' 5,000 denar fine...",
  "bannerlordActionTaken": "Deduct 5,000 denars from the clan treasury."
}
```

Legacy fields:
- `id`
- `status`
- `requiredAction`
- `reason`
- `authority`
- `suggestedBannerlordAction`
- `createdAt`
- `resolvedAt`
- `resolution`
- `bannerlordActionTaken`

Deprecated Bannerlord/module-specific fields:
- `suggestedBannerlordAction`
- `bannerlordActionTaken`
- legacy string `authority`

Clean Enforcement v0 logic must not adapt these fields into the core schema and must not insert them into LLM context or the clean API/feed projection.

### Clean Enforcement v0 Shape

Clean Enforcement v0 records represent scoped laws, customs, obligations, and restrictions.

Required fields for all clean v0 records:
- `id`: non-empty string
- `type`: one of `law`, `custom`, `obligation`, `restriction`
- `status`: one of `active`, `resolved`, `retired`
- `scopes`: non-empty array of typed scope references
- `createdAt`: non-empty string

Each scope must have:
- `type`: non-empty string
- `id`: non-empty string

Law/custom/restriction records require `statement` and do not require `requiredAction`.

Obligation records require `requiredAction`. `reason`, `resolution`, and `resolvedAt` are optional supported fields for obligations.

Example clean law:

```json
{
  "id": "enforce_wergild_law",
  "type": "law",
  "status": "active",
  "scopes": [
    {
      "type": "organization",
      "id": "org_al_sahir"
    }
  ],
  "statement": "Wergild is paid first to kin, then to battle-brothers if no kin exist. The clan takes no share.",
  "createdAt": "Summer 10, 1084"
}
```

Example clean obligation:

```json
{
  "id": "enforce_manteos_fine",
  "type": "obligation",
  "status": "resolved",
  "scopes": [
    {
      "type": "organization",
      "id": "org_player_clan"
    }
  ],
  "requiredAction": "Resolve Lord Manteos' demanded 5,000 denar fine.",
  "reason": "Manteos claims the clan recruited from villages under his protection without permission.",
  "resolution": "The player paid the fine and acknowledged his authority over recruitment in protected villages.",
  "createdAt": "Morning, Summer 1, 1085",
  "resolvedAt": "Midday, Summer 1, 1085"
}
```

Unknown extra fields on otherwise clean v0 records are tolerated for future compatibility, unless they are deprecated Bannerlord legacy fields. Unknown extra fields are not included in LLM context and are not exposed through the clean API/feed projection.

### Scope Matching

Clean Enforcement v0 uses exact typed-scope matching only.

Current active scopes come from `campaign/state.json` field `activeScopes` when present:

```json
{
  "activeScopes": [
    {
      "type": "actor",
      "id": "actor_dawud_al_sahir"
    }
  ]
}
```

If `activeScopes` is absent, the runtime uses only:

```json
{
  "type": "global",
  "id": "campaign"
}
```

A clean record affects LLM prose only when `status` is `active` and at least one record scope exactly matches one active scope by both `type` and `id`. No realm, organization, actor, household, clan, faction, location, or authority hierarchy is inferred in v0.

### Feed Projection vs LLM Constraints

`/api/chronicle.enforcementRecords` is a clean feed projection. It contains all clean v0 records, including `active`, `resolved`, and `retired` records, regardless of active scope. It excludes deprecated legacy records, invalid clean records, unknown extra fields, and Bannerlord-specific legacy fields.

LLM binding constraints are different. `campaignPacket.context` includes only clean v0 records that are `active` and match the current active scope set exactly. Resolved, retired, deprecated legacy, invalid, and nonmatching active records do not enter LLM binding constraints.

Read/write owner files:
- `src/store.js`: reads all enforcement records and writes individual records through `saveEnforcement()`.
- `src/engine.js`: creates pending enforcement records and resolves the Manteos fine.
- `src/server.js`: filters non-enforced legacy items into `pendingEnforcement` for compatibility and exposes clean v0 records through `enforcementRecords`.
- `src/enforcementContextController.js`: validates clean v0 records, classifies deprecated legacy records, extracts active scopes, performs exact matching, builds clean feed projection, and formats LLM binding constraints.
- `src/campaignPacketBuilder.js`: inserts matched active clean v0 binding constraints into `campaignPacket.context`.
- `src/context.js`: includes unresolved enforcement in legacy context packets.
- `src/html.js`: renders non-enforced items in the legacy static dashboard.

Bannerlord-specific assumptions:
- Enforcement assumes the player may need to use Bannerlord console commands or manual in-game actions to sync canon.
- Current examples include denars, noble authority, recruitment rights, and faction/legal obligations.

Notes before editing:
- This data area is conceptually important for the companion app because companion canon can trump game state, but current implementation is legacy-scripted and narrow.
- Future manual record design should likely account for enforcement as a first-class concept.
- Do not migrate or delete deprecated legacy records until the clean Enforcement v0 path is proven and legacy usage is audited.

## Data Area: Rollback Snapshots

Path:
- `campaign/snapshots/manifest.json`
- `campaign/snapshots/snapshot_*.json`

Purpose: Supports one-turn rollback by storing a full copy of campaign state and collections before player input / LLM scene commit.

Classification:
- Runtime/rollback support.
- Not player-facing canon by itself.

Manifest shape:

```json
[
  {
    "id": "snapshot_20260628132142655",
    "createdAt": "2026-06-28T13:21:42.655Z",
    "reason": "before_llm_scene_turn",
    "input": "Dawud asks Mela what she believes the scouts will find.",
    "time": {
      "block": "Evening",
      "season": "Summer",
      "day": 18,
      "year": 1085
    },
    "location": "Akkalat"
  }
]
```

Snapshot file shape:

```json
{
  "id": "snapshot_20260628132142655",
  "createdAt": "2026-06-28T13:21:42.655Z",
  "reason": "before_llm_scene_turn",
  "input": "Dawud asks Mela what she believes...",
  "campaign": {
    "state": {},
    "characters": [],
    "threads": [],
    "messages": [],
    "events": [],
    "enforcement": []
  }
}
```

Apparent required fields:
- Manifest items: `id`, `createdAt`, `reason`, `input`, `time`, `location`.
- Snapshot files: `id`, `createdAt`, `reason`, `input`, `campaign`.
- Snapshot `campaign`: `state`, `characters`, `threads`, `messages`, `events`, `enforcement`.

Read/write owner files:
- `src/store.js`: owns `createSnapshot()`, `clearSnapshots()`, `restoreSnapshot()`, and manifest reads/writes.
- `src/llmTurnAdapter.js`: creates snapshots with reason `before_llm_scene_turn`.
- `src/engine.js`: creates snapshots with reason `before_player_input`.
- `src/server.js`: calls `restoreSnapshot()` for `POST /api/rollback` and exposes snapshot availability/count.
- `public/app.js`: displays rollback availability and triggers rollback.

Bannerlord-specific assumptions:
- Snapshot metadata includes Bannerlord time and current location.

Notes before editing:
- Current code intentionally clears existing snapshots before writing a new one, limiting rollback to one turn.
- Snapshot files contain full copies of campaign data. They are runtime support, not a separate canon source.

## Data Area: Generated Static Dashboard

Path: `campaign/html/index.html`

Purpose: Legacy generated HTML dashboard created by `src/html.js`.

Classification:
- Runtime/generated support.
- Legacy scripted-flow output.

Read/write owner files:
- `src/html.js`: writes this file and also writes a copy to the workspace `outputs` folder.
- `src/engine.js`: calls `renderDashboard()` after legacy scripted turns.

Notes before editing:
- The current live app is served from `public/index.html`.
- This generated HTML may be stale and should not be treated as the current frontend source.

## Current Local LLM Flow Data Use

Current local LLM scene generation primarily uses:
- A temporary frontend-built `campaignPacket` in `public/app.js`.
- `src/routes/llmRoutes.js` and `src/llm/*` for generation/validation.
- `src/llmTurnAdapter.js` to write accepted scene turns into `campaign/events/` and update `campaign/state.json`.
- `src/store.js` for reads/writes and snapshots.

Current local LLM flow does not yet appear to dynamically retrieve:
- Character dossiers from `campaign/characters/`.
- Open threads from `campaign/threads/`.
- Pending messages from `campaign/messages/`.
- Enforcement records from `campaign/enforcement/`.

This means the current LLM scene flow writes to campaign memory but does not fully read from campaign memory yet.

## Legacy Scripted Flow Data Use

The older `/api/turn` and `src/engine.js` path uses more of the data model:
- Reads and writes `campaign/state.json`.
- Reads and mutates `campaign/characters/`.
- Reads and updates `campaign/messages/`.
- Writes `campaign/events/`.
- Writes `campaign/threads/`.
- Writes and resolves `campaign/enforcement/`.
- Creates rollback snapshots.
- Regenerates `campaign/html/index.html`.

This flow contains many hardcoded Bannerlord scenes and should be treated as legacy/prototype behavior unless explicitly revived.

## Bannerlord-Specific Assumptions Found

- Campaign time is Bannerlord-specific: four seasons, 21 days per season, six day blocks.
- Currency is denars.
- Records refer to Bannerlord entities: clans, factions, lords, ladies, villages, castles, roads, khans, and named settlements.
- Character records include Bannerlord encyclopedia-derived fields such as culture, occupation, relation, friends, enemies, visible skills, and last seen.
- Enforcement records assume companion-app canon can require manual Bannerlord changes such as deducting money or using console commands.
- Current prompt/validation data expects allowed Bannerlord locations, major characters, clans, and factions.

## Implications for Future Data Model

This section does not design a new model. It only summarizes what current shapes imply for future work.

- `characters`, `events`, `threads`, and `enforcement` are likely reusable concepts for a generic Roleplaying Companion, but their current fields are heavily Bannerlord-flavored.
- Character memory is already separated from Chronicle events, which is useful. Future memory retrieval should probably preserve that distinction.
- Chronicle events currently mix player-facing prose, future choices, validation metadata, model/provider metadata, and token/debug data. Future design may need a clearer separation between in-world history, next-action options, and technical metadata.
- `campaign/state.json` mixes general campaign state with Bannerlord-specific time and legacy `currentScene` state. Future module support may need module-owned time/location rules.
- `messages` look like a useful future event queue concept, but current implementation is legacy-scripted and not used by the local LLM scene route.
- `enforcement` is a strong product concept: it records required game-state synchronization when companion canon should be applied in Bannerlord. It should influence future manual record design.
- Rollback snapshots currently copy the whole campaign and enforce one-turn rollback. That is simple and useful for v1, but future persistence work should account for snapshot size and save timing.
- The current local LLM flow writes events and uses a server-built campaign packet with focused context helpers, but some Bannerlord context remains hardcoded. Future campaign-packet building should keep replacing hardcoded context with deliberate retrieval from state, relevant actors, open threads, recent events, enforcement, units, and side channels.
- Existing event records are heterogeneous. Future work should inspect record `type` before assuming fields exist.

## Known Issues Not Addressed

- Generated prose or Chronicle entries may echo the player's submitted action too directly.
