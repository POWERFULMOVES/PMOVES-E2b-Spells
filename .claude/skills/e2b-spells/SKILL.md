---
name: e2b-spells
description: Use when picking or adapting an example from this E2B cookbook — running an agent CLI, code interpreter, MCP server, browser, or Docker inside a sandbox — especially when targeting a self-hosted E2B cluster rather than hosted e2b.dev.
---

# E2B Spells

The 42 examples under `examples/` are the working reference for building on E2B. Finding one
is easy — the directory is flat and descriptively named. This skill covers the two things
that are **not** easy: telling near-identical examples apart, and adapting any of them to a
self-hosted cluster.

## The repo assumes hosted e2b.dev. Everywhere. Silently.

A grep across every example for `E2B_DOMAIN`, `domain:`/`domain=`, `api_url`, `self-host`
and `E2B_ACCESS_TOKEN` returns **zero hits**. Every `.env.template` lists only
`E2B_API_KEY`; every README sends you to `e2b.dev/dashboard` for it.

This matters because **nothing here fails loudly on the wrong cluster.** With no
`E2B_DOMAIN` set, the SDK silently creates sandboxes on hosted e2b.dev and bills the key in
`.env`. You get a sandbox, commands run, output looks right. Wrong cluster.

**Write `.env` yourself and always include `E2B_DOMAIN`.** Do not start from an example's
env template: where one exists it omits the only variable that matters, and the
Claude Code examples recommended below **ship no env template at all** (`mcp-claude-code-js`
has `env.template` — no leading dot — so even `cp .env.template .env` fails there).

## Self-host adaptation checklist

Apply to any example before first run. None of this is in the repo.

| # | Step | Why |
|---|------|-----|
| 1 | Set `E2B_DOMAIN=<your-domain>` in `.env` | Bare domain, no scheme, no trailing slash — it is concatenated into `<port>-<id>.<domain>`. This is the whole mechanism; no code change, no CLI flag |
| 2 | Use a key **your cluster** issued | `make prep-cluster` / `make seed-db` — these live in your **e2b infra deployment, not this repo**. A hosted key 401s against your cluster and vice versa |
| 3 | **Build the template onto your cluster first** | The single most common self-host failure — see below |
| 4 | Check build-time egress | `.fromNodeImage()` / `.fromImage()` pull from Docker Hub; `npmInstall()` needs the npm registry. Air-gapped builders need `.fromImage("<your-registry>/…")` |
| 5 | Check runtime egress | An agent CLI in the sandbox must reach its model API (`api.anthropic.com`, etc.). Examples inject only the API key via `envs` |
| 6 | Private CA? | Set `NODE_EXTRA_CA_CERTS` / `REQUESTS_CA_BUNDLE`. Proxy examples using `secure: true` will reject a private cert |

### Step 3 is the one that bites — the docs actively tell you the opposite

This is not a generic caution. It is a **flat contradiction between the READMEs and your
reality**, and it is the single biggest trap in this repo.

`anthropic-claude-code-in-sandbox-*`'s README says *"We prepared a sandbox template with
Claude Code already installed."* True on e2b.dev. **False on your cluster.** The README
never tells you to build anything, so you run `npm run start`, and
`Sandbox.create("anthropic-claude-code")` fails on an alias that was never registered.

On a self-hosted cluster you must run the build as a separate one-time step first:

```
npm run e2b:build:prod                                   # JS
python -m anthropic_claude_code_in_sandbox.build_prod    # Python
```

Use the `build.dev` variant first if you want the `-dev` alias and don't want to clobber.
A fresh cluster has only `base` until you do this.

Prefer examples that ship their own template definition and build script, because those are
the ones that can bootstrap themselves onto your cluster. `anthropic-claude-code-in-sandbox-{js,python}`
is the model: a `template.*` defining the image plus `build.prod.*` calling `Template.build()`,
which registers the alias on whatever cluster `E2B_DOMAIN` points at.

An example that only *consumes* a template alias gives you no way to create it.

## Name traps — check before you start

| You want | Use | NOT | Why the confusion |
|---|---|---|---|
| The Claude Code **CLI** in a sandbox | `anthropic-claude-code-in-sandbox-{js,python}` | `claude-code-interpreter-{js,python}` | The decoy is the Claude *model* driving E2B's code interpreter. It sorts first alphabetically and reads more like "Claude Code" |
| Claude Code **+ MCP** | start from `anthropic-claude-code-in-sandbox-*`, lift the MCP wiring from `mcp-claude-code-js` | `mcp-claude-code-js` as the base | It depends on E2B's **managed MCP gateway** and assumes `claude` is already on the default template's PATH. Both are hosted assumptions |

**The table above is the primary source — prefer it to the root `README.md`.** That table
is stale: it labels the interpreter example "Claude 3 Opus" while the code pins
`claude-3-5-sonnet-20241022`. It does still distinguish "Claude Code, running inside a
Sandbox" from the interpreter row, so it corroborates, but do not lean on it alone.

## Hosted-only features — will not work self-hosted unless your cluster runs them

Reading a directory name will not reveal these; only the source does.

- **Managed MCP gateway** — `Sandbox.create({ mcp: {…} })`, `sandbox.getMcpUrl()`,
  `sandbox.getMcpToken()`. Used by `mcp-claude-code-js`.
- **Pre-built public template aliases** — see Step 3.

## Custom domains

`custom-sandbox-domain-proxy` maps `my-app.yourdomain.com` to a sandbox via an Express
reverse proxy over `http-proxy-middleware`.

**On a self-hosted cluster you usually do not need it.** Setting `E2B_DOMAIN` already serves
sandboxes natively at `<port>-<id>.<your-domain>`. Reach for the proxy only when you want
*semantic* subdomains (`user-123.yourapp.com`) rather than sandbox-ID-derived ones.

If you do use it: the subdomain→sandbox map is a single in-memory object for one sandbox
(production needs a real store), and `secure: true` rejects a private-CA cert.

## Auth: two schemes

The SDK sends `X-API-KEY` for `apiKey` and `Authorization: Bearer` for `accessToken`
(`E2B_ACCESS_TOKEN`). A `requireAccessToken` guard exists in the **SDK repo, not this one**
(`packages/js-sdk/src/api/index.ts` — don't grep for it here), though no JS SDK call site
currently sets it true. Verify which your cluster's template-build path wants before your
first build.

Its error text hardcodes `https://e2b.dev/dashboard` — **ignore that URL on a self-hosted
team**; the token comes from your own deployment.

## Known defects in examples

- **`custom-sandbox-domain-proxy` binds `process.env.PORT`, defaulting to 80** — the README
  says 3000. The constant is *named* `PROXY_PORT`, but the env var it reads is `PORT`:
  `const PROXY_PORT = process.env.PORT ? parseInt(process.env.PORT) : 80`. Setting
  `PROXY_PORT=3000` in `.env` does nothing and it silently stays on 80, which also fails as
  non-root on Linux. **Set `PORT`.**
- **`anthropic-claude-code-in-sandbox-*` README step 2 is stale.** It says *"Replace
  `<your api key>` in the code with your actual Anthropic API key"*, but `src/index.ts`
  (`main.py`) reads `process.env.ANTHROPIC_API_KEY` / `os.getenv(...)`. The placeholder only
  exists in the README's own snippet. Put the key in `.env`; there is nothing to replace.
- Examples mix the `e2b` and `@e2b/code-interpreter` packages. Combining two examples means
  picking one.

## Verifying you actually hit your cluster

Call `sbx.getHost(port)` (JS) / `sbx.get_host(port)` (Python) after creating the sandbox.
**The recommended Claude Code example never calls it — add the line yourself.**

`<port>-<id>.e2b.app` coming back when you expected your own domain means `E2B_DOMAIN` never
reached the SDK and the work ran on hosted e2b.dev, billed to your key. Treat that as a
misconfiguration signal, not a working URL.

For CLI-driven sandbox operation rather than SDK example code, use the `agent-sandboxes`
skill, which covers the same three cluster modes.
