# Glass Agent

A translucent, search-bar-shaped panel floating at the top centre of your
screen. It sits above every other window and drives whatever application is
underneath by moving the real mouse and typing on the real keyboard.

Tell it what to do in plain language. It reads the window below, decides which
control to hit, and clicks it. No app-specific integration, no browser
extension, no API per target app — it works on Chrome, Power BI, Excel,
Explorer, or anything else with a window.

Runs on open-weight models. Nothing is hard-wired to a specific vendor.

---

## How it works

The hard part of desktop automation is not clicking — it is knowing *where* to
click. Glass Agent uses two perception paths and prefers the cheap one:

```
                    ┌─ Windows UI Automation ──────────────┐
   the screen ──────┤   real control tree: names, roles,   ├──┐
                    │   exact rectangles. Free, instant.   │  │
                    └──────────────────────────────────────┘  │
                                                              ├─► numbered
                    ┌─ vision fallback ────────────────────┐  │   element list
                    │   screenshot → VLM → coordinates.    ├──┘        │
                    │   Only when the tree has no match.   │           │
                    └──────────────────────────────────────┘           ▼
                                                              text model picks
                                                              {"action":"click",
                                                               "target": 7}
                                                                       │
                                                                       ▼
                                                            Win32 SendInput
```

The accessibility tree handles the overwhelming majority of steps at zero token
cost and pixel-exact accuracy. The vision path exists for canvas-rendered UI —
Power BI report visuals, chart interiors, custom-drawn controls — where the tree
genuinely has nothing to offer. Because the screenshot never reaches the model
on the common path, a step costs roughly 2K tokens instead of 20K.

### How deep to look

The accessibility tree is deep, and where you stop reading it decides what the
agent can see at all. A browser spends its first nine levels or so on its own
frame — tab strip, toolbar, address bar — and only then reaches the document.
The DOM below that nests hard: a link on a real page commonly sits 20 to 30
levels down.

Measured on a Google results page in a 1920x1200 window:

| `uia_max_depth` | links seen | text nodes | page elements the model can read | walk |
|---|---|---|---|---|
| 14 | 0 | 1 | **1** | 0.41 s |
| 22 | 4 | 7 | 34 | 0.53 s |
| 30 | 7 | 37 | **46** | 0.88 s |

At 14 the agent saw `Back`, `Reload` and the address bar, and nothing else —
and then answered questions about a page it had never actually read. That is
not a model failure, and no amount of prompting fixes it. The default is now
30, and the extra ~0.4 s is the cheapest accuracy in the system.

Depth is not free in every direction, though, which is why the walk prunes.
Subtrees whose bounding rectangle does not touch the window are skipped
wholesale rather than node by node — a scrolled-out DOM is unreachable no
matter how deep it goes — so the deeper walk still touches about a third of the
nodes a naive one would. Reading a control's *value* costs a further two COM
round-trips, so it is asked for only on roles that can carry one.

### What survives the cap

More elements are found than fit in one prompt, so the list is ranked and cut.
Ranking used to be blind to the goal, which meant the control the user actually
named was cut with exactly the same probability as a footer link. It now scores
text against the goal's content words, so on a page of hundreds of controls the
relevant ones are the ones that survive. Nothing is *filtered* by relevance —
an unrelated control is still listed whenever there is room for it.

Two smaller economies pay for that: the text inside a link is exposed by Chrome
as a second node with the same rectangle, and those echoes are dropped in
favour of the copy you can click; and element lines no longer carry pixel
coordinates, which the model never used — it targets by number — but which cost
about ten tokens on every one of a hundred-odd lines, and gave it plausible
numbers to hallucinate clicks at.

### Reaching things that are off screen

Ordinary perception reports only what is currently visible, so a Submit button
below the fold looks to the agent as though it does not exist. The accessibility
tree, however, has contained it all along — it is simply flagged off-screen.

So instead of an element number, the agent can name what it wants:

```json
{"action": "click", "description": "Submit Quarterly Report"}
```

That searches the whole window, fuzzy-matches the description against every
control's name and value, scrolls the match into view, and clicks it — one
action, no guessing. `{"action": "scroll_to", ...}` does the same without
clicking, for reading a total or loading more rows.

Scrolling tries three strategies in order, because no single one covers every
application: the provider's own `ScrollItemPattern` (correct and instant, and
the only one that works for virtualised lists where the row does not exist yet),
then driving the nearest scrollable ancestor via `ScrollPattern`, then the mouse
wheel — which is the fallback that rescues canvas-drawn dashboards implementing
no scroll pattern at all.

---

## Requirements

- Windows 10 or 11
- Python 3.10+ (tested on 3.14)
- Node.js 18+
- Ollama running locally (the default), **or** an API key for a hosted provider

---

## Setup

```powershell
# 1. Python side
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r agent\requirements.txt

# 2. Node side
npm install

# 3. The model — local by default, no key needed
winget install Ollama.Ollama
ollama pull qwen2.5vl:7b
ollama create qwen2.5vl-glass -f Modelfile
```

No credentials are required in the default configuration; nothing about your
screen leaves the machine.

That third command is not optional. It builds the same weights with a 16384
context, because Ollama serves `qwen2.5vl:7b` with a **4096-token** window and
truncates longer prompts *silently* — one step measures ~2.6k tokens on a bare
terminal and more on a real page. See [Running fully local](#a-note-on-running-fully-local).

To use a hosted provider instead, `copy .env.example .env`, put your key in it
(`OPENROUTER_API_KEY=sk-or-v1-...`, from <https://openrouter.ai/keys>), and set
`llm.provider` / `llm.model` in `config.yaml` from the table below.

## Running

```powershell
npm start
```

That launches the overlay *and* the Python backend together. The panel appears
as a slim bar at the top centre of the screen; position and shape live under
`overlay:` in `config.yaml` (`dock: left|center|right`, `anchor:
top|center|bottom`, `width`/`height` in px). Covering the top of an app costs
the agent nothing: screenshots are taken with the panel hidden, and every
synthetic click makes the panel click-through before it fires.

- **Ctrl+Shift+Space** — show/hide the panel
- **▶** — collapse to a thin edge strip so it stops covering your work
- **■** — stop the agent mid-task

### Where warnings go

Warnings do not appear in the chat log. The log is a record of what the agent
did, and a warning is an interruption — and while the agent works you are by
definition looking at the application it is driving, not at a slim overlay bar,
so a red bubble there is a warning that scrolls past unread.

So blocks, errors, backend failures, answers sent back to be re-checked, and
values the vision model could barely read all go to **Windows notifications**.
The panel keeps the status dot and one line under the composer saying what
happened.

Toasts can fail silently — Focus Assist, notifications turned off, an
unregistered AppUserModelID — and a warning that vanishes is worse than one in
the wrong place. So `main.js` reports both a synchronous failure and Windows'
asynchronous `failed` event back to the panel, which falls back to its banner.
Standing conditions (no API key, backend down) get the banner as well as the
toast, because a dismissed notification would leave nothing explaining why
nothing works. Click the banner to clear it.

To run the backend on its own (useful for reading tracebacks):

```powershell
.\.venv\Scripts\python.exe -m agent.server
```

---

## Choosing a provider and model

The panel's picker lists the **providers** with a usable key. Each row expands
(the `▸` arrow) into that provider's live catalogue, filtered by a search
box, so a model can be swapped without editing `config.yaml` or restarting.
`config.yaml` still holds the default each provider starts on; a model picked
from the panel overrides it and survives a restart, stored per provider in
`.provider_override.json`.

Models used to be deliberately absent from the picker, on the reasoning that a
provider's model is settled once in `config.yaml` and a catalogue of hundreds of
ids is not a menu anyone can read. That held until NaraRouter retired the id
`config.yaml` named — at which point the provider was dead with no in-app way to
pick a live one, and the only repair was hand-editing YAML. See
[When a provider goes dead](#when-a-provider-goes-dead).

The picker also probes the **model**, not just the endpoint, so a provider whose
configured model has been retired or is outside your plan shows `bad model`
rather than a green light that fails on the first step.

### Gemini: the default, switched on by the key alone

Gemini is the one provider you do not configure in `config.yaml`. Paste a key
into `.env` and it takes over, outranking every other provider; remove the keys
and the previous provider (NaraRouter, or the local model) comes back.

```dotenv
# .env — Gemini is used whenever at least one of these is filled in
GEMINI_API_KEY=AIza...
```

Restart the app after a change. Nothing else moves — not the provider line, not
the model, not the fallback block.

**Six key slots, because free-tier quota is per key and per day.** One key is
one working session; when it is spent, a run stops mid-task with HTTP 429 and
there is nothing to do but wait. So `.env` holds a pool:

```dotenv
GEMINI_API_KEY=key1
GEMINI_API_KEY_2=key2
GEMINI_API_KEY_3=key3
GEMINI_API_KEY_4=
GEMINI_API_KEY_5=
GEMINI_API_KEY_6=
# or, all on one line:  GEMINI_API_KEY=key1,key2,key3
```

The moment a key reports exhausted quota, rate limiting, or is rejected as
dead, the client retries the *same request* on the next key and the run carries
on. Empty slots are skipped, so a half-filled file costs nothing. The rotation
does not reset between steps — once key 3 is the live one, later steps start
there rather than paying key 1's 429 again — and errors that are not about the
key (a malformed request, say) do not burn through the pool, because all six
would reject those identically. Only when every key is spent does the step
fail, with `All 6 gemini API keys are spent or rejected`. `GET /health` names
the live key as `key_in_use`, and each rotation is logged:

```
gemini key 1/6 is spent (HTTP 429) - switching to key 2/6
```

**The previous provider is demoted, not discarded.** Behind an exhausted pool
sits whatever was primary before, so even total Gemini failure degrades to a
slower agent rather than a dead one.

Blank values and leftover placeholders count as *absent*, so
`GEMINI_API_KEY=` or `GEMINI_API_KEY=your-key-here` falls through to the next
provider rather than failing every request with a key-shaped string that is not
a key.

To confirm which brain is live, the backend says so at startup:

```
GEMINI_API_KEY found (6 keys in the pool) - using Gemini, nararouter:agnes-2.5-flash on standby
primary model : gemini:gemini-flash-lite-latest
```

`GET /health` reports the same thing as `gemini_auto`.

Change the Gemini model under `llm.gemini` in `config.yaml`. Note that the
perception budgets are tuned for local CPU inference (see the local note below);
on Gemini you can afford to raise `max_elements` and `read_max_width` back
toward 140 and 1900 for better recall.

### Switching provider from the panel

Click the provider name in the overlay's header and a picker opens listing
every provider that is configured — Gemini, NaraRouter, local Ollama, and
whatever else has a key. Each is probed as the menu opens and tagged with the
one thing that would stop you picking it: `no key` (shown greyed, not
selectable), `unreachable`, `3 keys` for a pool, or `local`. The model each one
runs is in the tooltip.

The switch applies immediately — mid-run, the next step already uses the new
provider. Switching demotes the previous primary to the fallback rather than
discarding it, and re-tunes the perception budgets (local and hosted models get
different ones). The choice is saved to `.provider_override.json` (gitignored)
so it survives a restart; it is ignored automatically if the provider it names
loses its key, and deleting the file returns to the key-based ranking.

A deliberately chosen local primary is never hedged: the hedge exists to
rescue a stalled *hosted* primary with the local model, and inverted it would
race the hosted fallback against every 40-second local step and let it win
each time — a local model you picked on purpose would silently never be used.

### NaraRouter: switched on by the key alone, below Gemini

NaraRouter (an OpenAI-compatible gateway at `https://router.bynara.id/v1`)
follows the same key-switch pattern: paste a key and it becomes the primary,
demoting the local model to the fallback.

```dotenv
NARAROUTER_API_KEY=sk-nry-...
```

It ranks **below** Gemini, so with both keys present Gemini runs and NaraRouter
waits as the standby — pick it by hand from the panel if you want it. Remove or
blank the key and the local model returns.

The model is `llm.nararouter.model` in `config.yaml` (default
`agnes-2.5-flash`: in-plan, multimodal, 512k context, ~6s per step). Expand
NaraRouter in the panel's picker to see and choose from the rest, or list them
by hand with:

```powershell
curl https://router.bynara.id/v1/models -H "Authorization: Bearer $env:NARAROUTER_API_KEY"
```

**The catalogue is not an allow-list.** `GET /models` returns every model
NaraRouter hosts for any valid key, but calling one outside your plan answers
`403 Your plan does not include the requested model`. On the key this was
developed against, the ids that actually answered were `agnes-2.5-flash` (~6s,
vision), `union-alpha` (~16s, vision), `stepfun-3.7-flash` (~42s, vision),
`nemotron-3-super-free` (~15s), `nemotron-3-ultra-free` (~25s),
`nemotron-3.5-lightning-free` (~24s) and `laguna-s-2.1` (~23s); everything else
returned 403. If a model you pick from the panel comes back with a plan error,
that is why — pick another.

Startup logs say `NARAROUTER_API_KEY found - using NaraRouter`, and
`GET /health` reports it as `nararouter_auto`.

### When a provider goes dead

A gateway retiring a model id looks exactly like the gateway being down, and
for a while it was reported as one. NaraRouter dropped `glm-5.3-flash-free` —
the old default here — and the result was every step failing with `HTTP 404`
while `GET /models` kept answering `200`, so the picker showed NaraRouter as
reachable right up until it was selected.

Three things now separate the two cases:

- The picker probes the configured model against the catalogue, not just the
  endpoint, and tags the provider `bad model` when the id is not in it.
- The error raised at call time names the model and says the endpoint is fine:
  *"nararouter will not run 'glm-5.3-flash-free' on this key ... pick another
  model from the overlay's picker"*.
- The picker lists the catalogue, so picking another model is the repair.

If a provider genuinely is unreachable — the key is wrong, or the host is down —
it still reports `unreachable`, with the transport error in the row's tooltip.

### Everything else

The rest route through the same OpenAI-compatible client, and are selected by
editing `config.yaml`.

| Goal | `provider` | `model` | Notes |
|---|---|---|---|
| **Default — fully local** | `ollama` | `qwen2.5vl-glass` | Free, private, vision-capable. ~40 s/step on CPU. Needs the `Modelfile` build; see the local note below. |
| Cheap and fast | `openrouter` | `z-ai/glm-5.3-flash` | $0.075/M, 1.3M context, vision. ~47 reasoning tokens per step. |
| Lower latency | `openrouter` | `mistralai/mistral-small-2603` | Not a reasoning model, so no thinking delay per step. |
| Fastest hosted | `groq` | `llama-3.3-70b-versatile` | No vision — the `look`/`click_visual` path is unavailable. |

**Reasoning models cost latency, not just tokens.** Most current open-weight
models think before answering, and that thinking is billed and waited on even
when the answer is one line of JSON. Measured on a trivial reply:
`glm-5.3-flash` spent 47 reasoning tokens, `qwen3.7-flash` spent 968, and
`qwen3.6-35b-a3b` spent 922.

Deliberation is worth very little here, because a step is not an open question
— it is picking one line out of a list that is already on screen. So the
reasoning budget is capped rather than the model swapped. Measured on a real
step (a 119-element Google results page, same prompt, same model):

| | best | average |
|---|---|---|
| full reasoning | 7.1 s | **12.7 s** |
| `reasoning_effort: low` + throughput routing | 1.8 s | **2.7 s** |

Set `llm.reasoning_effort: null` to let the model think as long as it likes.
It is `null` in the shipped config, along with `route_for_speed: false`: both
are OpenRouter-only knobs and are ignored by a local provider. Restore them
(`low` / `true`) when switching back to a hosted model.

Because of this, `llm.max_tokens` must exceed the *reasoning* budget, not the
answer length. Set it too low and the provider returns HTTP 200 with
`content: null`, which looks like a broken model — the client detects this case
and says so explicitly rather than reporting an empty reply.

The `llm.fallback` block is used automatically whenever the primary provider
errors or is unreachable, so a hosted primary with a local fallback keeps
working when your connection drops.

### Failover

> **Currently disabled.** The default configuration is fully local, where a
> hosted fallback would reintroduce the API key that going local avoids. The
> design below applies whenever you switch back to a hosted primary — set
> `llm.fallback.enabled: true` and `llm.hedge_after_s: 6.0`.

`llm.fallback` is a second hosted model from a **different upstream vendor**,
not a local one. That distinction matters: OpenRouter returns 429 per upstream
provider, so a fallback on the same vendor as the primary fails for the same
reason at the same instant. Pairing Z-AI (primary) with Mistral (fallback)
means a rate limit on one does not take out the other.

The fallback is also vision-capable, so `click_visual` keeps working — and it
is not a reasoning model, so failing over is *faster* than the primary rather
than slower.

**Hedging.** Plain failover only helps when the primary *errors*. It does
nothing for the case that actually costs you time — a primary that is merely
slow, queued behind a shared rate-limit pool or spending a long reasoning
budget. So with `llm.hedge_after_s` set, the fallback is started *alongside*
the primary once that many seconds have passed, and whichever answers first
wins; the loser is cancelled. Below the threshold the fallback is never called
and never billed, so this costs nothing until it saves something.

The threshold is a quality decision as much as a speed one, because the
fallback is a weaker model — on the test page above it misread which result was
top. Six seconds sits comfortably above the primary's 2.7 s average, so it
fires on genuine stalls and not on ordinary steps. Set it to `0` to go back to
strictly sequential failover.

### A note on running fully local

This is now the default: `ollama` + `qwen2.5vl-glass`, a 7B vision-language
model. No API key, no per-token bill, and no screenshot of your desktop leaving
the machine. The trade is entirely latency.

**Reading survives the move; pointing mostly does not.** `qwen2.5vl:7b` read a
window title and a rendered figure correctly off a real 1920×1200 capture at
every width tested, so `look` works. Pointing is a different capability, and it
is much weaker. Asked to locate four known controls, with ground truth taken
from the accessibility tree:

| target | true centre | model said | error | |
|---|---|---|---|---|
| File | (79,26) | (77,24) | 3px | hit |
| Edit | (134,26) | (192,24) | 58px | **miss — lands on the next menu** |
| View | (297,26) | (307,30) | 11px | hit |
| Go Back | (500,26) | not found | — | miss |

Two of four, 24px mean error, ~90 s per attempt. On a 54px-wide menu item a
58px error is not "close" — it is the neighbouring control. So an earlier claim
here that the canvas path "still works" was too generous: it works for reading,
not for aiming.

**What actually clicks accurately is the accessibility tree**, and by a margin
that makes the choice obvious:

| path | accuracy | cost |
|---|---|---|
| `target` / `description` | 10/10 exact | ~0.04 s |
| `click_visual` | 2/4, up to 58px off | ~90 s |

So `click_visual` now tries the tree first and only falls through to vision when
no confident match exists — which is the genuine canvas case it was written for.
On ordinary UI that turns a 90-second approximate guess into an exact,
sub-second click. When vision *is* used, the returned point is snapped onto a
small control under it if there is one; on a real chart interior there is no
such element and the raw coordinate is used unchanged.

**The cost is time, on CPU.** Ollama does not offload to integrated Radeon
graphics, so this runs at 100% CPU. Measured here, a step is roughly **40 s**
and a `look` roughly **80 s**, against 2–12 s hosted. `llm.timeout_s` is raised
to 240 to match, and `agent.max_steps: 100` is a ceiling you are unlikely to
want to reach — a long task is genuinely long.

Three settings matter, and getting them wrong looks like a stupid model rather
than a misconfiguration:

- **Context window.** Ollama serves `qwen2.5vl:7b` with a **4096-token**
  window. One step measures ~2.6k tokens against a bare terminal and exceeds
  4096 on a real browser or Power BI page; add a screenshot and it is far over.
  Ollama truncates silently, so the model then appears to invent element
  numbers. `./Modelfile` rebuilds the same weights at `num_ctx 16384`:

  ```powershell
  ollama create qwen2.5vl-glass -f Modelfile
  ```

  Confirm it took with `ollama ps` — the CONTEXT column should read 16384, not
  4096. `config.yaml` deliberately names `qwen2.5vl-glass`, not `qwen2.5vl:7b`.

- **`perception.max_elements`.** On CPU the prompt is the dominant cost of a
  step. Measured on one fixed snapshot (401 elements seen), same goal, and the
  same correct decision reached in all three cases:

  | budget | prompt | step time |
  |---|---|---|
  | 140 elements / 150 chars | ~2579 tokens | 63.7 s |
  | **90 elements / 120 chars** | ~2206 tokens | **41.4 s** |
  | 50 elements / 90 chars | ~1987 tokens | 26.0 s |

  What the cap removes is the tail of the relevance ranking, and it is not
  lost — the model reaches anything outside it by name with `description`,
  which searches the whole window.

- **`perception.read_max_width`.** The most expensive single line in the file.
  On a full 1920×1200 screen, same question, correct answer every time:

  | width | `look` time |
  |---|---|
  | 1900 px | 204.1 s — over the old 90 s timeout |
  | **1280 px** | **79.8 s** |
  | 900 px | 62.9 s |

  `read_quality` stays at 90: JPEG quality changes fidelity, not how many image
  tokens the model spends, so it is accuracy that costs nothing. Resolution is
  what you pay for. The accurate path for small text is the `read_zoom_width`
  crop, not brute-force width.

**Failover is off** in this configuration. There is only one local model, and a
hosted fallback would need the API key that going local was meant to avoid. For
the same reason `llm.hedge_after_s` is `0`: hedging assumes two independent
providers, and two concurrent requests to one Ollama queue on the same CPU, so
it would make slow steps slower. Both are one line to restore — see below.

---

## Safety

The agent moves your real mouse in your real applications, so two guards sit
between the model and your machine.

**Hard blocks** — the agent refuses to interact with windows matching
`safety.blocked_window_patterns` (UAC, Windows Security, credential and
password dialogs). This cannot be overridden from the chat panel, deliberately:
an instruction that arrives through a web page the agent is reading must not be
able to talk its way into your password manager.

**Confirmations** — actions whose target or typed text matches
`safety.destructive_patterns` (delete, send, pay, submit, …) pause the run and
ask you in the panel before proceeding. Matching is word-boundary aware, so
"send" fires on a Send button but not on "Sender Name". Shortcut chords that
destroy without touching a labelled control (Alt+F4, Ctrl+W, Shift+Delete) are
caught separately.

Set `safety.confirm_destructive: false` to run unattended. Consider carefully.

### The panel and the screen underneath

The panel is always-on-top, which creates a subtle class of bug: **UI Automation
reports controls regardless of occlusion**, so the agent perceives elements the
panel is physically sitting on top of and targets them confidently — while the
click lands on the panel. Three mechanisms handle this, all in `agent/overlay.py`:

- **Clicks pass through.** Before any mouse action the panel is temporarily
  removed from hit-testing. Note that `WS_EX_TRANSPARENT` alone does *not* do
  this — it only takes effect combined with `WS_EX_LAYERED`. The panel stays
  fully visible throughout; only its mouse interception is suspended, and the
  original window style is always restored, including on exception.
- **Screenshots exclude the panel.** Click-through is not enough for the vision
  path, because the panel is still *drawn*; a screenshot would show the chat
  panel pasted over the region the model is being asked about. So the panel is
  genuinely hidden for the duration of the capture, then restored without
  stealing focus.
- **Focus on the panel does not blind the agent.** The panel is excluded from
  perception so the agent never clicks on itself, but when you click into the
  chat box the panel *becomes* the foreground window — and excluding it with no
  fallback left the agent seeing nothing at all, at exactly the moment you gave
  it work. It now falls back to the frontmost window that is not its own.

Other boundaries worth knowing:

- The backend binds to `127.0.0.1` only, and the panel's CSP permits no network
  destination except that socket.
- The agent cannot see or act on its own overlay window; it is filtered out of
  perception so it never clicks on itself.
- A non-elevated agent cannot send input to an elevated window. Windows blocks
  this at the OS level (UIPI). Run as administrator only if you actually need
  to automate an elevated app.
- Every run is capped at `agent.max_steps` actions.

---

## Layout

```
config.yaml              all tuning lives here
.env                     API keys (gitignored)

agent/
  server.py              FastAPI + WebSocket bridge
  loop.py                observe → think → act
  llm.py                 provider-agnostic OpenAI-compatible client
  prompts.py             system prompt and step prompt construction
  safety.py              blocks and confirmations
  actions.py             Win32 SendInput: mouse, keyboard, scroll
  perception/
    uia.py               accessibility tree walker (COM thread)
    screen.py            screenshot capture for the vision path
    elements.py          the shared Element type

electron/
  main.js                overlay window, acrylic, hotkey, backend spawn
  preload.js             the renderer's entire capability surface
  renderer/              chat panel UI
```

---

## Tuning

Everything below is in `config.yaml`.

| Setting | Effect |
|---|---|
| `agent.max_steps` | Ceiling on actions per instruction. Raise for long tasks. |
| `agent.verify_answers` | Refuse an answer given before anything was read off the screen. |
| `agent.check_answer_figures` | Also refuse an answer containing a figure that appears nowhere in what was read. |
| `agent.answer_retries` | How many times an answer may be sent back to be grounded. |
| `llm.reasoning_effort` | Cap thinking per step. `null` lets the model deliberate freely. |
| `llm.hedge_after_s` | Start the fallback alongside a primary that is taking this long. `0` disables. |
| `llm.route_for_speed` | Ask OpenRouter for the fastest upstream rather than the cheapest. |
| `perception.max_elements` | Elements shown per step. Higher = better recall, more tokens. |
| `perception.reading_max_elements` | The same cap for question goals, where text has to compete with controls. |
| `perception.text_chars` / `reading_text_chars` | How much of each element's text survives into the prompt. |
| `perception.uia_max_depth` | How deep to read the tree. Below ~22 a browser shows no page content. |
| `perception.find_max_depth` | Depth for the `description` search, which must cross the whole document. |
| `perception.uia_timeout_s` | Raise if a heavy Chrome page truncates the element list. |
| `perception.screenshot_max_width` | Lower to cut vision cost; below ~900 hurts small targets. |
| `perception.read_max_width` / `read_quality` | Capture budget for `look`. Small text needs both high. |
| `perception.read_zoom_width` | Enlarge a `look` crop to at least this width before reading it. |
| `overlay.width` | Panel width in logical px (default 265). |
| `overlay.height_ratio` | Fraction of screen height, `0.2`–`1.0` (default 0.62). |
| `overlay.anchor` | `bottom`, `center` or `top` — where the panel sits vertically. |
| `overlay.opacity` | Tint over the blur. Lower is more see-through; below ~0.2 gets hard to read. |
| `overlay.fade_older` | Older messages fade as new ones arrive. `false` keeps them all solid. |
| `overlay.acrylic` | Set `false` for a flat panel if blur costs you frames. |

---

## Three modes, because there are three jobs

Which mode a goal is in is decided once, from its wording, and it changes what
perception collects, what the prompt asks for, and how the answer is checked.

| Mode | The goal | Example | The job |
|---|---|---|---|
| **action** | change something | "export the report" | find the control, click it |
| **reading** | the USER asks about the screen | "what is the total in row 4?" | read a value back, and prove it was read |
| **solving** | the SCREEN asks the agent | "complete my assignment" | work the answer out, then click the option |

The first two are opposites already — see below. Solving is a third thing
again, and it used to be run as an action goal, which is why it was bad at it.

### Solving: assignments, quizzes and exams

Detected from the goal ("assignment", "quiz", "exam", "mcq", "answer the
questions", "solve the…", and similar). Three things change, all of which were
wrong for this job before:

**It can see the whole question.** In action mode a `TextControl` scores 35
against a radio button's 100, so the question stem — the one element that
decides the answer — is what the element cap discards first, and what survives
is cut to `text_chars` (150) at capture. A quiz page reaches the model as four
crisp options above half a question. Solving uses the reading-mode ranking and
text budgets instead, so the stem arrives whole and the options stay clickable.

Measured on six realistic assignment questions — same model, same prompt,
varying only the view of the page:

| What the model was shown | Correct |
|---|---|
| full question stem and options | **6 / 6** |
| stem cut at 150 chars (what action mode did) | **2 / 6** |
| stem ranked out of the list entirely | **0 / 6** |

The middle row is not a coincidence: 2 of 6 is exactly what this agent scored
on a real assignment. The failures were not bad reasoning. In most of them the
model correctly noticed it could not read the question and answered `look` or
`scroll` — it was being asked to answer a question it had only half of.

**It is allowed to solve.** The rules that make reading mode trustworthy —
*never answer from prior knowledge*, *do not compute a figure that is not
displayed* — are precisely wrong here, and they were being applied. A
multiple-choice question is not on screen to be transcribed. The prompt now
separates the two cases explicitly and tells the model that in this one it must
work the answer out, rule the distractors out one at a time, and click the
option itself rather than `Next`.

**It gets room to think.** Every other step of this agent is a shallow choice
that one sentence of `thought` covers, so the reply format was built for that.
A question with four plausible distractors is not, and `reasoning_effort: low`
with a one-sentence thought is not where one gets solved. Solving steps add a
`working` field, filled in *before* the action, and run at `agent.solve_effort`
(high) with `agent.solve_max_tokens` (4000) — the budget has to cover reasoning
tokens as well as the reply, or the model thinks up to the limit and returns
nothing. The working is shown in the panel and kept in history, so a wrong
answer can be read back rather than guessed at.

Cost of all this: about half a second per step (3.3s → 3.8s median here), and
no extra requests — the working rides in the same reply as the action, which
matters because the free tier's request budget is the scarce resource.

## Answering questions

When the goal is a question rather than an instruction, the `done` message *is*
the deliverable, and a model that has not read the screen will still happily
produce one from prior knowledge — fluent, specific, and unrelated to what is
in front of it. So a question puts the agent into **reading mode**, which
changes perception, the prompt, and what is required of the answer.

### Perception: clicking and reading want opposite things

The element list is ranked and cut, and the default ranking exists to find
things to *click*. Text scores 65 points below every button on the page, which
is fine when the goal is "press Submit" and disastrous when the goal is "what
is the total" — the cap discards the one line that held the answer and keeps a
full complement of navigation controls. The model then answers from a list that
could not possibly contain the value, and it answers anyway.

In reading mode:

| | action goal | question goal |
|---|---|---|
| text vs. buttons | text ranks below every control | text ranks above them, and figures higher still |
| elements kept | `max_elements` (90) | `reading_max_elements` (130), half reserved for readable content |
| text per element | `text_chars` (120), values clipped at 60 | `reading_text_chars` (300) |
| static text | optional (`include_static_text`) | always on |

The 60-character clip on values was its own quiet source of wrong answers: a
long cell arrived truncated mid-sentence, and the model answered from the
half it was given without knowing there was another half.

### Rows, not a shuffled column

Elements are numbered in reading order, which used to mean sorting by `top`.
That is not reading order on any real screen: a label and the figure beside it
rarely share an exact top coordinate, and two pixels of difference is enough to
interleave the end of one row with the start of the next. A table then reaches
the model as a shuffled column of fragments, and it pairs the wrong number with
the wrong label — which looks like a reasoning failure and is not one. Elements
whose vertical extents mostly overlap are now banded into a row and ordered
left to right within it.

### The answer has to exist on the screen

Two checks stand between a `done` message and the user:

- **`agent.verify_answers`** catches an answer produced before anything at all
  was read, and sends it back to be grounded.
- **`agent.check_answer_figures`** catches the commoner failure, which the
  first one misses entirely: the agent *did* read the screen, and then reported
  a number that was never on it. Every figure in the answer is compared against
  everything actually read during the run — element text plus `look` results —
  on digits alone, so `$1,284.50` matches `1284.50` and formatting differences
  raise no false alarms. A figure that appears nowhere sends the answer back
  with the offending values named. At most `agent.answer_retries` (2) times, so
  a genuinely derived figure costs two turns rather than blocking the run.

The prompt also asks for the value character for character — same digits, same
separators, same currency symbol, same units — because a rounded or reformatted
answer is a different answer, and names which element it came from.

### Reading with the vision path

Reading is captured differently from pointing. A button is still recognisably a
button at half size, but a price rendered at 13px is not still a price at
quality 70 — so `look` captures at `read_max_width` / `read_quality` rather than
the smaller budget used to locate a control.

Beyond that, `look` can be aimed:

```json
{"action": "look", "question": "what is the total?", "description": "the Revenue card"}
```

That finds the named element, scrolls it into view, and captures a padded crop
around it, enlarged to `read_zoom_width` — because reading a whole full-width window
to answer a question about one card is the worst case for a vision model, where
the thing that matters occupies a few dozen of those pixels. The crop is small,
so enlarging it is nearly free in tokens.

The reply is kept in full rather than reduced to a sentence: the exact quoted
text, the label next to it, and the confidence. A confidence below 0.5 is passed
back as an explicit warning not to report the value as fact, and if the figure
in the answer does not appear in the model's own quoted text — a self-
contradiction, and a reliable sign of a misread — that is flagged too.

## Known limitations

- **Windows only.** UI Automation and SendInput are Win32 APIs.
- **Elevated apps need an elevated agent** (UIPI, see above).
- **Chromium page content is deep, not absent.** Chrome does enable full
  accessibility once a UIA client asks, which this agent does; the content that
  looked missing was simply below the old depth limit. See *How deep to look*.
  Some Electron apps genuinely do expose almost nothing, and those still fall
  back to vision.
- **Power BI report visuals are canvas-drawn.** Menus, ribbons, and the field
  pane come through the tree; the chart interiors need the vision path.
- **The agent acts on the foreground window.** Clicking into another app
  mid-run changes what it perceives on the next step.
#   g l a s s - a g e n t  
 