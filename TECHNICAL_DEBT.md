# Technical Debt Register

This file tracks known technical debt discovered during development passes.

Rules:
- This is a register, not an active task list.
- Do not fix debt unless the current task explicitly includes that fix.
- Do not turn debt notes into broad refactors.
- Add entries only when debt is directly relevant to the current task.
- Update entries when debt is resolved or made obsolete.

## TD-0001 — server.js owns too many responsibilities

Status: open  
Severity: medium-high  
Discovered in: CP-0006 / FR-0006  
Area: backend routing / orchestration

Description:
`server.js` currently handles static serving, API routing, JSON parsing, playerAction validation, campaign summary projection, LLM scene orchestration, rollback, and legacy `/api/turn` routing.

Why deferred:
CP-0006 was scoped to move temporary campaignPacket ownership server-side. Refactoring `server.js` during the same task would mix architectural relocation with cleanup.

Future direction:
Split routes/controllers/helpers after server-side campaign context retrieval is stable.

Do not fix during:
Pass 2 campaignPacket ownership work.

## TD-0002 - campaignPacketBuilder still uses temporary Bannerlord context

Status: open  
Severity: medium  
Discovered in: CP-0007  
Area: server-side campaign context retrieval

Description:
`src/campaignPacketBuilder.js` now owns server-side packet construction and includes authoritative campaign state time/location plus a compact recent-events slice, but the rest of the packet still uses temporary Bannerlord-focused hardcoded context, entities, clans, factions, and allowed locations.

Why deferred:
CP-0007 was scoped only to state/time/location context. CP-0008 added only recency-based recent event context. Semantic retrieval, characters, threads, messages, enforcement, full memory retrieval, and final packet shape are later Pass 3 work.

Future direction:
Replace the temporary hardcoded context one focused retrieval helper at a time, keeping packet construction server-side and preserving the LLM route contract.

Do not fix during:
State/time/location context work.

## TD-0003 - State Boundary and temporal guard are minimal prototype safeguards

Status: open  
Severity: medium  
Discovered in: CP-0007b  
Area: state boundary / temporal input guard

Description:
`src/stateBoundary.js` allows `stateStatus: "legacy"` for compatibility with the current prototype data. `src/temporalInputGuard.js` blocks obvious date assertions and time jumps over seven days, and normalizes obvious duration units only. It is a conservative guardrail, not a full natural-language parser, scheduler, actor controller, organization policy system, or time-skip simulator.

Why deferred:
CP-0007b was scoped to reject unsafe temporal input before packet construction. Strict state IDs, organization ownership, actor scheduling, catch-up events, and long time-skip simulation need explicit future data/model work.

Future direction:
Replace legacy compatibility with strict state and organization ownership only after CRUD/schema-enforced data exists. Treat `maxTimePassedDays` as a guardrail until a proper time-skip planner and actor/event handling system exists.

Do not fix during:
Temporal input guard work.

## TD-0004 - Dual-read actor and legacy character storage

Status: open  
Severity: medium  
Discovered in: CP-0009  
Area: actor storage / legacy compatibility

Description:
Dual-read support for `campaign/actors` and legacy `campaign/characters` is intentional until CRUD and safe migration tooling exist. CP-0009 reads both sources for Actors Context, prefers `campaign/actors` when duplicate records exist, and treats `campaign/characters` as legacy actor-like records.

Why deferred:
The app does not yet have actor CRUD, schema-enforced actor records, or safe migration tooling. Bulk-moving character files now would risk losing campaign memory and would mix context retrieval with storage migration.

Future direction:
Future actor writes should go only to `campaign/actors`. Do not remove legacy character reads during TB-0009, and do not bulk-migrate character files during TB-0009.

Do not fix during:
Actors Context work.

## TD-0005 - Actor records need a core/module boundary

Status: open  
Severity: medium  
Discovered in: CP-0010a  
Area: actors / module boundaries

Description:
Actors Context currently uses actor terminology, but still reads Bannerlord-specific actor facts directly from legacy character/actor records. Core actor fields should remain module-agnostic, such as `id`, `displayName`/`name`, `aliases`, and `actorState`.

Why deferred:
CP-0010a is documentation-only. Actor and legacy character records should not be migrated until a deliberate module boundary and safe compatibility adapter exist.

Future direction:
Module-specific actor facts should move under `moduleFacts.<moduleId>.*`. For Bannerlord, fields such as culture, clan, faction, noble status, titles, and biography should eventually live under `moduleFacts.bannerlord.*`. Actors Context should eventually read module-specific facts through a module boundary or compatibility adapter, not by permanently hardcoding Bannerlord fields such as `bannerlordFacts.biography`, `culture`, `clan`, or `faction` in generic actor-context logic.

Do not fix during:
Actors Context documentation work.

## TD-0006 - Unit v0 is intentionally minimal

Status: open  
Severity: medium  
Discovered in: CP-0010  
Area: units / board pieces

Description:
Unit records are intentionally minimal in CP-0010: `id`, `name`, and `notable`. `notable` is currently only a boolean marker that says a unit is important enough to mark as story-notable.

Why deferred:
Unit templates, stats, composition, ownership, location, movement, thread attachment, reference linking, and CRUD are deferred until the unit model is better understood. Notable-thread attachment and cross-record reference identifiers are deferred until a deliberate reference model exists.

Future direction:
Design the Unit, Notable, and Actor boundaries deliberately before adding unit writes, movement, ownership, thread links, or promotion from unit to actor/notable.

Do not fix during:
Units Context work.

## TD-0007 - LLM prose fields need schema-echo guards

Status: open  
Severity: medium  
Discovered in: CP-0010b  
Area: LLM validation / commit boundary

Description:
LLM prose-field validation must reject schema echo, serialized JSON, and event-object metadata before commit. Time-skip and waiting actions are especially likely to produce schema-shaped text in prose fields if the model mirrors internal context.

Why deferred:
CP-0010b adds a targeted guard for the current `scene` prose field. Broader prose-quality validation, repair prompts, and historical cleanup are separate work.

Future direction:
Keep prose-field validation close to the commit boundary and expand it only with concrete failure cases.

Do not fix during:
Scene description validation hardening.

## TD-0008 - Thread Context Controller v0 is recency-blind and global

Status: open  
Severity: medium  
Discovered in: CP-0011  
Area: threads / context selection

Description:
Thread Context Controller v0 validates minimal Thread v0 records, selects open threads globally, caps the result, and formats compact LLM context. It does not yet select by active actor, current location, recent events, units, module entities, or relationship graph.

Why deferred:
CP-0011 is scoped to establish the controller seam without implementing a full thread architecture, read models, CRUD, reference indexes, progression, resolution, or migration.

Future direction:
Add actor-scoped and reference-scoped thread indexes, a richer canonical Thread schema, safe CRUD/schema migration, thread progression/resolution policy, and selection based on active actor, location, current scene, and recent events.

Do not fix during:
Thread Context Controller v0 work.

## TD-0009 - Messages Side Channel v0 is binary-only

Status: open  
Severity: medium  
Discovered in: CP-0012  
Area: messages / mailbox side channel

Description:
Messages Side Channel v0 exposes only binary mailbox availability through `sideChannels.messages.hasAvailable`. Message contents, counts, sender/recipient data, related threads, due dates, and previews are deliberately excluded from Prose and LLM context.

Why deferred:
CP-0012 establishes the side-channel seam without implementing an inbox, read/unread state, delivery mutation, message promotion into LLM context, message-to-thread behavior, sender/recipient identity, rumor/quest conversion, notification history, CRUD, or a richer message schema.

Future direction:
Design the Mailbox surface and message controllers deliberately before adding message counts, read/open behavior, message promotion into narration context, or message-triggered campaign changes.

Do not fix during:
Messages Side Channel v0 work.

## TD-0010 - Deprecated Bannerlord-shaped enforcement records

Status: open  
Severity: medium  
Discovered in: CP-0013  
Area: enforcement / module boundaries

Description:
Existing `campaign/enforcement/*.json` records are Bannerlord-shaped manual synchronization records. They include deprecated/module-specific fields such as `suggestedBannerlordAction`, `bannerlordActionTaken`, and legacy string `authority`.

Why deferred:
CP-0013 preserves legacy data and adds a clean Enforcement v0 path beside it. Deletion or migration is deferred until clean Enforcement v0 is proven and legacy usage is audited.

Future direction:
Keep clean Enforcement v0 records scoped and module-agnostic. Move any future Bannerlord-specific synchronization details behind a deliberate module boundary or compatibility layer instead of making them core Enforcement fields.

Do not fix during:
Scoped Enforcement v0 context work.
