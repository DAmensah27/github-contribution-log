# github-contribution-log
# Contribution [1]: [Adapter: OpenClaw]

**Contribution Number:** 1  
**Student:** Darin Andoh-Mensah  
**Issue:** [orthogonalhq/nous-core #299](https://github.com/orthogonalhq/nous-core/issues/299)  
**Status:** Phase III Complete  

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

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
