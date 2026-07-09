# luciddreamer

Landing page for **luciddreamer.ai** — a Cloudflare Worker that serves a single
static HTML page introducing *luciddreamer-agent*, a Python package for dream
journaling, sleep-session tracking, and lucid-dream trigger management.

This repo is part of the PurplePincher / SuperInstance domain family. Its
siblings — [`activelog`](https://github.com/purplepincher/activelog) and
[`activeledger`](https://github.com/purplepincher/activeledger) — share the
exact same Worker + design-system skeleton, differing only in the page they
serve and the Worker name.

> **Good news up front:** unlike its two siblings, this landing page's code
> example **is accurate**. The imports it shows match the real package, and the
> package genuinely runs. The one caveat is version pinning — see
> [Honesty / status](#honesty--status).

---

## What is actually in this repo

A Cloudflare Worker whose entire job is to serve a static asset directory. There
is no application logic, no server-side processing, and no build step beyond
what Wrangler does natively.

```
luciddreamer/
├── src/index.ts        # 13-line request handler: env.ASSETS.fetch(request) + 404/500
├── public/index.html   # the page that gets served (luciddreamer-agent explainer)
├── family/             # shared PurplePincher design-system skeleton (see below)
│   ├── README.md          # operator's manual for the design system
│   ├── tokens.css         # :root palette + type scale (inlined into the page)
│   ├── base.css           # reset + component classes (.eyebrow, .chain, .ledger, …)
│   ├── provenance-panel.css
│   └── provenance-panel.html
└── wrangler.jsonc      # name="luciddreamer", assets dir ./public, binding ASSETS
```

`src/index.ts` is byte-for-byte identical to the handler in `activelog` and
`activeledger`:

```ts
export default {
  async fetch(request: Request, env: any): Promise<Response> {
    try {
      const response = await env.ASSETS.fetch(request);
      if (!response) return new Response("Not found", { status: 404 });
      return response;
    } catch (e) {
      return new Response(`Error: ${e instanceof Error ? e.message : 'Unknown error'}`, { status: 500 });
    }
  },
};
```

Every request is handed to the Workers [static assets](https://developers.cloudflare.com/workers/static-assets/)
binding (`env.ASSETS`). If no file matches, the Worker returns `404`; on an
exception it returns `500`. That is the whole runtime behavior.

### The `family/` design system

`family/` is the shared PurplePincher design skeleton, documented in
[`family/README.md`](family/README.md). Its architecture is a deliberate
**inline-at-build-time, never-fetch-at-runtime** rule: `tokens.css` and
`base.css` are copied into the page's `<style>` block (which is why
`public/index.html` is self-contained and makes no runtime CSS requests).

Two notes a reader should be aware of:

- The page does **not** perform the per-site accent swap described in
  `family/README.md`. It leaves `--claw` at the default aubergine, so this site
  currently renders in the reference palette rather than a dedicated
  luciddreamer accent.
- The `provenance-panel.*` honesty component ships in `family/` but is **not**
  used by this page. The page has its own inline "Honesty note" instead.

---

## Run it

No `package.json`, no dependencies to install — Wrangler talks to the TypeScript
entry directly. Wrangler 4.x is what this was authored against.

```bash
# local dev server (serves public/index.html)
wrangler dev

# validate config + bundle without deploying
wrangler deploy --dry-run

# deploy to Cloudflare (requires you to be authenticated to the luciddreamer account)
wrangler deploy
```

`wrangler deploy --dry-run` was verified against the sibling repos (identical
config shape): it reads the `./public` assets directory, bundles the Worker, and
reports the `env.ASSETS` binding.

---

## The `luciddreamer-agent` API (verified from the published wheel)

`luciddreamer-agent` (PyPI, published by `superinstance`; project
`github.com/SuperInstance/luciddreamer-agent`) exposes a self-contained data
model. The landing page's import line is **accurate** — it matches the package's
`__all__` exactly:

```python
from luciddreamer_agent import (
    LucidDreamerAgent,
    DreamMood,
    DreamEntry,
    SleepQuality,
    TriggerType,
)
```

`LucidDreamerAgent` provides: `record_dream(...)`, `record_sleep(...)`,
`register_trigger(...)`, `record_trigger_attempt(...)`, `suggest_triggers()`,
`get_top_dream_signs()`, `get_statistics()`, and `export_json()` / `import_json()`.

`TriggerType` covers the standard induction techniques the page names: `MILD`,
`WBTB`, `SSILD`, plus `RealityCheck`, `WakeUp`, and `Custom`.

### This was executed, not just read

Against the `0.1.0` wheel, the following ran with **no external dependencies**
and **no server** — the package works entirely standalone:

```python
from datetime import date
from luciddreamer_agent import LucidDreamerAgent, DreamMood, SleepQuality

agent = LucidDreamerAgent()
agent.record_dream(title="Flying over water", description="…",
                   mood=DreamMood.VIVID, lucidity_level=2)
agent.record_sleep(sleep_date=date.today(), quality=SleepQuality.REFRESHING)
print(agent.get_statistics())
# {'total_sessions': 2, 'total_dreams': 1, 'total_lucid_dreams': 1,
#  'lucid_dream_rate': 1.0, 'mood_distribution': {'vivid': 1},
#  'trigger_statistics': [], 'top_dream_signs': []}
```

---

## Honesty / status

Using the family's honesty-marker convention:

- ✅ **real today** — the static landing page renders; the Worker serves it via
  `env.ASSETS`; the `family/` design-system assets are present and inlined; and
  the `luciddreamer-agent` **0.1.0** package genuinely works standalone, with an
  API that matches the page.
- ⚠️ **real but conditional — pin the version.** The **current** PyPI release is
  **0.2.0** (sdist only), and it has two problems the 0.1.0 wheel does not:
  1. It now `import`s `fleet_agent` (`BaseAgent`, `EmergenceDetector`,
     `HolonomyConsensus`), so it is no longer dependency-free.
  2. Its `__init__.py` is malformed — the same mis-indented module-level
     `__init__` pattern as the sibling packages — and raises `SyntaxError`
     (`unindent does not match any outer indentation level`, line 209) on
     import.

  So `pip install luciddreamer-agent` today installs the broken 0.2.0. To get
  the working package the landing page describes, install
  **`luciddreamer-agent==0.1.0`**.
- 🔮 **later phase / not done** — no tests, no CI, no `package.json` in this
  repo; the page does not apply the per-site accent swap or the provenance
  panel.

---

## Related

- [`family/README.md`](family/README.md) — the design-system operator's manual.
- Sibling landing repos: [`activelog`](https://github.com/purplepincher/activelog),
  [`activeledger`](https://github.com/purplepincher/activeledger).
- The package source: [`SuperInstance/luciddreamer-agent`](https://github.com/SuperInstance/luciddreamer-agent)
  (separate repo; this landing repo does not contain it).
