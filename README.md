# github-contribution-log
# Contribution [1]: [Adapter: OpenClaw]

**Contribution Number:** 1  
**Student:** Darin Andoh-Mensah  
**Issue:** [orthogonalhq/nous-core #299](https://github.com/orthogonalhq/nous-core/issues/299)  
**Status:** ✅ Complete — PR #402 merged (2026-07-05)  

---

## Why I Chose This Issue

I chose this issue because it deals directly with building out clean, fast, and highly efficient backend systems, which aligns perfectly with my goal to expand my backend expertise. Implementing an engine adapter for a personal-AI framework requires a deep understanding of data flows, data validation, and minimal-overhead processing. 

Because this task involves mapping complex, real-time lifecycle functions without introducing execution latency, it gives me a great opportunity to learn how to write robust, production-ready server-side logic that remains lightweight and responsive.

---

## Understanding the Issue

> **Scope correction (per the 2026-06-18 maintainer update on #299):** This issue was originally written against the legacy `AgentAdapter` integration path under `self/subcortex/coding-agents/`. That path has been **superseded** and must not be used. OpenClaw is now to be implemented as a **CLI provider leaf** under `self/subcortex/providers/src/providers/<vendor>/`, against the integration branch `feat/contributor-friendly-inference-provider-surface`. The sections below describe the issue in terms of that current contract.

### Problem Description

`nous-core` talks to external AI models and agents through **certified provider leaves** — small, self-contained modules that live under `self/subcortex/providers/src/providers/` and conform to a shared contract so the platform can discover, construct, and route to them uniformly. The platform ships leaves for `anthropic`, `openai`, `ollama`, and a command-line agent (`codex-cli`), **but no leaf for OpenClaw.** Without one, there is no supported way for the platform to run OpenClaw as a model provider.

OpenClaw is a command-line (CLI) agent, so it belongs to the same family as `codex-cli`: it is driven by the **`agent-cli` protocol** (the platform spawns a local process, sends a prompt, and reads the response back from the process output) rather than by an HTTP API.

### Expected Behavior

The provider catalog should include a production-grade `openclaw` leaf that:

* Declares its metadata via `ProviderDefinitionLeaf` (vendor key, protocol, capabilities, and a declared `executionCapabilityProfile`), with its built-in provider ID **derived from the `vendorKey`** rather than hand-authored.
* Speaks the `agent-cli` protocol: builds a non-interactive CLI invocation, delivers the prompt, and returns the model output — supporting both single-shot `invoke()` and streaming.
* Is auto-discovered by the provider code generator and resolvable through the shared registry/adapter resolver, exactly like the existing `codex-cli` leaf.

### Current Behavior

On the integration branch, `self/subcortex/providers/src/providers/` contains `anthropic`, `codex-cli`, `ollama`, and `openai` — and no `openclaw` directory. As a result, `openclaw` is absent from the generated provider catalog (`provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts`), so the registry has no factory to construct an OpenClaw provider and no adapter key to resolve. The capability simply does not exist yet — this is a missing-feature gap, not a runtime crash.

### Affected Components

* **New provider leaf:** `self/subcortex/providers/src/providers/openclaw/` — `definition.ts`, `adapter.ts`, `implementation.ts`, `provider.ts`, `index.ts`.
* **Shared contracts (read-only references):** the `agent-cli` protocol under `self/subcortex/providers/src/protocols/agent-cli/`, the `ProviderDefinitionLeaf` schema in `self/subcortex/providers/src/schemas/provider-definition.ts`, and the `vendorKey → providerId` derivation in `self/subcortex/providers/src/provider-identity.ts`.
* **Generated catalog (regenerated, not hand-edited):** `provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`.
* **Wiring + tests:** the codegen's per-vendor extra-exports map in `scripts/generate-provider-aggregates.mjs`, the package entry point `src/index.ts`, the new test file under `src/__tests__/providers/`, and the existing roster assertions that enumerate the certified providers.

---

## Reproduction Process

### Environment Setup

Per `CONTRIBUTING.md`, the repo requires **Node 22+** and **pnpm 10+** (npm-based installs are not supported for this workspace). Setup:

1. Clone the fork and add the upstream remote (`orthogonalhq/nous-core`).
2. Fetch and base the work on the active integration branch: `feat/contributor-friendly-inference-provider-surface` — this is where the CLI-provider machinery (`agent-cli` protocol, `cli-session-manager`, `provider-identity`, and the reference `codex-cli` leaf) lives. None of it exists on `main`.
3. `pnpm install` then `pnpm build` from the provider package to confirm a clean baseline.

### Steps to Reproduce (the missing-feature gap)

1. Check out the integration branch and list the provider leaves: `ls self/subcortex/providers/src/providers/`.
2. Observe the directory contains only `anthropic`, `codex-cli`, `ollama`, and `openai` — there is **no `openclaw` leaf**.
3. Inspect the generated catalog (`provider-definitions.ts`) and confirm `openclaw` is not among the registered vendor keys.
4. **Observed result:** the platform has no way to construct or route to an OpenClaw provider — the integration the issue asks for is simply absent.

### Reproduction Evidence

- **Working branch:** [feature/openclaw-adapter](https://github.com/DAmensah27/nous-core/tree/feature/openclaw-adapter) (based on upstream `feat/contributor-friendly-inference-provider-surface`).
- **Findings:** Confirmed via directory inspection and the generated catalog that no `openclaw` leaf or vendor key exists on the integration branch, while `codex-cli` provides a complete, working template for a CLI-based provider leaf to model the implementation on.

---

## Solution Approach

### Analysis

The gap is that no `openclaw` provider leaf exists. The platform's provider catalog is **auto-generated** by scanning `self/subcortex/providers/src/providers/` for leaves that supply the four required files (`definition.ts`, `adapter.ts`, `provider.ts`, `index.ts`). So the fix is to add a correctly-shaped `openclaw` leaf and let the generator register it — no hand-editing of the generated catalog. The existing `codex-cli` leaf is the closest analog (it is also a CLI agent on the `agent-cli` protocol) and serves as the reference pattern.

### Proposed Solution

Implement an `openclaw` CLI provider leaf that mirrors `codex-cli`'s structure but uses OpenClaw's own clean CLI contract: a one-shot, non-interactive `openclaw run --headless --no-color` invocation, the prompt delivered over **stdin**, and the final response read back from **stdout**. The leaf declares `executionCapabilityProfile: 'session_bound_command'` (a one-shot exec, not a long-lived process), supports an optional `--model` flag, honors executable overrides via environment variables, handles abort signals, streams stdout transcript chunks, and maps CLI failures to typed `NousError`s. After adding the leaf, regenerate the catalog so it is discovered, and update the codegen's extra-exports map and the roster assertions that enumerate certified providers.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** `nous-core` has no OpenClaw provider leaf, so it cannot run OpenClaw as a model provider. The current contract is a CLI provider leaf on the `agent-cli` protocol, not the deprecated `AgentAdapter`.

**Match:** Use the existing `codex-cli` leaf as the reference pattern — it is the certified example of a CLI agent on the same protocol — and follow the `ProviderDefinitionLeaf` schema and `vendorKey`-derived ID convention.

**Plan:**
1. Rebase the working branch onto the integration branch `feat/contributor-friendly-inference-provider-surface`.
2. Create the five-file leaf under `self/subcortex/providers/src/providers/openclaw/`.
3. Implement the `agent-cli` invocation (stdin prompt → stdout response), streaming, executable resolution, and error mapping.
4. Run `pnpm run generate:providers` to register the leaf in the generated catalog, and add the OpenClaw helpers to the codegen extras map + `src/index.ts`.
5. Update the existing roster assertions to include `openclaw`.
6. Write a unit suite (modeled on `codex-cli`'s) using an injected fake runner, then run `pnpm build` and the full provider test suite.

**Implement:** [Branch Link](https://github.com/DAmensah27/nous-core/tree/feature/openclaw-adapter)

**Review:** Verify compliance with `CONTRIBUTING.md` — strict typing, no `fetch`/`process.env` in the definition (metadata-only), and a clean `generate:providers --check`.

**Evaluate:** Run `pnpm build` (clean typecheck/compile), the new `openclaw.test.ts` suite, and the full provider package suite to confirm no regressions. *(Results recorded in the Testing Strategy section below.)*

---

## Testing Strategy

Testing follows the existing provider-leaf conventions in the repo. The reference leaf (`codex-cli`) is mirrored: all CLI execution is driven through an **injected fake runner** (`createFakeAgentCliRunner`) so the unit suite never shells out to a real binary. The full provider package suite was run to prove the new leaf integrates without regressing the existing roster.

### Unit Tests

New suite: `self/subcortex/providers/src/__tests__/providers/openclaw.test.ts` — **9 cases, all passing**:

- [x] **Definition metadata** — `OPENCLAW_PROVIDER_DEFINITION` declares `vendorKey: 'openclaw'`, `protocol: 'agent-cli'`, `adapterKey: 'openclaw'`, `providerClass: 'local_text'`, `isLocal: true`, `executionCapabilityProfile: 'session_bound_command'`, `capabilities.streaming: true`; the `agentCli` block validates cleanly through `AgentCliProviderMetadataSchema`; `defaultArgs` equal `['run', '--headless', '--no-color']`.
- [x] **Prompt rendering** — `renderOpenClawPrompt` flattens system prompt + gateway context frames + tool definitions into a single prompt string.
- [x] **ProviderAdapter contract** — `executionCapabilityProfile` is exposed, `capabilities.streaming` is `true`, `formatRequest` produces the expected prompt, and `parseResponse` falls back to a text-safe response for non-JSON output instead of throwing.
- [x] **`invoke()` happy path** — with an injected runner, stdout is returned as `output` (trimmed), `usage.computeMs` is derived from the runner timing, and the invocation carries the correct executable, env (`NO_COLOR: '1'`), timeout, metadata, and CLI args (`--model claw-pro` appended).
- [x] **Default-model path** — when `modelId` is the synthetic default, no `--model` flag is passed; `messages[]` input is rendered to the stdin prompt (`user: Summarize this.`).
- [x] **Error mapping** — a non-zero CLI exit is mapped to a thrown typed `NousError` carrying the stderr tail.
- [x] **Streaming** — `stream()` yields each stdout transcript chunk as a content delta, then a terminal `{ content: '', done: true }` chunk.
- [x] **Executable resolution** — `selectOpenClawExecutable` honors precedence: explicit option → `NOUS_OPENCLAW_CLI_BIN` → `OPENCLAW_CLI_BIN` → `'openclaw'`.
- [x] **Live-runner construction** — the provider constructs with the default (real) process runner without spawning anything during tests.

**Worked examples (input → expected → actual, captured from a live run):**

| Case | Input | Expected | Actual |
|---|---|---|---|
| `invoke()` output | `{ prompt: "Build the provider leaf." }`, runner stdout `"openclaw saw: …"` | `output: "openclaw saw: Build the provider leaf."`, `usage.computeMs: 80` | identical ✅ |
| Default-model args | `modelId: "openclaw/default"` | `["run","--headless","--no-color"]` | identical ✅ |
| Custom-model args | `modelId: "claw-pro"` | `["run","--headless","--no-color","--model","claw-pro"]` | identical ✅ |
| `messages[]` → stdin | `[{ role:"user", content:"Summarize this." }]` | `"user: Summarize this."` | identical ✅ |
| Non-zero exit | `exitCode: 1`, stderr `"openclaw: model not found"` | throws `NousError` containing stderr | `threw: Agent CLI exited with code 1. openclaw: model not found` ✅ |
| Streaming | CLI emits `"Hello "`, `"world"` | `[{content:"Hello ",done:false},{content:"world",done:false},{content:"",done:true}]` | identical ✅ |

### Integration Tests

The leaf is auto-discovered by the provider codegen and flows through the shared catalog/registry. Existing integration suites were extended to include `openclaw` and re-run:

- [x] `provider-codegen.test.ts` — `--list` output and the checked-in generated aggregates (`provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`) stay in sync after adding the leaf.
- [x] `provider-definitions.test.ts` / `provider-definition-types.test.ts` — `openclaw` is in the validated vendor roster, derives its `wellKnownProviderId` from `vendorKey`, and the definition source stays metadata-only (no `fetch` / `process.env` / `wellKnownProviderId`).
- [x] `adapter-resolver.test.ts` — the `openclaw` adapter module resolves through the canonical resolver.
- [x] `provider-pipeline-integration.test.ts` — `openclaw` aggregates by vendor key, and the registry constructs an `OpenClawProvider` end-to-end (definition → factory → adapter → registry).

### Manual Testing / Validation

Run from `self/subcortex/providers`:

- `pnpm run check:generated` → generated catalogs in sync (exit 0).
- `pnpm run build` (`check:generated` + `tsc --build --force`) → **clean typecheck/compile, no errors**.
- `npx vitest run src/__tests__/providers/openclaw.test.ts` → **9/9 passing**.
- `npx vitest run` (full package suite) → **314 passing, 2 skipped** (the 2 skipped are the pre-existing `codex-cli.live.test.ts` cases that require a real CLI binary). No regressions introduced by the new leaf.

---

## Implementation Notes

### Progress Summary

**Pivot to the current contract.** A maintainer update (2026-06-18) redirected #299 from the deprecated `AgentAdapter` / `coding-agents` path to the new **CLI provider leaf** contract on `feat/contributor-friendly-inference-provider-surface`. First step was rebasing the working branch onto that integration branch (the CLI-provider machinery — `agent-cli` protocol, `cli-session-manager`, `provider-identity`, and the reference `codex-cli` leaf — lives there, not on `main`).

**Studied the reference leaf.** Read `codex-cli` end-to-end (definition / adapter / implementation / provider / index), the `agent-cli` protocol (`adapter.ts`, `runner.ts`), the `ProviderDefinitionLeaf` schema, `provider-identity` (the `vendorKey → wellKnownProviderId` derivation), and the codegen that auto-generates the provider aggregates.

**Built the OpenClaw leaf.** Implemented the five-file leaf mirroring `codex-cli`'s shape but with OpenClaw's own clean CLI contract: a one-shot `openclaw run --headless --no-color` invocation, prompt delivered over **stdin**, final response read back from **stdout**, optional `--model` flag, env-var executable overrides, abort-signal handling, streaming via stdout transcript chunks, and typed `NousError` failure mapping. Wired it into the catalog by regenerating the aggregates and updated the codegen's per-vendor extras map so the package root re-exports the OpenClaw helpers.

### Challenges Faced

- **Stale issue scope.** The issue text and the original branch pointed at a superseded subsystem. Resolved by following the maintainer note, switching to the integration branch, and treating `codex-cli` as the source-of-truth pattern.
- **Catalog is generated, not hand-edited.** `provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts` carry a "do not edit by hand" header and are produced by `generate-provider-aggregates.mjs`. The leaf is auto-discovered from the directory, so the fix was to add the four required files + run `generate:providers`, then keep the codegen `--check` green.
- **Roster-coupled tests.** Several suites assert the exact vendor roster (sorted) and exact type unions (`ProviderVendorKey`). Adding a leaf intentionally breaks these as a tripwire; updated each assertion to include `openclaw` in the correct alphabetical position (`openclaw` sorts after `openai`).
- **Scoping the execution profile.** Chose `session_bound_command` (matching `codex-cli`) because OpenClaw runs as a one-shot headless exec, not a long-lived `persistent_process` — so persistent-chat surfaces correctly reject it via capability guardrails. Documented in the definition `caveats`.

### Code Changes

**Branch:** [`feature/openclaw-adapter`](https://github.com/DAmensah27/nous-core/tree/feature/openclaw-adapter) (based on upstream `feat/contributor-friendly-inference-provider-surface`).

- **New files (the leaf):** `self/subcortex/providers/src/providers/openclaw/{definition,adapter,implementation,provider,index}.ts`
- **New tests:** `self/subcortex/providers/src/__tests__/providers/openclaw.test.ts`
- **Generated (regenerated, not hand-edited):** `provider-definitions.ts`, `provider-adapters.ts`, `provider-factories.ts`
- **Hand-edited:** `scripts/generate-provider-aggregates.mjs` (added `openclaw` to the per-vendor extra-exports map); `src/index.ts` (export `OpenClawProvider`); roster assertions in `adapter-resolver.test.ts`, `provider-codegen.test.ts`, `provider-definition-types.test.ts`, `provider-definitions.test.ts`, `provider-pipeline-integration.test.ts`
- **Approach decisions:**
  - *Mirror `codex-cli`, don't invent a new pattern* — keeps the contribution review-friendly and consistent with the certified-leaf shape reviewers expect.
  - *stdin prompt + stdout response, transcript format `text`* — simplest honest contract for a headless CLI; fully exercisable through the fake runner with no real binary.
  - *Inject the runner everywhere* — the live `spawn`-based runner is only constructed lazily; all unit tests use `createFakeAgentCliRunner`, so the suite is deterministic and offline.

---

## Pull Request

**PR Link:** [orthogonalhq/nous-core #402](https://github.com/orthogonalhq/nous-core/pull/402) (against `feat/contributor-friendly-inference-provider-surface`)

**PR Description:**

> ### Add an OpenClaw CLI provider leaf
>
> **What & why.** `nous-core` routes to external models and agents through certified **provider leaves** under `self/subcortex/providers/src/providers/`. The catalog ships leaves for `anthropic`, `openai`, `ollama`, and the `codex-cli` command-line agent, but there is **no leaf for OpenClaw** — so the platform has no supported way to run OpenClaw as a provider (issue #299). OpenClaw is a CLI agent, so it belongs to the same family as `codex-cli` and uses the **`agent-cli` protocol** (spawn a local process, send a prompt, read the response) rather than an HTTP API.
>
> **What this PR adds.** A production-grade `openclaw` leaf modeled on the `codex-cli` reference:
> - Five-file leaf under `self/subcortex/providers/src/providers/openclaw/` (`definition`, `adapter`, `implementation`, `provider`, `index`).
> - Declares metadata via `ProviderDefinitionLeaf` with the provider ID **derived from `vendorKey`**, `protocol: 'agent-cli'`, `providerClass: 'local_text'`, `isLocal: true`, and `executionCapabilityProfile: 'session_bound_command'` (a one-shot headless exec, not a long-lived process).
> - Speaks the `agent-cli` contract: a non-interactive `openclaw run --headless --no-color` invocation, prompt delivered over **stdin**, response read back from **stdout**, with an optional `--model` flag, env-var executable overrides (`NOUS_OPENCLAW_CLI_BIN` → `OPENCLAW_CLI_BIN` → `openclaw`), streaming via stdout transcript chunks, and CLI failures mapped to typed `NousError`s.
> - Auto-discovered by the provider codegen; the generated aggregates (`provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts`) are regenerated, not hand-edited.
>
> **Testing.** A focused suite (`src/__tests__/providers/openclaw.test.ts`) drives all CLI execution through an injected fake runner (`createFakeAgentCliRunner`) so unit tests never shell out, plus live-process tests that exercise the real spawn path. `pnpm build` is clean and the full provider package suite passes with no regressions.
>
> **Scope note.** Built against `feat/contributor-friendly-inference-provider-surface` (the CLI-provider machinery does not exist on `main`). Per the 2026-06-18 maintainer update on #299, this intentionally targets the new provider-leaf contract rather than the superseded `AgentAdapter` / `coding-agents` path.

**Maintainer Feedback:**

- **2026-06-22 — Changes requested (early-access provider integration review).** The maintainer confirmed the overall provider-leaf shape is correct (agent-cli path, right metadata, `executionCapabilityProfile: 'session_bound_command'`, generated catalog updates, focused fake-runner tests) and requested two changes before merge:
  1. **Windows command-injection risk.** The process runner passed the user-controllable `config.modelId` into CLI args (`['--model', this.config.modelId]`) and spawned with `shell: platform === 'win32'`. On Windows that routes user-controlled values through `cmd.exe`, where shell metacharacters in a model id could be interpreted instead of treated as a literal argument. Requested fix: avoid `shell: true` and spawn with literal argv semantics; handle any Windows `.cmd` resolution without letting model-id/prompt-derived values reach the shell.
  2. **Abort handling only worked pre-start.** The implementation snapshotted the abort state (`{ aborted: request.abortSignal.aborted }`) before spawn and only checked it before launching. Once `openclaw` was running, later cancellation did not kill the child, so a canceled request could run to completion or timeout — contradicting the PR's claim that abort signals are supported. Requested fix: either wire the abort signal to terminate the child after spawn, or narrow the claim/tests to pre-start-only.
  - *(Non-blocking, maintainer-side follow-up they flagged: some persistent-chat guardrails still look Codex-specific and should become generic for agent-cli providers. They are handling this as provider-surface work; nothing required from this PR.)*

- **2026-06-28 — Addressed both requested changes.**
  1. **Shell-safe spawning.** Removed `shell: platform === 'win32'`; the process runner now always spawns with `shell: false` and passes arguments as a **literal argv array**, so the model id and prompt-derived values can never be interpreted by a shell. Added `planOpenClawSpawn`, which resolves the executable per platform: POSIX and native Windows binaries are spawned directly, while a Windows `.cmd`/`.bat` shim (which Node refuses to spawn without a shell) is routed through `cmd.exe /d /s /c` with **every argument explicitly escaped** (`windowsVerbatimArguments: true`) rather than relying on the shell to join the command line. On Windows the bare `openclaw` name is resolved via `where.exe`, preferring a native `.exe`/`.com` over a `.cmd` shim. New tests assert literal argv passthrough for a metacharacter-laden model id and cover the `.cmd` escaping and `.exe`-preference branches.
  2. **Live abort.** The provider now forwards the real `AbortSignal` to the runner (alongside the pre-start snapshot the shared contract expects). After spawn the runner registers an `abort` listener that kills the child with `SIGTERM` and resolves the run as a failure, with the listener and timeout cleaned up on settle. New live-process tests confirm a post-start abort terminates the child promptly (not at timeout) and that a pre-start abort still short-circuits without spawning. The "abort supported" claim is now accurate for both pre- and post-start cancellation.
  - *Verification:* `pnpm build` clean; full provider suite **321 passing / 2 skipped**; OpenClaw suite **16/16**.

- **2026-07-05 — Merged (initial early-access OpenClaw provider integration).** The maintainer confirmed the final version lands OpenClaw as an `agent-cli` provider leaf with the expected `session_bound_command` metadata, generated provider-catalog wiring, package exports, focused fake-runner coverage, the Windows spawn hardening, and live abort handling. They resolved the final merge conflict on their side as **maintainer-side provider-roster/catalog churn** — additive conflict resolution from other provider leaves landing on the integration branch, not a contributor-side issue with this PR. The remaining generic `agent-cli` / persistent-chat guardrail cleanup is tracked separately so this contribution stays focused.

**Status:** ✅ Merged — 2026-07-05 ([PR #402](https://github.com/orthogonalhq/nous-core/pull/402))

---

## Learnings & Reflections

### Technical Skills Gained

- **Building to a plugin contract instead of a bespoke integration.** The biggest shift was learning to implement a *certified provider leaf* — a small, self-contained module that conforms to a shared contract (`ProviderDefinitionLeaf`, the `agent-cli` protocol, a `vendorKey`-derived provider ID, a declared `executionCapabilityProfile`) so the platform can discover, construct, and route to it uniformly. Modeling the leaf on the existing `codex-cli` reference taught me how to read a certified pattern and reproduce its shape rather than inventing my own.
- **Code-generated catalogs and "do not hand-edit" boundaries.** The provider catalog (`provider-definitions.ts` / `provider-adapters.ts` / `provider-factories.ts`) is produced by a generator that scans the providers directory. I learned to add the leaf's four required files and run `generate:providers` to register it, keeping `--check` green, instead of editing generated output — and why roster-coupled tests intentionally break as a tripwire when a new vendor is added.
- **The `agent-cli` process model.** Driving a CLI agent as a provider: a non-interactive `openclaw run --headless --no-color` invocation, prompt over **stdin**, response over **stdout**, streaming via transcript chunks, env-var executable resolution, and mapping CLI failures to typed `NousError`s.
- **Secure child-process spawning.** The review taught me the concrete mechanics of shell injection on Windows: how `shell: true` joins arguments into a single `cmd.exe` command line where metacharacters get interpreted, and how to avoid it by spawning with `shell: false` and a **literal argv array**, resolving the executable myself, and routing `.cmd`/`.bat` shims through `cmd.exe` with each argument explicitly escaped.
- **Correct cancellation semantics.** The difference between a *snapshot* of an abort state (checked once before spawn) and a *live* `AbortSignal` wired to actually kill the running child — including cleaning up the listener and timeout on settle so nothing leaks.
- **Deterministic, offline testing of process code.** Using an injected fake runner (`createFakeAgentCliRunner`) for the unit suite so tests never shell out, plus a few live-process tests (spawning real `node`) to prove literal-argv passthrough and post-start abort.

### Challenges Overcome

- **A stale issue scope.** Issue #299 was originally written against the deprecated `AgentAdapter` / `coding-agents` path. A maintainer update redirected it to the new CLI provider-leaf contract on the `feat/contributor-friendly-inference-provider-surface` integration branch. I resolved this by following the maintainer note, rebasing onto the integration branch (where all the CLI-provider machinery lives — none of it exists on `main`), and treating `codex-cli` as the source-of-truth pattern.
- **Security review feedback.** My first version inherited two flaws from the reference pattern: a Windows shell-injection vector (user-controlled `modelId` passed through `shell: true`) and abort handling that only worked *before* the process started. Rather than doing the minimum, I fully wired live abort and rebuilt the spawn path to be shell-safe on every platform, then added focused tests for each so the "abort supported" claim became accurate.
- **Working in a busy, generated area.** The provider catalog and roster assertions are touched by every new provider leaf, so the final merge hit conflicts from other leaves landing on the integration branch. The maintainer handled that as additive roster/catalog churn on their side — a reminder that generated, roster-coupled surfaces are naturally conflict-prone and that keeping the leaf's own files clean and regenerating the catalog is what keeps a contribution mergeable.

### What I'd Do Differently Next Time

- **Read the reference pattern *and* its known weaknesses first.** I copied `codex-cli`'s spawn/abort shape faithfully — which also copied its latent shell-injection and pre-start-only abort issues. Next time I'll treat "the reference does it this way" as a starting point, not a guarantee, and run a security lens over child-process and user-input handling *before* review.
- **Verify environment assumptions earlier.** I lost a little time to node's `-e` argument parsing (`--model` being read as a node flag) while writing the live-process tests. Confirming how the harness passes argv up front would have saved a debug cycle.
- **Confirm the target contract before writing code.** The scope pivot from `AgentAdapter` to the provider-leaf contract could have cost a lot more if caught late; checking issue comments for maintainer updates before starting is now part of my process.

---

## Resources Used

- **The `codex-cli` provider leaf** (`self/subcortex/providers/src/providers/codex-cli/`) — the certified reference pattern this contribution was modeled on.
- **In-repo contracts** — the `agent-cli` protocol (`src/protocols/agent-cli/`), the `ProviderDefinitionLeaf` schema (`src/schemas/provider-definition.ts`), and the `vendorKey → providerId` derivation (`src/provider-identity.ts`).
- **`orthogonalhq/nous-core` #299** and the maintainer's scope-pivot comment redirecting the work to the provider-leaf contract on the integration branch.
- **`CONTRIBUTING.md`** — Node 22+ / pnpm 10+ toolchain, strict-typing and metadata-only definition rules, and the `generate:providers --check` requirement.
- **Node.js `child_process` docs** — `spawn` options (`shell`, `windowsVerbatimArguments`, `windowsHide`) and the Windows `.cmd`/`.bat` spawning restrictions that informed the shell-injection fix.
- **PR #402 review thread** — the maintainer's two requested changes (Windows spawn hardening, live abort) and the merge confirmation.

---
---

# Contribution [2]: [AI Provider type safety — replace `any` with `LanguageModel`]

**Contribution Number:** 2  
**Student:** Darin Andoh-Mensah  
**Issue:** [Vets-Who-Code/vets-who-code-app #879](https://github.com/Vets-Who-Code/vets-who-code-app/issues/879)  
**Status:** 🔨 Phase III In Progress — fix implemented, verified, and pushed to fork; PR ready to submit  

---

## Why I Chose This Issue

After a heavy, architecture-first contribution (the OpenClaw provider leaf in #299), I wanted a focused, self-contained issue in a domain I care about — a well-scoped TypeScript type-safety task in a real production Next.js codebase. This issue is labeled `beginner` / `good first issue` / `type-safety` / `typescript`, which made it a good fit for learning a new repo's conventions quickly without a large surface area. It also supports a mission I connect with: #VetsWhoCode builds software to help military veterans and spouses learn to code.

More concretely, I chose it because "replace `any` with a proper type" is deceptively educational: doing it *correctly* means researching the real type the AI SDK expects, understanding how the model instances flow through the code, and proving the change compiles without weakening the public API — exactly the kind of careful, type-driven reasoning I want to get faster at.

---

## Understanding the Issue

### Problem Description

`src/lib/ai-provider.ts` is a small utility that initializes the project's AI providers (Google Gemini, Azure OpenAI, OpenAI, and Phi-3) and exposes helpers to pick a primary provider and run an operation with automatic fallback. It uses the `any` type in three places, which turns off TypeScript's type checking exactly where the model object is passed around — so mistakes using the model would not be caught at compile time, and editors give no autocomplete.

### Expected Behavior

The three `any` types are replaced with the proper type from the AI SDK, so the provider instance and the fallback helpers are fully type-checked, and TypeScript still compiles.

### Current Behavior

Three untyped `any` values (line numbers on `master` at the time of work):

- **Line 10** — `instance: any;` in the `ProviderConfig` interface (the constructed model object).
- **Line 134** — `model: any;` in the return type of `getAIModelWithFallback()`.
- **Line 164** — `operation: (model: any) => Promise<T>` — the callback parameter of `tryProvidersWithFallback<T>()`.

All three refer to the **same underlying thing**: the model instance returned by `google("…")`, `azureProvider("…")`, `openai("…")`, i.e. the value stored in `ProviderConfig.instance` and later handed to the caller's operation.

### Affected Components

- **Only file changed:** `src/lib/ai-provider.ts` (three `any` sites + one import).
- **Type source (read-only):** the `ai` package's `LanguageModel` type (AI SDK v5) and, transitively, `@ai-sdk/provider`'s `LanguageModelV2`.
- **Providers referenced:** `@ai-sdk/openai`, `@ai-sdk/azure`, `@ai-sdk/google` — the factories whose return value is being typed.

---

## Reproduction Process

### Environment Setup

- **Toolchain:** Node 24, npm (the repo uses `package-lock.json`; scripts `typecheck` = `tsc -p tsconfig.json` and `lint` = `biome lint`).
- **Setup:** forked `Vets-Who-Code/vets-who-code-app`, cloned the fork, and ran `npm install` (~1,800 packages).
- **Challenge solved:** the machine's global npm cache had root-owned files (`EACCES`), so `npm install` initially failed. Worked around it by pointing npm at a project-local cache dir (`npm install --cache <local-cache>`) rather than needing `sudo`.

### Steps to Reproduce (the type-safety gap)

1. Open `src/lib/ai-provider.ts` on `master`.
2. `grep -n ": any" src/lib/ai-provider.ts` → three hits (lines 10, 134, 164).
3. **Observed result:** the model instance flows through `ProviderConfig.instance` → `getAIModelWithFallback()` → `tryProvidersWithFallback()` entirely as `any`, so no type checking or editor assistance exists on the model object anywhere in the file.

### Reproduction Evidence

- **Fork:** [`DAmensah27/vets-who-code-app`](https://github.com/DAmensah27/vets-who-code-app)
- **Findings:** confirmed all three `any` sites describe the *same* model type, so a single imported type applied in three places resolves the whole issue — no runtime code paths change.

---

## Solution Approach

### Analysis

The root cause is simply that the model object's type was never declared. The `@ai-sdk/*` factories return a concrete model instance; in AI SDK v5 (`ai@^5.0.93`, `@ai-sdk/openai@^2.0.67`) `openai("gpt-4-turbo")` returns a `LanguageModelV2` (verified in `@ai-sdk/openai`'s `.d.ts`: `(modelId): LanguageModelV2`). So all three `any`s should be that model type.

**Research finding (a subtlety in the issue text):** the issue suggested importing `LanguageModel` from `@ai-sdk/provider`. In this repo's installed v5, `@ai-sdk/provider` actually exports `LanguageModelV2` (not `LanguageModel`), and it is only a **transitive** dependency — importing from it would be fragile. The **`ai`** package *is* a direct dependency and exports `LanguageModel = string | LanguageModelV2`, which the concrete `LanguageModelV2` instances satisfy, and which is exactly the type AI SDK functions like `generateText({ model })` consume. So the idiomatic, robust choice is `LanguageModel` from `ai`.

### Proposed Solution

Add `import type { LanguageModel } from "ai";` and replace all three `any`s with `LanguageModel`. Type-only change; no runtime behavior changes.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** three `any` sites in `ai-provider.ts` all describe the AI SDK model instance; typing them restores compile-time safety with no behavior change.

**Match:** follow the AI SDK's own public typing — `LanguageModel` from `ai` is the type the SDK exposes for a model, so consumers should use it rather than reaching into the transitive `@ai-sdk/provider`.

**Plan:**
1. Fork + clone + `npm install`; establish a clean `npm run typecheck` baseline.
2. Import `LanguageModel` from `ai`.
3. Replace the three `any`s (`ProviderConfig.instance`, the `getAIModelWithFallback()` return `model`, the `tryProvidersWithFallback()` `operation` param).
4. Run `npm run typecheck` and `npx biome lint` on the file.
5. Commit on a topic branch, push to the fork, open a PR to upstream `master`.

**Implement:** [`fix/879-ai-provider-types`](https://github.com/DAmensah27/vets-who-code-app/tree/fix/879-ai-provider-types) — commit `394e5c4`.

**Review:** self-checked against the acceptance criteria — correct `@ai-sdk` type researched, all three `any`s replaced, `tsc` compiles, no runtime paths touched; commit message conforms to the repo's commitlint config (`refactor` type, sentence-case subject, ≤72-char header).

**Evaluate:** `npm run typecheck` exits 0 with no errors; `biome lint` on the file is clean apart from a pre-existing, unrelated warning. *(Results in Testing Strategy below.)*

---

## Testing Strategy

This is a **type-only** change (the `LanguageModel` annotation is erased at compile time), so there is no new runtime behavior to unit-test; correctness is proven by the type checker and linter.

### Verification

- [x] `grep -n ": any" src/lib/ai-provider.ts` → **no matches remaining**.
- [x] `npm run typecheck` (`tsc -p tsconfig.json`) → **exit 0, no errors**.
- [x] `npx biome lint src/lib/ai-provider.ts` → clean, except one **pre-existing, unrelated** warning (`await` inside a loop at line 177 of the original file — not introduced by this change).
- [x] Diff review → exactly one added import + three `any → LanguageModel` replacements; no runtime code paths altered.

### Manual Testing

Confirmed the three usage sites still typecheck against how the value is used: the concrete `LanguageModelV2` instances (from `openai()`/`google()`/`azure()`) are assignable to `LanguageModel`, and the value continues to flow unchanged through `getAIModelWithFallback()` and `tryProvidersWithFallback()`.

---

## Implementation Notes

### Progress Summary

- **Researched the real type.** Read the installed `.d.ts` files to confirm the v5 return types and where `LanguageModel` vs `LanguageModelV2` actually live, rather than taking the issue's suggested import path at face value.
- **Made the minimal correct change.** One import + three annotations; verified with `typecheck` + `lint`.
- **Committed and pushed** to the fork on branch `fix/879-ai-provider-types` (commit `394e5c4`). PR to upstream is prepared and ready to open.

### Challenges Faced

- **The issue's suggested type was slightly off for this version.** It pointed at `LanguageModel` from `@ai-sdk/provider`; in v5 that package exports `LanguageModelV2` and is only transitive. Resolved by using `LanguageModel` from the `ai` direct dependency (documented above).
- **npm cache permissions.** Root-owned files in the global npm cache blocked `npm install`; solved with a project-local `--cache` dir instead of `sudo`.
- **Commit hooks.** The repo enforces commitlint via husky; the first commit was rejected for a non-sentence-case subject. Rewrote the message to conform (`refactor(...)`, sentence-case, ≤72 chars).

### Code Changes

- **File modified:** `src/lib/ai-provider.ts` — `+import type { LanguageModel } from "ai";` and three `any → LanguageModel` replacements.
- **Branch:** [`fix/879-ai-provider-types`](https://github.com/DAmensah27/vets-who-code-app/tree/fix/879-ai-provider-types) (commit `394e5c4`).
- **Approach decision:** use the AI SDK's public `LanguageModel` type from a direct dependency instead of the more specific `LanguageModelV2` from a transitive one — matches how the SDK is meant to be consumed and keeps the import robust to dependency changes.

---

## Pull Request

**PR Link:** _Ready to submit — branch pushed to the fork; PR against `Vets-Who-Code/vets-who-code-app:master` pending._

**PR Description (draft):**

> ### Replace `any` with `LanguageModel` in the AI provider utility (#879)
>
> **What & why.** `src/lib/ai-provider.ts` used `any` in three places that all describe the same thing — the AI SDK model instance returned by `openai()`/`google()`/`azure()`. This PR types them with `LanguageModel` from the `ai` package (AI SDK v5), restoring compile-time safety and editor autocomplete with no runtime change.
>
> **Changes.** Added `import type { LanguageModel } from "ai";` and replaced the `any` on `ProviderConfig.instance`, the `getAIModelWithFallback()` return `model`, and the `tryProvidersWithFallback()` `operation` parameter.
>
> **Note on the type.** The issue suggested `LanguageModel` from `@ai-sdk/provider`, but in the installed v5 that package exports `LanguageModelV2` and is only a transitive dependency. `ai` (a direct dependency) exports `LanguageModel = string | LanguageModelV2`, which the concrete model instances satisfy and which is the type the SDK's own APIs consume — so I imported from `ai`.
>
> **Testing.** Type-only change; `npm run typecheck` passes with no errors and `biome lint` is clean on the file.

**Maintainer Feedback:**
- _(none yet — PR not yet opened)_

**Status:** Ready to submit / Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

- **Verifying types from source, not docs.** Reading the installed `.d.ts` files (`@ai-sdk/openai`, `ai`, `@ai-sdk/provider`) to confirm what `openai()` actually returns and where each type is exported — and catching that the issue's suggested import didn't match the installed version.
- **Direct vs. transitive dependencies.** Why importing a type from a transitive package is fragile, and how to prefer the equivalent type re-exported by a direct dependency.
- **A new repo's guardrails.** Getting a clean `tsc`/`biome` baseline and satisfying a commitlint + husky commit-message policy.

### Challenges Overcome

- Diagnosed the `EACCES` npm-cache issue and worked around it without elevated permissions.
- Reconciled the issue's suggested type with the repo's actual installed SDK version, choosing the correct, robust import.

### What I'd Do Differently Next Time

- Check the installed package versions and their exported types *before* accepting an issue's suggested import path — it saves a round-trip when the issue predates a dependency bump.

---

## Resources Used

- **[Vets-Who-Code/vets-who-code-app #879](https://github.com/Vets-Who-Code/vets-who-code-app/issues/879)** — the issue, acceptance criteria, and the affected lines.
- **Vercel AI SDK v5** — the `LanguageModel` / `LanguageModelV2` types (`ai` and `@ai-sdk/provider` type declarations) and the `@ai-sdk/openai`/`azure`/`google` provider factory return types.
- **The repo's own config** — `tsconfig.json`, `biome` lint config, and `commitlint.config.js` (conventional-commits rules) that shaped the fix and the commit.
