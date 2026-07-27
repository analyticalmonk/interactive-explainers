# Research notes: embedding OpenAI Codex (app-server + SDK)

Date: 2026-07-21
Researcher: Task 1 (research and source notes) for the Codex embedding explainer.

**Purpose:** This file is the sole factual source for the rest of the Codex-embedding
explainer build. Figure 2's `STEPS` data (Task 5) and the SDK code samples (Task 6) must
use the names recorded here verbatim. Every bullet ends with its source URL. Where a
detail could not be confirmed, that gap is stated explicitly instead of guessed.

**Pinning note:** GitHub file citations use `raw.githubusercontent.com` URLs pinned to
commit `d5998e74522245ce6e7b746dab6c5f0ed428a8d1` of `openai/codex` (HEAD of `main` at
fetch time, 2026-07-21T07:55:56Z), so the quoted text stays reproducible even as `main`
moves. `learn.chatgpt.com` pages are undated/versionless in the CMS; fetched 2026-07-21.

**Access notes:** `developers.openai.com/codex/*` URLs 308-redirect to
`learn.chatgpt.com/docs/*`; both are cited where a redirect occurred. Several
`openai.com/index/*` blog posts and the human-facing `npmjs.com` and `bloomberg.com`
pages returned Cloudflare/bot-check pages (HTTP 403 or a JS challenge shell) to both
WebFetch and direct `curl`, so those are cited as corroborating sources (via search
snippet or an accessible mirror/API) rather than as directly-quoted primary fetches;
each such case is flagged inline below.

---

## App-server protocol

- The app-server is launched with the subcommand `codex app-server`. Default transport
  is stdio (`--stdio` or `--listen stdio://`); other transports are `--listen ws://IP:PORT`
  (websocket, marked "experimental / unsupported"), `--listen unix://` or
  `--listen unix://PATH` (websocket-over-unix-socket via HTTP Upgrade handshake), and
  `--listen off` (no local transport).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Transport is JSON-RPC 2.0 ("similar to MCP"), with the `"jsonrpc":"2.0"` header
  omitted on the wire; stdio transport is newline-delimited JSON (JSONL).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Core primitives: **Thread** (a conversation, contains multiple turns), **Turn** (one
  turn of the conversation, contains multiple items), **Item** (user/agent
  inputs/outputs - user message, agent reasoning, agent message, shell command, file
  edit, etc.).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Initialization handshake: the client must send a single `initialize` request per
  transport connection (with `clientInfo`: `name`, `title`, `version`) before invoking
  any other method; the server responds with the user-agent string, `codexHome`,
  `platformFamily`, `platformOs`; the client then sends an `initialized` notification.
  Any other request before this handshake gets a `"Not initialized"` error; a repeated
  `initialize` on the same connection gets `"Already initialized"`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Thread lifecycle methods: `thread/start` (create; also accepts `ephemeral: true` for
  an in-memory thread), `thread/resume` (reopen by id), `thread/fork` (branch to a new
  thread id with copied history), `thread/list`, `thread/read`, `thread/archive`,
  `thread/unarchive`. `thread/start` and `thread/resume`/`thread/fork` all emit a
  `thread/started` notification.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Turn lifecycle methods: `turn/start` (send user input; targets a `threadId`, returns
  the new turn object immediately with `status: "inProgress"`), `turn/steer` (append
  input to an active turn), `turn/interrupt` (cancel a running turn). The server emits
  `turn/started` when the turn actually begins running and `turn/completed` (status
  `completed` | `interrupted` | `failed`) when it ends.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Streaming: after `turn/start`, the client keeps reading JSON-RPC notifications on
  stdout: `item/started`, `item/completed`, and item-specific deltas such as
  `item/agentMessage/delta` (streamed agent text - concatenate `delta` values per
  `itemId`). Per-item lifecycle is always `item/started` -> zero-or-more deltas ->
  `item/completed`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Approval round-trip (server-initiated request to the client): for a pending shell
  command, the sequence is `item/started` (pending `commandExecution` item) ->
  `item/commandExecution/requestApproval` (request, carries `threadId`, `turnId`,
  `itemId`, `command`, `cwd`, `commandActions`, `reason?`) -> client responds with a
  single `{ "decision": ... }` payload (`accept`, `acceptForSession`,
  `acceptWithExecpolicyAmendment`, `applyNetworkPolicyAmendment`, `decline`, or
  `cancel`) -> `serverRequest/resolved` notification (`{ threadId, requestId }`) ->
  `item/completed` with final status (`completed` | `failed` | `declined`). The
  equivalent for a proposed edit uses `item/fileChange/requestApproval` in the same
  position. The turn blocks/pauses until the client answers.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Approval routing policy: `approvalsReviewer` on `turn/start` accepts `"user"`
  (default - review directly in the client) or `"auto_review"` (route to an automatic
  reviewer subagent that applies a risk-based decision framework; legacy alias
  `"guardian_subagent"` still accepted).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Runtime auth (JSON-RPC methods, no separate REST layer): `account/read` (current
  account info, optional token refresh), `account/login/start` (`type`: `apiKey`,
  `chatgpt`, `chatgptDeviceCode`, or experimental `amazonBedrock`), `account/logout`,
  plus notifications `account/login/completed` and `account/updated` (`authMode`:
  `apikey` | `bedrockApiKey` | `chatgpt` | `personalAccessToken` | `null`).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- Backpressure: when request ingress is saturated, new requests are rejected with
  JSON-RPC error code `-32001`, message `"Server overloaded; retry later."`; clients
  should retry with exponential backoff + jitter.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- The message schema can be dumped for a given Codex build with
  `codex app-server generate-ts --out DIR` (TypeScript) or
  `codex app-server generate-json-schema --out DIR` (JSON Schema).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- **Important - do not conflate with MCP server mode:** `codex mcp-server` (or
  `codex-mcp-server`) is a *separate*, explicitly "experimental" interface that speaks
  MCP-flavored JSON-RPC (`thread/start`, `turn/start`, etc. are shared v2 method names,
  plus legacy v1 compatibility methods `getConversationSummary`, `getAuthStatus`,
  `gitDiffToRemote`, `fuzzyFileSearch*`). Critically, **its approval request method
  names differ from the app-server's**: this doc names them `applyPatchApproval` and
  `execCommandApproval` (client replies `{ decision: "allow" | "deny" }`), not
  `item/fileChange/requestApproval` / `item/commandExecution/requestApproval` /
  `{ decision: "accept" | "decline" | ... }` as in the main app-server README above.
  Figure 2 (host <-> app-server <-> agent) must use the app-server names, not these.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/docs/codex_mcp_interface.md

## SDK (TypeScript)

- Package: `@openai/codex-sdk`. Install: `npm install @openai/codex-sdk`. Requires
  Node.js 18+ (`engines.node: ">=18"`).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/package.json
- Package identity/latest version confirmed directly against the npm registry API
  (the human npmjs.com package page returned HTTP 403 to both WebFetch and curl, so the
  registry JSON is cited instead): name `@openai/codex-sdk`, `dist-tags.latest` =
  `0.144.6` as of 2026-07-21.
  Source: https://registry.npmjs.org/@openai/codex-sdk (npm page for reference, not
  directly fetchable: https://www.npmjs.com/package/@openai/codex-sdk)
- **The TypeScript SDK does not speak the app-server JSON-RPC protocol directly.** Per
  its own README: "The TypeScript SDK wraps the `codex` CLI from `@openai/codex`. It
  spawns the CLI and exchanges JSONL events over stdin/stdout." The source confirms
  this spawns `codex exec --experimental-json` (an alias of `--json`, see
  `codex exec` baseline section below), not `codex app-server`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/src/exec.ts
  (`const commandArgs: string[] = ["exec", "--experimental-json"];`)
- Because it rides on `codex exec`'s event stream, the TS SDK's underlying event names
  are the `codex-rs/exec` schema - `thread.started`, `turn.started`, `turn.completed`,
  `turn.failed`, `item.started`, `item.updated`, `item.completed`, `error` (dotted,
  lowercase) - which is a **different, simpler naming scheme** than the app-server's
  slash-based `thread/started`, `turn/started`, `item/started`, etc. Do not present
  these as the same wire protocol in the article.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/exec/src/exec_events.rs
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/src/events.ts
  (comment: "based on event types from codex-rs/exec/src/exec_events.rs")
- Minimal "start thread, run prompt" (verbatim from the README):
  ```typescript
  import { Codex } from "@openai/codex-sdk";

  const codex = new Codex();
  const thread = codex.startThread();
  const turn = await thread.run("Diagnose the test failure and propose a fix");

  console.log(turn.finalResponse);
  console.log(turn.items);
  ```
  Calling `thread.run(...)` again on the same `Thread` continues that conversation.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
- Resume an existing thread by id (threads persist under `~/.codex/sessions`):
  ```typescript
  const savedThreadId = process.env.CODEX_THREAD_ID!;
  const thread = codex.resumeThread(savedThreadId);
  await thread.run("Implement the fix");
  ```
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
- Streaming: `thread.run()` buffers events until the turn finishes; `runStreamed()`
  returns `{ events }`, an async generator of structured events (`item.completed`,
  `turn.completed`, etc. - SDK-level, not raw JSON-RPC).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
- Sandbox control: the TS SDK has **no first-class `Sandbox` enum** (unlike Python).
  `CodexExecArgs.sandboxMode` maps straight to the CLI's `--sandbox` flag, and finer
  control goes through the generic `config` option, e.g.
  `config: { sandbox_workspace_write: { network_access: true } }`, flattened to
  `--config` overrides.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/src/exec.ts
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md
- **What it does NOT expose - approval hooks:** `codex exec`'s JSONL event enum
  (`ThreadEvent` in `exec_events.rs`) has **no approval-request event variant at all**.
  Approval behavior is fixed at spawn time via a static `approvalPolicy` CLI/config
  flag (`CodexExecArgs.approvalPolicy` -> `--config approval_policy="..."`); there is
  no runtime callback the host implements to answer an individual approval request.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/exec/src/exec_events.rs
  (enumerated variants: `thread.started`, `turn.started`, `turn.completed`,
  `turn.failed`, `item.started`, `item.updated`, `item.completed`, `error` - none named
  approval) and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/src/exec.ts
- **What it does NOT expose - event granularity:** no equivalent of the app-server's
  `item/agentMessage/delta` text-delta notification is named in the README; streaming
  is described at the "structured events" level (`item.completed`, etc.), not
  per-character/per-token deltas.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/typescript/README.md

## SDK (Python)

- Package: `openai-codex`. Install: `pip install openai-codex`. Requires Python
  `>=3.10`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/pyproject.toml
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/README.md
- PyPI project page confirms the same package: https://pypi.org/project/openai-codex/
  (info.summary: "Python SDK for Codex"; verified via `https://pypi.org/pypi/openai-codex/json`).
  Note: candidate names `openai-codex-sdk`, `codex-sdk`/`codex_sdk` also exist on PyPI
  but are unrelated third-party packages (`codex-sdk` belongs to `cleanlab-codex`) -
  `openai-codex` is the correct one, confirmed against the repo's own `pyproject.toml`.
  Source: https://pypi.org/pypi/openai-codex/json
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/pyproject.toml
- The package depends on a pinned CLI runtime, `openai-codex-cli-bin` (pinned to
  `==0.144.4` at research time - "SDK release versions track the corresponding Codex
  CLI release").
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/pyproject.toml
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/getting-started.md
- **Unlike the TypeScript SDK, the Python SDK talks the real app-server protocol.**
  `CodexClient` is documented in its own docstring as "Synchronous typed JSON-RPC
  client for `codex app-server` over stdio," and its `start()` method spawns
  `[codex_bin, ..., "app-server", "--listen", "stdio://"]` as a subprocess.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/src/openai_codex/client.py
- Minimal "start thread, run prompt" (verbatim from the README):
  ```python
  from openai_codex import Codex

  with Codex() as codex:
      thread = codex.thread_start()
      result = thread.run("Explain this repository in three bullets.")
      print(result.final_response)
  ```
  `thread.run(...)` returns a `TurnResult` with `final_response`, `items`, and `usage`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/README.md
- Resume an existing thread by id:
  ```python
  with Codex() as codex:
      thread = codex.thread_resume("thr_123")
      print(thread.run("Continue where we left off.").final_response)
  ```
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/getting-started.md
- Sandbox-permission API: `from openai_codex import Codex, Sandbox`, then
  `codex.thread_start(sandbox=Sandbox.workspace_write)`. Presets: `Sandbox.read_only`
  (read without writes), `Sandbox.workspace_write` (default for trusted projects; read
  + write inside the workspace/configured writable roots), `Sandbox.full_access` (no
  filesystem restriction). The same `sandbox=` kwarg works on `thread.run(...)` /
  `thread.turn(...)` as a per-turn override.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/getting-started.md
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/api-reference.md
- Approval control is policy-based: `thread_start`/`thread_resume`/`thread_fork` accept
  `approval_mode` (type `ApprovalMode`), defaulting to `ApprovalMode.auto_review`
  (maps to app-server `AskForApproval.on_request` + `ApprovalsReviewer.auto_review`) or
  `ApprovalMode.deny_all` (maps to `AskForApproval.never`).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/src/openai_codex/_approval_mode.py
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/api-reference.md
- **What it does NOT expose - a public approval-decision hook:** the public `Codex`
  class constructor takes only `config: CodexConfig | None`, and `CodexConfig` (the
  public config dataclass) has no approval-callback field. Internally,
  `CodexClient.__init__` *does* accept a private `approval_handler` parameter and has a
  `_default_approval_handler` method that auto-answers `item/commandExecution/requestApproval`
  and `item/fileChange/requestApproval` - but `Codex.__init__` constructs
  `CodexClient(config=config)` without ever forwarding a custom handler. So a host
  application cannot supply its own per-request "allow/deny this specific command"
  logic through the documented public API; it can only choose between the two
  `ApprovalMode` policies above.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/src/openai_codex/api.py
  (`class Codex.__init__(self, config: CodexConfig | None = None)`) and
  https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/src/openai_codex/client.py
  (`approval_handler: ApprovalHandler | None = None` parameter and
  `_default_approval_handler`, not reachable from the public `Codex` class)
- Event stream granularity: `Thread.run(...)` returns only the final `TurnResult`;
  `Thread.turn(...)` returns a `TurnHandle` whose `.stream()` yields raw `Notification`
  objects - i.e., because this SDK talks the real app-server protocol, `stream()` is
  finer-grained than the TypeScript SDK's `runStreamed()` (which is limited to the
  `codex exec` event schema).
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/api-reference.md
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/sdk/python/docs/faq.md

## codex exec and CLI baseline

- `codex exec "<prompt>"` runs Codex non-interactively/headlessly (no TTY loop),
  streaming to stdout and optionally resuming a previous session.
  Source: https://learn.chatgpt.com/docs/non-interactive-mode
- `--json` is the canonical flag for JSONL event output; `--experimental-json` is
  defined as a `clap` **alias** of the same flag (confirmed directly in source: `#[arg(long = "json", alias = "experimental-json", ...)] pub json: bool`) -
  they are not two different features.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/exec/src/cli.rs
- Other confirmed `codex exec` flags (from the CLI source): `--output-last-message
  <FILE>` (write final message to a file), `--output-schema <FILE>` (write a
  schema-conformant JSON response), `--ephemeral` (skip persisting session rollout
  files), `--skip-git-repo-check`, `--ignore-user-config`, `--ignore-rules`,
  `--full-auto`, `--image <PATH>` (attach an image), `--color`.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/exec/src/cli.rs
- Resume a previous `codex exec` session: `codex exec resume --last` (most recent in
  the directory) or `codex exec resume <SESSION_ID>`.
  Source: https://learn.chatgpt.com/docs/non-interactive-mode
- `codex exec`'s JSONL event schema (`ThreadEvent`, dotted/lowercase names):
  `thread.started`, `turn.started`, `turn.completed`, `turn.failed`, `item.started`,
  `item.updated`, `item.completed`, `error`. This is the same schema the TypeScript SDK
  consumes (see SDK section above) - it is **not** the app-server's slash-named
  protocol.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/exec/src/exec_events.rs
- Interactive CLI runs an "agent loop": iterative inference/tool-call cycles per turn
  (a single user message can trigger many tool calls) until a final assistant message
  ends the turn; the CLI uses prefix caching on the Responses API and compacts
  conversation history once a token threshold is exceeded.
  Note: this description is sourced from an OpenAI devrel blog post
  (`openai.com/index/unrolling-the-codex-agent-loop/`) that returned an HTTP 403
  bot-check to direct WebFetch/curl; the summary above is corroborated only via the
  WebSearch tool's cached snippet of that page, not a directly-quoted fetch - treat
  turn-loop mechanics as reasonably confirmed but not verbatim-sourced.
  Source: https://openai.com/index/unrolling-the-codex-agent-loop/ (fetch blocked;
  cited via search snippet)
- Sandbox modes (config key `sandbox_mode`, CLI flag `--sandbox`): `read-only` (agent
  can inspect but not edit/run without approval), `workspace-write` (default; read +
  edit within the workspace + run routine local commands inside that boundary),
  `danger-full-access` (no filesystem/network sandboxing).
  Source: https://learn.chatgpt.com/docs/sandboxing
- Approval policies (config key `approval_policy`, CLI flag `--ask-for-approval`):
  `untrusted` (ask before anything outside a trusted command set), `on-request`
  (work inside the sandbox by default, ask only to go beyond it), `never` (no approval
  prompts). A related `granular` policy form and `approvals_reviewer` /
  `sandbox_workspace_write.writable_roots` config keys are also documented.
  Source: https://learn.chatgpt.com/docs/agent-approvals-security
  and https://learn.chatgpt.com/docs/sandboxing
- CLI defaults: running `codex` with no flags uses the `Auto` preset, i.e.
  `--sandbox workspace-write --ask-for-approval on-request` (the agent reads/edits/
  runs commands inside the workspace on its own, and asks only to edit files outside
  the workspace or reach the network). On launch Codex detects version control and
  recommends `Auto` (workspace-write + on-request) for version-controlled folders and
  `read-only` for non-version-controlled folders. So the default approval policy for
  the interactive CLI is `on-request` and the default sandbox is `workspace-write`.
  Source: https://learn.chatgpt.com/docs/agent-approvals-security
- Sandboxing and approval are explicitly two different, orthogonal controls: "the
  sandbox defines technical boundaries. The approval policy decides when the agent
  must stop and ask before crossing them."
  Source: https://learn.chatgpt.com/docs/agent-approvals-security
- The repo's own `docs/sandbox.md` is a one-line stub redirecting to this same
  documentation, confirming `learn.chatgpt.com/docs/sandboxing` (redirect target of
  `developers.openai.com/codex/security`) as the canonical source rather than an
  in-repo protocol doc.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/docs/sandbox.md

## Decision map inventory

- **Codex CLI** - terminal-based interface to "inspect code, make changes, run
  commands, and automate repeatable work without leaving your terminal"; runs the
  interactive agent loop with sandbox + approval-mode controls per session.
  Source: https://learn.chatgpt.com/docs/codex/cli
- **Codex IDE extension** (VS Code, JetBrains) - brings open files/selections into the
  prompt, lets you review edits in place inside the editor, and can hand off longer
  work to Codex cloud without leaving the IDE; built on the app-server protocol (same
  interface documented in the App-server section above powers "rich interfaces such as
  the Codex VS Code extension").
  Source: https://learn.chatgpt.com/docs/codex/ide
  and https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- **Codex desktop app** - now merged into the ChatGPT desktop app (macOS/Windows, as of
  July 9); a "command center" to run multiple projects/agents in parallel and keep
  long-running work moving from one workspace.
  Source: https://learn.chatgpt.com/docs/app (redirect target of
  https://developers.openai.com/codex/app)
- **Codex cloud** - runs coding tasks in isolated, parallel cloud environments; you
  dispatch work and review the result asynchronously from the web, GitHub, Linear, or
  Slack, instead of tying up a local machine.
  Source: https://learn.chatgpt.com/docs/cloud
- **Codex SDK** (TypeScript and Python) - "Programmatically control local Codex agents"
  from your own code: start/resume threads, run prompts, stream progress, for
  CI/CD integration or building your own agent host.
  Source: https://learn.chatgpt.com/docs/codex-sdk
- **`codex app-server`** - the stdio JSON-RPC interface Codex uses to power rich
  clients; the first-class embedding path when you need a non-Node/non-Python host, a
  custom UI over the full event stream, or your own approval UX.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
- **`codex exec`** - non-interactive, scriptable one-shot invocation for CI pipelines
  and automation; JSONL event output via `--json`.
  Source: https://learn.chatgpt.com/docs/non-interactive-mode
- **`codex mcp-server`** - a separate, explicitly experimental interface that exposes
  Codex as an MCP-compatible JSON-RPC server so MCP clients (e.g., other agents) can
  call it as a tool.
  Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/docs/codex_mcp_interface.md
- **Codex GitHub Action** (`openai/codex-action`) - drops into a CI workflow in about
  ten lines of YAML; installs the Codex CLI on the runner and runs `codex exec`
  headlessly to post automated PR code review / gate changes. It is a wrapper around
  `codex exec`, not a distinct ninth protocol - included here because Figure 1's
  automation band names it separately from generic `exec`.
  Source: https://github.com/openai/codex-action
  and https://learn.chatgpt.com/docs/code-review

## Runtime trajectory (cloud, Ona)

- OpenAI's own Newsroom account announced the Ona acquisition on 2026-06-11 with a
  direct quote: "We've reached an agreement to acquire @ona_hq. Its secure cloud
  execution technology will help Codex take on longer-running work, even when laptops
  are closed, and help more organizations deploy agents securely in production. After
  closing, Ona will join OpenAI's Codex team."
  Source: https://x.com/OpenAINewsroom/status/2065088002335158753
- The corresponding OpenAI blog post exists at the URL below but returned a Cloudflare
  JS-challenge page to both WebFetch and direct curl (no article text retrievable);
  its existence/headline ("OpenAI to acquire Ona") is confirmed via WebSearch result
  metadata only, not a direct quote-level fetch.
  Source: https://openai.com/index/openai-to-acquire-ona/ (fetch blocked; cited via
  search snippet)
- Ona's own announcement (accessible, fetched directly) describes Ona as providing
  "trusted, customer-controlled cloud environments where work continues across
  devices, inside the systems where software actually lives," with "reproducible
  environments, repeatable automations, deployment inside the customer's cloud, scoped
  credentials, audit trails, agent orchestration, and runtime AI security," plus
  automatic checkpointing (roughly every 10 minutes) so a long task survives a dropped
  connection or a closed laptop.
  Source: https://ona.com/stories/ona-joins-openai
- Ona was formerly known as Gitpod; deal announced 2026-06-11. Secondary reporting
  (CNBC, Bloomberg) corroborates the date and the "run Codex agents for hours without
  your laptop on" framing, though both outlet pages returned bot-check/paywall
  responses to direct fetch and are cited via WebSearch snippets rather than
  quote-level access.
  Source: https://www.cnbc.com/2026/06/11/open-ai-ona-acquisition-codex.html
  (fetch blocked; cited via search snippet) and
  https://www.bloomberg.com/news/articles/2026-06-11/openai-to-acquire-cloud-platform-ona-to-support-ai-agents
  (fetch blocked; cited via search snippet)
- **EDITORIAL SYNTHESIS (not sourced to a single citation - my own inference from the
  bullets above, not a fact from any one source):** what this signals for embedders is
  that the thread/turn model and the app-server/SDK client surface documented above
  stay the interface; what changes is *where* the agent executes - moving toward
  customer-controlled, persistent, checkpointed cloud runtimes rather than a process
  tied to one open laptop. Task 5/7 authors should treat this bullet as framing/opinion
  to adapt in prose, not as a citable fact to restate as-is.

## Figure 2 message sequence

Verbatim ordered sequence for **one full app-server turn, including a command-execution
approval round-trip**, drawn entirely from the App-server protocol section above (all
names sourced from
https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md).
Direction is `host -> app-server` (request the host sends) or `app-server -> host`
(notification/request the server sends). Steps 1-2 are one-time per connection, not
per turn; steps 3 onward are the per-turn sequence.

1. `initialize` (request, host -> app-server) - host sends `clientInfo` (name/title/version).
2. `initialized` (notification, host -> app-server) - host acknowledges; server is now
   ready to accept other methods.
3. `thread/start` (request, host -> app-server) - response returns the new thread object.
4. `thread/started` (notification, app-server -> host) - confirms the thread is open.
5. `turn/start` (request, host -> app-server) - targets `threadId`, carries user
   `input`; response returns the new turn object with `status: "inProgress"`.
6. `turn/started` (notification, app-server -> host) - the turn begins running.
7. `item/started` (notification, app-server -> host) - a pending `commandExecution`
   item appears (the agent wants to run a shell command).
8. `item/commandExecution/requestApproval` (request, app-server -> host) - **approval
   round-trip begins**; carries `threadId`, `turnId`, `itemId`, `command`, `cwd`,
   `commandActions`, optional `reason`. The turn blocks here until the host answers.
9. Host reply (response, host -> app-server) - `{ "decision": "accept" }` (or
   `"decline"` / `"cancel"` / `"acceptForSession"` / etc.).
10. `serverRequest/resolved` (notification, app-server -> host) - `{ threadId,
    requestId }`, confirms the pending approval request is resolved.
11. `item/completed` (notification, app-server -> host) - the `commandExecution` item
    finalizes with `status: "completed"` (or `"failed"` / `"declined"`) and its output.
12. `item/started` (notification, app-server -> host) - a new `agentMessage` item
    begins (empty/partial text so far). Not shown in a worked example in the source,
    but required by the general rule stated for *all* items: "The per-item lifecycle
    is always: `item/started` → zero or more item-specific deltas → `item/completed`"
    and "All items emit shared lifecycle events" (`item/started`, `item/completed`).
    Inserted here on that basis rather than a specific agentMessage worked example -
    see the note directly below the source citation for exact wording and location.
13. `item/agentMessage/delta` (notification, app-server -> host) - the same
    agentMessage item's reply streams in as text deltas (repeats; concatenate by
    `itemId`).
14. `item/completed` (notification, app-server -> host) - the `agentMessage` item
    finalizes with the full accumulated text.
15. `turn/completed` (notification, app-server -> host) - the turn ends with
    `status: "completed"` and token usage.

**On step 12:** the source's worked examples for approvals (command-execution,
file-change) both open with `item/started` for *that* item, and the source states the
per-item lifecycle rule twice - once specifically for turn events ("Each turn emits
`turn/started` ... The per-item lifecycle is always: `item/started` → zero or more
item-specific deltas → `item/completed`") and once under "Items" ("All items emit
shared lifecycle events: `item/started` ... `item/completed`"). Neither passage carves
out an exception for `agentMessage`, and the `agentMessage` item type is listed in the
same "Items" enumeration those shared lifecycle events apply to. No worked example in
the fetched README shows the literal sequence `item/started` immediately followed by
`item/agentMessage/delta`, so step 12 is inserted by direct application of the
documented general rule, not copied from a specific example - flagged here so a
reviewer can re-check it against the primary source if that distinction matters.
Source: https://raw.githubusercontent.com/openai/codex/d5998e74522245ce6e7b746dab6c5f0ed428a8d1/codex-rs/app-server/README.md
(Turn events section, the `turn/started`/`turn/completed` paragraph and the "Items"
subsection immediately below it).

Note for Task 5/6: this is the app-server's own naming. The Python SDK talks this exact
protocol under the hood (see SDK (Python) section), so its `sdkAbsorbed` framing is
accurate - the SDK's `thread_start()`/`thread.run()` calls absorb steps 1-6 and 8-15
into a single blocking call, only surfacing the final `TurnResult` (or raw
`Notification`s via `TurnHandle.stream()`), and the public API has no way to answer
step 8/9 itself (see the Python "what it does NOT expose" bullet). The TypeScript SDK,
by contrast, does not run this protocol at all - it wraps `codex exec`, whose event
schema (see `codex exec` section) has no approval step or slash-named events; a
"custom host vs SDK as host" toggle should either caveat this asymmetry explicitly or
scope the toggle's "SDK as host" state to the Python SDK's behavior, since the two
SDKs are not interchangeable with respect to this figure.
