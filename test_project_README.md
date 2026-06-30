# Bannerlord Companion Prototype

This is a small test environment for the companion app loop.

It proves the first version of the idea:

```text
player input
-> load campaign time and location
-> retrieve local memory
-> estimate time passing
-> advance Bannerlord-scaled time
-> check due messages
-> roll semi-random events
-> produce a narrated GM response
-> write event/memory/enforcement records
-> regenerate an HTML dashboard
```

## Run A Test Turn

Use the bundled Node runtime from Codex:

```powershell
& "C:\Users\Gebruiker\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe" "work\bannerlord_companion_proto\src\run_turn.js" "We meet Lord Manteos outside Ortysia. He objects to us recruiting from villages under his protection."
```

The prototype writes:

- campaign state in `campaign/state.json`
- new events in `campaign/events/`
- updated character dossiers in `campaign/characters/`
- generated dashboard in `campaign/html/index.html`
- copied dashboard in `outputs/bannerlord_companion_dashboard.html`

## Run The Live Browser Client

```powershell
& "C:\Users\Gebruiker\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe" "work\bannerlord_companion_proto\src\server.js"
```

Then open:

```text
http://127.0.0.1:5177/
```

The browser client polls the local server every 3 seconds and updates the chronicle without a manual refresh.

## Runtime LLM Gateway

The app runtime can call an OpenAI-compatible text model through the Node backend. The browser never calls LM Studio directly.

Copy `.env.example` to `.env` and adjust as needed:

```text
LLM_PROVIDER=lmstudio
LLM_BASE_URL=http://localhost:1234/v1
LLM_API_KEY=local
LLM_MODEL=qwen/qwen3-14b
LLM_TEMPERATURE=0.45
LLM_MAX_OUTPUT_TOKENS=1400
LLM_CONTEXT_LENGTH=4096
LLM_SUPPORTS_IMAGES=false
```

The v1 gateway is text-only and uses:

```text
POST /api/llm/generate-scene
```

The endpoint validates the campaign packet, builds prompts, calls the configured provider through `src/llm/llmClient.js`, requests JSON Schema structured output, validates canon/entity rules, and returns either accepted scene data or clear validation errors.

Run the LM Studio scene test:

```powershell
& "C:\Users\Gebruiker\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe" "work\bannerlord_companion_proto\test\test-lmstudio-scene.js"
```

If LM Studio is not running, or the configured model is not loaded, the test and endpoint return a clear provider error.
