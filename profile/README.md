# Apature

Automated design review for rendered UI. The stack captures a real page in a real headless
Chromium, measures what can be measured deterministically (WCAG contrast, overflow, touch-target
sizes), critiques the rest against the repository's own design system with a vision-language
model, and deletes every finding the model cannot point at. It judges and verifies; it never
edits code — acting on findings is the job of your coding agent or your team.

Everything below is MIT-licensed TypeScript on Node 24 + pnpm, runs offline from a clean clone
with no credentials, and says so honestly when nothing judged a page: a result without a model
behind it is labeled unjudged, never shown as a green pass.

## The components

| Repo | What it is |
|---|---|
| [verdict](https://github.com/apatureai/verdict) | The critique engine. Deterministic capture (pinned clock, frozen animations), measured WCAG contrast / overflow / touch-target facts, a VLM critique behind a grounding gate that drops any finding citing a route or element that was never captured, calibration-gated confidence, and an async job API. Works against any OpenAI-compatible endpoint that accepts images. |
| [gate](https://github.com/apatureai/gate) | The GitHub surface: an Action and an App. Runs a PR's preview build inside a hardened sandbox supervisor, hands the verified preview URL to a critique service over an HTTP contract, and publishes the review as one sticky comment plus a Check Run. Fails closed and never reds a PR on its own errors. |
| [bastion](https://github.com/apatureai/bastion) | A remote MCP server for in-loop review. A coding agent submits a preview URL, gets structured findings (route, viewport, element ref, suggested fix), applies fixes itself, then asks for a recheck. Also a worked reference for OAuth 2.1 resource-server auth, SSRF-safe URL handling, and long-running jobs over MCP. |
| [lattice](https://github.com/apatureai/lattice) | A scene graph for browser agents. Fuses DOM/layout, accessibility tree, computed style, and text-run evidence into one immutable content-addressed graph — retaining source disagreements instead of picking a winner — then renders small budgeted text views for a model prompt. |
| [canon](https://github.com/apatureai/canon) | A strict DTCG 2025.10 design-token resolver and a scanner that reads a project's declared design system out of its own files (CSS custom properties, Tailwind v3/v4, DTCG and Style Dictionary files). Abstains and explains instead of guessing; emits deterministic, content-addressed snapshots. Builds the `ui-dna` CLI from a source checkout; npm publication is planned but nothing is on npm yet. |
| [sigil](https://github.com/apatureai/sigil) | Error bars for LLM-as-judge evals: calibration against human labels, finite-sample conformal risk certificates for abstention thresholds, anytime-valid drift monitoring, and paired significance tests for model-switch claims. Dependency-free, byte-reproducible reports. |

## How they compose

- **gate → verdict.** Gate does not screenshot or run the model; both sit behind an HTTP
  contract (`GateReviewRequest` / `GateReviewResult` in gate's `packages/types`), and verdict is
  the public reference implementation of that contract. Gate reads the result's `provenance`
  stamp and withholds the grade unless a model actually judged the page.
- **bastion → verdict.** Bastion serves fixture judgments offline; point `VERDICT_CLI` at a
  built verdict checkout and the same tools return real captures and real judgments, in the same
  result contract.
- **canon → verdict.** Canon's snapshot is the machine-readable description of a repo's design
  system; verdict's design-system grounding consumes a `ui-dna.json` snapshot when one is
  present, and falls back to what the repo states directly (tokens files, brand block, detected
  component libraries) when it is not.
- **lattice and sigil stand alone.** Lattice is the capture/representation layer for any
  browser or computer-use agent, whether or not the rest of the stack is involved. Sigil
  measures any LLM judge — including the one in verdict — from recorded verdicts and human
  labels, with no model call of its own.

## Adopting it

### a) Design review in CI (gate + verdict)

Run verdict as the critique service, point gate's Action at it. Try the pair locally first —
gate's `pnpm demo:live` exercises the identical client, signing, and parsing in about a minute:

```bash
# terminal 1: the engine
git clone https://github.com/apatureai/verdict.git && cd verdict
pnpm install --frozen-lockfile && pnpm browser:install && pnpm build
export ENGINE_HMAC_SECRET="$(openssl rand -hex 32)"
node packages/serve/dist/main.js --port 8791 --model mock   # add MODEL_BASE_URL/MODEL_API_KEY for real judgments
# up? curl http://127.0.0.1:8791/readyz — /livez and /readyz are the only unsigned routes;
# every /jobs request must be HMAC-signed with ENGINE_HMAC_SECRET, so a bare curl there gets 401

# terminal 2: gate against it
git clone https://github.com/apatureai/gate.git && cd gate
pnpm install --frozen-lockfile
export GATE_ENGINE_ENDPOINT=http://127.0.0.1:8791
export GATE_ENGINE_HMAC_SECRET=<the same value>
pnpm demo:live
```

In a workflow, the same two variables are all the Action needs, with `checks: write` and
`pull-requests: write` (never `contents: write`):

```yaml
- uses: apatureai/gate@v1
  with:
    preview-url: ${{ steps.deploy.outputs.preview-url }}
  env:
    GATE_ENGINE_ENDPOINT: ${{ secrets.GATE_ENGINE_ENDPOINT }}
    GATE_ENGINE_HMAC_SECRET: ${{ secrets.GATE_ENGINE_HMAC_SECRET }}
```

The full workflow, the threat model for capturing hostile PR code, and the `.gate.yml` reference
are in [gate's README](https://github.com/apatureai/gate#using-the-action-in-a-workflow).

### b) In-loop review for coding agents (bastion, over MCP)

Give an agent eyes on the UI it just changed: it submits the preview URL, reads back findings
with element refs and suggested fixes, applies them, and rechecks.

```bash
git clone https://github.com/apatureai/bastion.git && cd bastion
corepack enable && pnpm install --frozen-lockfile && pnpm build
pnpm demo        # offline: fixture judgments, clearly stamped model_backed=false
```

For real judgments, set `VERDICT_CLI` to a built [verdict](https://github.com/apatureai/verdict)
checkout — see [Getting real judgments](https://github.com/apatureai/bastion#getting-real-judgments).
Every result carries `provenance` and `coverage`, so an agent can tell a judged page from a
replayed fixture from the payload alone.

### c) The libraries (lattice, canon, sigil)

Each is useful on its own, with no other Apature component involved. None of them is on npm
yet — clone the repo, `pnpm install --frozen-lockfile && pnpm build`, and run from the checkout
(npm publication is planned):

- **[lattice](https://github.com/apatureai/lattice)** — `pnpm capture https://example.com`
  turns a page into a sealed, content-addressed scene graph plus four budgeted prompt views.
  `@apatureai/lattice` is the pure graph library (JSON in, JSON out, no browser) and
  `@apatureai/lattice-capture` is the Playwright/CDP producer — workspace packages you build
  and consume from the repo.
- **[canon](https://github.com/apatureai/canon)** — `pnpm ui-dna tokens your-tokens.json` resolves a
  DTCG token file under the strict 2025.10 profile and names exactly which tokens resolve, which
  are refused, and why; pointing it at a project directory yields a deterministic design-system
  snapshot with per-field confidence and provenance.
- **[sigil](https://github.com/apatureai/sigil)** — feed it judge verdicts, human labels, and a
  captured panel run: `node dist/bin.js examples/credit-memo out/` emits a byte-reproducible
  report with calibration, certified abstention thresholds, and drift monitors.

## Ground rules, everywhere in the stack

- **No write path.** Nothing here edits code, commits, opens fix PRs, or drives the UI.
- **Unjudged is said out loud.** A result no model produced is stamped `model_backed: false` and
  surfaces as neutral / unjudged, never as a pass.
- **Grounded or dropped.** Findings must cite a route and element the capture actually
  produced; ungrounded ones are deleted and the drops are counted.
- **Deterministic artifacts.** Captures, snapshots, and reports are content-addressed and
  byte-reproducible where the input is frozen.
