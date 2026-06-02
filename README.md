# OMNIBRAIN // COGNITION TERMINAL

A LAN-syncable chat terminal for any **OpenAI-compatible** endpoint (e.g.
text-generation-webui / oobabooga, llama.cpp server, vLLM, LM Studio…), wrapped
in an augmented-cyberpunk HUD. Every monitor or PC that opens the terminal
mirrors the **same** conversation and settings in real time — one shared
cognition core, many screens.

```
omnibrain/
├── python.py            ← launcher  (python python.py)
├── requirements.txt
├── app/
│   ├── config.py        factory defaults (endpoint, samplers, context…)
│   ├── llm.py           engine: tokens, context cropping, think handling, streaming
│   ├── state.py         thread-safe persistent store (data/state.json)
│   └── server.py        Flask + Socket.IO sync, access gating, streaming relay
├── static/
│   ├── css/style.css    the theme
│   └── js/app.js        client: sync, streaming render, animated core
├── templates/index.html
├── data/                created at runtime (state.json) — your saved state
└── tests/               mock endpoint + integration test
```

## Quick start

```bash
pip install -r requirements.txt
python python.py
```

Then open **http://localhost:5005**. Override the port with
`OMNIBRAIN_PORT=8080 python python.py`.

By default the terminal talks to `http://10.0.0.113:5000/v1` with model
`Omnibrain-UE` and a blank API key. Change any of this under **SETTINGS → LINK**
and press **TEST CONNECTION** — the core lights up when the link is live.

## Sharing across screens (LAN sync)

The server always binds to the LAN; the **NETWORK → LAN VISIBILITY** switch
decides whether remote machines are accepted (toggle it off for local-only).
On any other device on the same network open:

```
http://<this-machine-ip>:5005
```

Every connected screen shows the identical chat, active session, system prompt,
samplers, and settings. A change on one screen appears on all of them instantly.
Set an optional **ACCESS TOKEN** to require a shared secret to connect.

## How the engine behaves

- **Context window** (default `8196`): when fullness crosses the **crop
  threshold** (default `95%`) the *oldest whole messages* fall out of context
  one at a time, in the background. The system prompt is always pinned and the
  most recent message is always kept — conversations continue indefinitely with
  a clean cut at message boundaries (never mid-message). The terminal shows the
  live fullness meter and how many messages are currently past the horizon.
- **System prompt**: sent as the `system` role and applied by the chat
  template; never dropped by cropping.
- **Samplers**: every parameter is passed to the endpoint **verbatim**
  (`temperature`, `dynatemp_low/high/exponent`, `top_p`, `top_k`, `min_p`,
  `xtc_probability`, `dry_multiplier`, `frequency_penalty`,
  `repetition_penalty_range`, `dynamic_temperature`, `temperature_last`, …).
  Use **LOAD DEFAULTS** for the recommended profile, the **EXTRA PARAMETERS**
  box for anything else (also passed verbatim), or flip *use endpoint/model
  sampler defaults* to omit them entirely and let the model decide.
- **Thinking models**: text inside `<think>…</think>` is shown as a collapsible
  **COGNITION TRACE**. Turn **SHOW THINKING** off to hide it (a `◇` marker still
  flags that reasoning happened). Reasoning is stripped from history after every
  turn, so past thinking is never replayed into context. **ENABLE THINKING** off
  appends a configurable directive (default `/no_think`).
- **Generation info** (tokens, tok/s, elapsed, finish reason) sits under each
  reply — hidden by default, toggle under **SETTINGS → DISPLAY**.
- **Prompt format**: default mode posts the `messages` array to
  `/v1/chat/completions` (the server applies its own template). Flip *use custom
  Jinja template* to render a template locally and post raw to
  `/v1/completions`.

## Sessions

Under **SESSIONS** you can create, switch, rename, duplicate, and delete chats.
Saving is automatic and persistent. Selecting a session switches every screen to
it. Per-message **COPY** / **DELETE** live on hover in the transcript.

## Testing

```bash
# terminal 1 — a fake streaming endpoint
python tests/mock_endpoint.py
# terminal 2 — the app pointed at it
OMNIBRAIN_PORT=5005 python python.py
# terminal 3 — the checks
OMNIBRAIN_PORT=5005 python tests/integration_test.py
```

The suite covers state sync, streaming, think-stripping, verbatim sampler
pass-through, context cropping, session ops, and on-disk persistence.

## Notes

- All state lives in `data/state.json` (atomic writes). Delete it to reset to
  factory defaults.
- The API key is never broadcast to clients or written until you set one.
- Token counts use `tiktoken` if installed, otherwise a configurable
  characters-per-token estimate.
- The bundled dev server is fine for a LAN. For a hardened deployment, run
  behind a production WSGI/ASGI server.
