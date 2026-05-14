# Subagent Runner Design

## Status

Draft. Phase 1 extraction is implemented in `pi-default-setup/extensions/lib/` and `markov-design.ts`; later consumers are still pending.

The current implementation uses isolated `pi --mode json` subprocesses. That is compatible with pi's CLI integration surface, but the target implementation should move toward the SDK patterns documented in pi's `docs/sdk.md`: `createAgentSession()`, explicit resource loading, in-memory sessions, direct event subscription, and caller-owned tool/resource selection.

## Problem

Two retained skills depend on real subagent orchestration, not a sequential approximation:

- `design-an-interface`
- `improve-codebase-architecture` (especially `INTERFACE-DESIGN.md`, and later the exploration phase)

Their value comes from contrast between multiple independent explorations running from different prompts and constraints. Rewriting them into a single-agent sequential workflow would preserve wording but remove the core behavior.

`extensions/markov-design.ts` already demonstrates one useful pattern:

- spawn isolated `pi` subprocesses
- give each subprocess a focused prompt and optional model override
- collect structured output through `--mode json`
- parse final assistant output back into the parent extension

That pattern is good enough to justify a reusable subagent runtime. It is not yet a reusable runtime.

## Goals

1. Preserve the original subagent-based use case of the imported skills.
2. Extract the reusable orchestration logic from `markov-design.ts`.
3. Support parallel fan-out so multiple subagents can run concurrently.
4. Default subagents to read-only exploration, not workspace mutation.
5. Keep the first API small, explicit, and extension-oriented.
6. Make `design-an-interface` the first non-Markov consumer.
7. Align the runner with pi SDK patterns once the subprocess extraction is proven.

## Non-goals

1. Build a general-purpose autonomous multi-agent framework.
2. Expose arbitrary write-capable subagents by default.
3. Reproduce another harness's exact Task/Agent tool UX.
4. Solve persistence, workflow resumes, or DAG scheduling in the first pass.
5. Add provider-level optimizations before the local runtime contract is stable.

## Existing Markov Pattern

`markov-design.ts` originally contained these reusable pieces:

- pi executable resolution via `getPiInvocation`
- child process spawn and abort handling via `runAgent`
- JSONL event parsing from `--mode json`
- final assistant text extraction via `getFinalOutput`
- focused system-prompt loading via `loadAgents`
- structured output validation boundary in the caller

It also contains Markov-specific logic that should *not* become part of the generic runtime:

- stage-specific types and validators
- Markov convergence state
- iteration loop and user action loop
- Markov renderers
- artifact generation flow

## SDK alignment target

Pi's reference docs describe the SDK as the programmatic layer for embedding agent capabilities and for building custom tools that spawn subagents. The runner should therefore evolve from a CLI-subprocess implementation to an SDK-backed implementation once the public contract is stable.

The SDK-backed runner should preserve the same `SubagentJob` / `SubagentResult` surface while changing the execution backend:

- Use `createAgentSession()` for each subagent job.
- Use `SessionManager.inMemory()` so subagent runs are ephemeral and do not write normal pi session history.
- Use an explicit `DefaultResourceLoader` configuration instead of ambient discovery.
- Do not load parent extensions, skills, prompt templates, or context files unless the job explicitly asks for them.
- Supply tools through the SDK's `tools` / `customTools` options rather than shell flags.
- Subscribe directly to session events instead of parsing JSONL subprocess output.
- Extract final assistant text from SDK events/messages, preserving the current output contract.
- Dispose each subagent session after completion.
- Keep batch orchestration and result normalization in `subagent-runner.ts`, not in individual workflows.

The SDK-backed version still needs the same safety boundary as the subprocess version: read-only by default, no write-capable tools unless a workflow explicitly opts in, and all structured outputs validated by the parent workflow.

### Compatibility phase

The current subprocess runner may remain as a compatibility backend while the SDK backend is introduced. The runner should expose one logical API and hide backend selection internally. Workflows such as Markov Design, `design-an-interface`, and architecture exploration should not know whether a job is executed by subprocess or SDK.

A later implementation can support:

```ts
type SubagentBackend = "sdk" | "subprocess";

interface SubagentRunnerOptions {
  cwd: string;
  modelRef?: string;
  signal?: AbortSignal;
  backend?: SubagentBackend;
  maxConcurrency?: number;
}
```

Default should become `"sdk"` once it matches subprocess isolation and cancellation behavior.

## Constraints

### Safety

Subagents used for architecture and interface work should not mutate the repo by default.

Default tool policy should therefore be:

- `read`
- `grep`
- `find`
- `ls`

No `write`, `edit`, or `bash` by default.

### Isolation

Each subagent must run as an isolated `pi` subprocess with:

- `--no-session`
- `--no-extensions`
- `--no-skills`
- `--no-prompt-templates`
- `--no-context-files`

This keeps the result attributable to the explicit prompt contract rather than ambient session state.

### Output contract

The parent extension must treat subprocess output as untrusted.

That means:

- parse final output explicitly
- validate JSON shape in the parent
- surface stderr and stop reason on failure
- fail one job without corrupting the whole batch state

## Proposed Design

The runner has two layers:

1. A stable orchestration contract (`SubagentJob`, `SubagentResult`, `runSubagent`, `runSubagents`).
2. A replaceable execution backend.

The first implementation uses CLI subprocesses because Markov already proved that pattern. The target backend should use the pi SDK while keeping the same orchestration contract.

## File layout

```text
pi-default-setup/
  docs/
    subagent-runner-design.md
  extensions/
    lib/
      pi-subprocess.ts
      sdk-subagent.ts
      subagent-runner.ts
      subagent-output.ts
    markov-design.ts
```

### `pi-subprocess.ts`

Compatibility child-process runner for `pi`.

Responsibilities:

- resolve how to invoke `pi`
- spawn child process
- apply cwd/model/system prompt/tool flags
- stream stdout/stderr
- support abort propagation
- return raw execution result

This module should know nothing about Markov, architecture review, or interface design. It should eventually become a fallback backend, not the default implementation.

### `sdk-subagent.ts`

Target SDK-backed executor for one subagent job.

Responsibilities:

- create one isolated `AgentSession` with `createAgentSession()`
- use `SessionManager.inMemory()` for ephemeral execution
- construct an explicit resource loader so ambient extensions, skills, prompt templates, and context files do not leak into subagents
- map tool presets to SDK tool configuration
- subscribe to session events and capture assistant output, stop reason, errors, and timing
- abort the session when the parent `AbortSignal` fires
- dispose the session after completion

This module should provide the same raw execution information that the subprocess backend provides, but without JSONL parsing or process management.

### `subagent-output.ts`

Helpers for:

- extracting final assistant text from SDK messages or JSONL events
- parsing fenced JSON output
- normalizing errors
- timing and metadata capture

### `subagent-runner.ts`

Mid-level orchestration for one or many subagent jobs.

Responsibilities:

- accept a typed job spec
- choose the SDK or subprocess backend
- run one or many jobs
- enforce concurrency limit
- return normalized results

## Core types

```ts
type ToolPreset = "none" | "read-only" | { tools: string[] };

type ExpectedOutput = "text" | "json";

interface SubagentJob {
  id: string;
  prompt: string;
  systemPrompt?: string;
  model?: string;
  cwd?: string;
  tools?: ToolPreset;
  expected?: ExpectedOutput;
  timeoutMs?: number;
  metadata?: Record<string, string>;
}

interface SubagentResult {
  id: string;
  ok: boolean;
  text: string;
  json?: Record<string, unknown>;
  stderr: string;
  exitCode: number;
  stopReason?: string;
  errorMessage?: string;
  durationMs: number;
  metadata?: Record<string, string>;
}
```

## API surface

First-pass API should stay internal to extension code:

```ts
async function runSubagent(job: SubagentJob, options?: {
  cwd: string;
  modelRef?: string;
  signal?: AbortSignal;
}): Promise<SubagentResult>

async function runSubagents(jobs: SubagentJob[], options?: {
  cwd: string;
  modelRef?: string;
  signal?: AbortSignal;
  maxConcurrency?: number;
}): Promise<SubagentResult[]>
```

No generic LLM-callable tool in the first pass.

Reason: the right first consumers are extension-controlled workflows with explicit prompts and validation, not arbitrary model-issued subprocess trees.

## Tool presets

### `none`

Equivalent to current Markov behavior.

Use for:

- structured writer agents
- pure synthesis agents
- stages that only consume parent-provided context

### `read-only`

Maps to:

```text
--tools read,grep,find,ls
```

Use for:

- architecture exploration
- interface-design exploration
- codebase reconnaissance

### explicit tools

Reserve for later. Needed only if a future subagent needs a non-default safe shape.

## Execution model

### Single job — SDK target

1. Build the subagent prompt and system prompt from the job spec.
2. Create an isolated SDK session with in-memory session management.
3. Load only explicitly requested resources.
4. Provide only tools allowed by the job's tool preset.
5. Subscribe to session events.
6. Prompt the session and wait for completion.
7. Capture final assistant text, stop reason, error state, and duration.
8. If `expected === "json"`, parse and return `json`.
9. Dispose the session.
10. Return normalized `SubagentResult`.

### Single job — subprocess compatibility

1. Build subprocess args.
2. Spawn child.
3. Read JSONL event stream.
4. Capture assistant messages and stderr.
5. On completion, extract final assistant text.
6. If `expected === "json"`, parse and return `json`.
7. Return normalized `SubagentResult`.

### Batch jobs

1. Accept `jobs[]`.
2. Run with bounded concurrency.
3. Preserve stable result ordering by input job order.
4. Do not fail fast on the first job failure by default.
5. Return all results so the caller can decide whether partial success is usable.

Rationale: interface design and architecture exploration often benefit from seeing *all* candidate outputs, even if one fails.

## Parent workflow responsibilities

The subagent runner should stay narrow. Callers still own:

- prompt construction
- result validation
- result comparison
- user-facing rendering
- retry policy
- deciding whether partial batch failure is acceptable

This keeps the runtime generic without turning it into a workflow engine.

## Integration plan

### Phase 1: Markov extraction — implemented

Refactor `markov-design.ts` to use:

- `pi-subprocess.ts`
- `subagent-output.ts`
- `subagent-runner.ts`

Behavior should remain unchanged.

Success criteria:

- no user-visible Markov regression
- no behavioral change in stage contracts
- helper API proven by a real consumer

### Phase 2: Parallel batch support — implemented

Add `runSubagents()` with bounded concurrency.

Success criteria:

- multiple subprocesses can run concurrently
- cancellation propagates to all children
- result order remains deterministic

### Phase 3: SDK backend

Add an SDK-backed executor while preserving the existing runner API.

Expected shape:

1. create one `AgentSession` per subagent job
2. use `SessionManager.inMemory()`
3. use explicit resource loading and tool selection
4. subscribe to events directly
5. normalize output into the existing `SubagentResult`
6. keep subprocess execution available as a fallback until SDK parity is proven

Success criteria:

- SDK backend can run Markov stages without changing Markov workflow code
- output contract matches subprocess backend
- cancellation propagates to active SDK sessions
- no ambient skills/extensions/context files leak into subagent prompts
- read-only tool preset maps to the intended SDK tool set

### Phase 4: `design-an-interface`

Implement a pi-native workflow backed by the runner.

Expected shape:

1. parent gathers requirements
2. parent spawns 3-4 design jobs with distinct constraints
3. all jobs run in parallel with read-only tools
4. parent presents results sequentially
5. parent compares and recommends

Success criteria:

- retains original multiple-design contrast
- no sequential fallback hidden behind the same name

### Phase 5: `improve-codebase-architecture` interface design branch

Use the same runner for `INTERFACE-DESIGN.md`.

Expected shape:

1. parent frames chosen deepening candidate
2. parent spawns multiple interface-design subagents in parallel
3. parent compares by depth, locality, and seam placement

### Phase 6: `improve-codebase-architecture` exploration phase

Extend runner usage into exploration itself.

Expected shape:

- separate subagents with different exploration lenses
- aggregate findings into candidate list
- preserve ADR/domain-language guidance

## Failure modes

### One subagent fails

Return a failed `SubagentResult` for that job.
Caller decides whether to:

- ignore it
- retry it
- stop whole workflow

### All subagents fail

Treat as parent workflow failure.
Surface:

- job ids
- stderr summaries
- stop reasons
- first parsed error message if available

### Malformed JSON

If `expected === "json"` and parsing fails:

- mark job failed
- keep raw text for debugging
- do not coerce partial JSON into success

### Parent abort

Abort should:

- terminate all in-flight children
- stop launching queued jobs
- return quickly

### Tool misuse

Architecture/design subagents should default to read-only. Any future write-capable mode should be explicit and opt-in.

## Why not expose a generic `spawn_subagents` tool first?

Because the first risk is not implementation difficulty, but misuse.

A generic tool callable by the model would make it easy to:

- spawn unnecessary subprocess trees
- create overlapping write activity
- hide workflow logic inside prompts with no stable validation layer

The extension should first prove the pattern through fixed workflows, then consider a public tool later.

## Open questions

1. Should subagents inherit the parent model by default, or should each workflow declare its own model explicitly?
2. Should `read-only` include `bash` in a sandboxed form later, or stay restricted to file-inspection tools only?
3. Should batch execution support per-job cwd overrides in the first version, or only one shared cwd?
4. Should the runner expose intermediate status updates to UI, or should callers simply set coarse-grained status labels?
5. Should we store subagent transcripts for debugging, or keep only normalized results in the first version?
6. What is the exact SDK resource-loader configuration that best mirrors `--no-extensions --no-skills --no-prompt-templates --no-context-files`?
7. Should the SDK backend use built-in tool factories or explicit tool instances for cwd-bound read-only tools?
8. How should SDK stop/error events be mapped to the existing `stopReason` and `errorMessage` fields?

## Recommendation

Keep the current subprocess extraction as the compatibility baseline, but treat the SDK-backed runner as the target architecture.

Do not rewrite subagent-dependent skills into sequential approximations.
Do not expose a generic model-callable subagent tool yet.
Do not let workflows depend on backend details.
Prove SDK parity with Markov first, then port `design-an-interface`, then port the architecture skills that depend on true multi-agent contrast.
