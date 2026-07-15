# Reasonix 工程规范

> Reasonix 是采用简洁的Harness技术驱动多个模型的编程Agent，所有的工具和能力都由配置和插件系统提供。本文档是工程代码所遵循的约定，修改代码前应相应地更改约定。

## 1. 设计原则

1. **以配置文件和插件为核心**。核心代码只会看到接口，实际的模型和工具有三种生效的方式：根据名称从注册表中解析，在配置文件中声明或由插件注入。请不要硬编码 `switch model`。

2. **单一静态二进制**。 `CGO_ENABLED=0`；交叉编译只需一行命令；CLI

3. **保守的依赖策略**。 默认使用标准库；而第三方依赖必须是轻量化的纯 Go 实现，且不应影响“单一二进制-跨平台-分发”原则。`TOML parsing` 已经被接受。

4. **两层扩展架构**。 第一层在编译期通过`init()`函数内置；第二层在运行时通过兼容 MCP 的 stdio JSON-RPC 子进程注入。

5. **接口和注册表优先**。 `Provider` 和 `Tool` 都应注册为接口

6. **快速迭代，不要一次性做过多改动**

语言： **对所有代码来说英语是主要语言** —— 评论，面向用户的字符串和工具描述都应采用英语。 `docs` 文档则为中英双语。

## 2. 项目架构

```
reasonix/
├── go.mod / go.sum          # module reasonix; require BurntSushi/toml
├── Makefile                 # build / cross / vet / fmt / test
├── README.md / README.zh-CN.md
├── reasonix.example.toml         # sample config
├── docs/SPEC.md             # this file
├── cmd/reasonix/main.go          # entry; blank-imports built-in providers/tools
├── cmd/reasonix-plugin-example/  # reference MCP stdio plugin (a runnable example)
└── internal/
    ├── cli/                 # subcommand routing, flags, assembly, exit codes
    ├── config/              # TOML loading (flag > project > user > defaults)
    ├── provider/            # Provider interface + types + kind→factory registry
    │   └── openai/          # OpenAI-compatible impl; init() registers "openai"
    ├── tool/                # Tool interface + Registry
    │   └── builtin/         # read_file/write_file/edit_file/move_file/bash/ls/glob/grep
    ├── permission/          # per-call Policy: allow/ask/deny rules → Decision
    ├── command/             # custom slash commands loaded from .reasonix/commands/*.md
    ├── plugin/              # stdio JSON-RPC (MCP) client; adapts remote tools
    └── agent/               # Session + harness loop
```

无环依赖链: `cli → {agent, plugin, config} → {tool, provider}`.
内置子模块（如 `provider/openai`, `tool/builtin`）会导入其父模块到自注册表；而父模块永远不会导入子模块。

## 3. 核心抽象

### 3.1 Provider + registry (`internal/provider`)

```go
type Provider interface {
    Name() string
    Stream(ctx context.Context, req Request) (<-chan Chunk, error)
}

// Factory builds a Provider from a resolved config instance.
type Factory func(cfg Config) (Provider, error)

// Register adds a factory under a kind (e.g. "openai"). Called from init().
func Register(kind string, f Factory)

// New instantiates the provider of the given kind.
func New(kind string, cfg Config) (Provider, error)

type Config struct {
    Name    string         // instance name, e.g. "deepseek"
    BaseURL string
    Model   string
    APIKey  string
    Extra   map[string]any // kind-specific options
}
```

- `openai` 是兼容OpenAI对话格式 `/chat/completions` 的实现。
- **兼容OpenAI的模型提供商本质是 `kind = "openai"` 的配置实例**，仅靠 `base_url` / `model` / `api_key_env`区分。增加新的OpenAI兼容模型只需要编辑配置文件而无需修改代码。
- **Provider是模型提供商端口的抽象**，由一对 `base_url` 和 `api_key_env` 确定，可以提供一个或多个模型。兼容OpenAI的对话一般向 `base_url + "/chat/completions"发生POST请求；只在使用需要完整请求地址的网关时配置 `chat_url`。一个 `Provider` 条目要么单独声明一个模型 `model = "..." ，要么以list列表声明多个模型 `model = ["...", "..."]` ，此时可附加一个可选参数 `default` 指定默认模型；通过这种列表声明，一个提供商无需反复声明端口和key就能暴露多个模型。**模型引用**，可通过 `default_model` ， `--model` 标志或桌面应用的模型切换器配置，并通过 `Config.ResolveModel`解析，接受 `Provider` 名称，不带提供商的纯模型名称和显式指定的 `provider/model`。`context_window` / `price` 则是按提供商确定的，因此要区分模型的上下文长度和定价，应声明单独的 `model` 条目。
- 流式工具调用输出的增量信息会按索引在提供商层级累积拼接，只有当工具调用结束后信息才会发送给模型。

### 3.2 Tool + registry (`internal/tool`)

```go
type Tool interface {
    Name() string
    Description() string
    Schema() json.RawMessage // JSON Schema for parameters
    Execute(ctx context.Context, args json.RawMessage) (string, error)
}
```

- 内置工具通过 `init()` 的 `tool.RegisterBuiltin(t)` 自注册入一个进程域的内置工具集，`tool.Builtins()` 会列出内置工具集的列表。
- 运行时注册表 `*Registry` 会在每次程序运行时组装，由被配置文件启用的内置工具**加上**插件提供的工具组成。

// FIXME: Translation of this paragraph remains awkward due to difficuties with 'schema'.
- Tool schemas are canonicalized on registry insertion. The built-in contract is
  documented in [`TOOL_CONTRACT.md`](TOOL_CONTRACT.md) and backed by tests that
  compare the documented surface against the same canonical schema path.
- 工具纲要在注册表插入时标准化，内置工具的约定记录在 [`TOOL_CONTRACT.md`](TOOL_CONTRACT.md) 并受相应的测试支持，该测试对比约定中记录的调用接口和对应的标准路径
- `Execute` 解析原始的 JSON 参数。工具会回传非致命的错误，Agent 将把错误反馈给模型以便模型自我纠错。

### 3.3 插件 (`internal/plugin`) — MCP 客户端

An external plugin is an MCP server declared in config. The wire protocol is
**JSON-RPC 2.0** in every case; only the transport differs. A `transport`
interface (`call` / `notify` / `close`) abstracts that, so the MCP-level logic
(handshake, `tools/list`, `tools/call`, …) is written once.
外部插件是指在配置文件中声明的 MCP 服务器。所有的实例的通信协议都应采用**JSON-RPC 2.0**，只是传输渠道 `transport` 不同。`transport`接口包含 `call` / `notify` / `close` 提供对通信端口的抽象，因此 MCP 层面的逻辑，如握手、 `tools/list` ， `tools/call` 等， 只会实现一次。

- **传输渠道** (配置文件 `type` 字段):
  - `stdio` （默认） — 本地子进程; one JSON message per line over the
    child's stdin/stdout (the MCP stdio convention). Declared with
    `command` / `args` / `env`; terminated on ctx cancel / shutdown.
    
  - `http` (a.k.a. `streamable-http`) — a remote server at `url`. Each request
    is an HTTP POST; the server replies with either `application/json` (one
    response) or `text/event-stream` (an SSE stream carrying the response plus
    any server notifications). The `Mcp-Session-Id` response header, once seen,
    is echoed on subsequent requests. Static `headers` (e.g. a bearer token) are
    sent on every request. OAuth is out of scope for now (see §9).
  - `sse` — the legacy 2024-11-05 HTTP+SSE transport; recognised but deferred
    (deprecated upstream — use `http`). Configuring it returns a clear error.
- `${VAR}` / `${VAR:-default}` are expanded in `command`, `args`, `env`, `url`,
  and `headers` so secrets come from the environment, not the config file.
- Lifecycle: `initialize` → `notifications/initialized` → `tools/list`;
  invocation via `tools/call {name, arguments}`.
- Each remote tool is adapted to the `Tool` interface and injected into the run
  registry, namespaced `mcp__<server>__<tool>` (spaces normalised to `_`) to
  match Claude Code and avoid clashes.
- A tool's MCP `annotations.readOnlyHint` maps to `Tool.ReadOnly()`. It defaults
  to false (a remote tool is opaque — we can't see its side effects), so a
  plugin opts a tool into parallel-batch dispatch and the permission layer's
  reader-default by declaring `readOnlyHint: true` in `tools/list`.
- `prompts/list` + `prompts/get` surface as `/mcp__<server>__<prompt>` slash
  commands; `resources/list` + `resources/read` are referenced as
  `@<server>:<uri>` in chat. `/mcp` shows connected servers and their counts.
- `cmd/reasonix-plugin-example` is a runnable reference stdio server (`echo`,
  `wordcount`), driven by an end-to-end test that builds the real binary.

### 3.4 Agent (`internal/agent`)

- `Session` holds `[]Message`.
- `Run(ctx, input)` loop: build `Request` (with tool schemas) → `provider.Stream`
  → print text deltas live, collect complete tool calls → if none, done; else
  execute each tool (built-in or plugin) and append results → repeat, bounded by
  `maxSteps`. `ctx` threads throughout (Ctrl-C aborts in-flight requests).
- A `Runner` is anything with `Run(ctx, input) error`; both `Agent` and
  `Coordinator` satisfy it, so the CLI is agnostic to single- vs two-model mode.

### 3.5 Two-model collaboration (`Coordinator`)

When `agent.planner_model` names a provider different from the executor, a
`Coordinator` runs two models in **separate sessions** to keep each one's prompt
prefix cache-stable:

- The **planner** (low-frequency) runs in its own session with the same standing
  memory context plus a filtered read-only research tool set, then produces a
  concise plan. It can inspect files/docs before planning, but writer and
  workflow tools are not exposed to it. `agent.planner_max_steps` bounds this
  read-only exploration independently from the executor's `agent.max_steps`.
- The plan is handed off as structured text to the **executor** — a full
  tool-using `Agent` in its own session — which carries it out.
- The sessions never mix, so neither model's prefix is disturbed by the other's
  turns; both grow prepend-only and stay cache-friendly. This reconciles
  "cache-first" with "two-model collaboration": switching models *inside one
  shared conversation* would break the prefix and tank cache hits, so we don't.

### 3.6 Context management (compaction)

Long tasks eventually fill the model's context window. Reasonix manages this with
**low-frequency compaction** that respects the cache-first design:

- Each provider declares its `context_window` (tokens). Context maintenance is
  tiered: below `agent.tool_result_snip_ratio` (default `0.6`) the session is
  left untouched apart from the soft notice; at the snip ratio, stale tool
  results before the recent tail are archived and shortened with deterministic
  head/tail markers; at `agent.compact_ratio` (default `0.8`) stale tool results
  are archived and pruned to short placeholders before any summary call; only if
  pruning still leaves the prompt above the threshold does summary compaction
  run. At `agent.compact_force_ratio` (default `0.9`), the existing forced fold
  may proceed even when the fold economics would normally skip it.
- Tool-result snip/prune never removes messages, so assistant `tool_calls` and
  tool results stay paired. `KeepErrors` preserves error/blocked tool outputs,
  and the recent tail is not rewritten. Snipped results can later be upgraded to
  pruned placeholders; already-pruned results are left alone.
- When summary compaction runs, it folds only the assistant/tool work. Every
  **user turn** small enough to be a brief and every **prior digest** is kept
  verbatim; the foldable remainder is summarized — using the executor's own
  provider, no tools — in place. The boundary is aligned backward off any tool
  result so the recent tail never begins with an orphan tool message whose
  `tool_calls` were summarized away.
- The dropped originals are archived under the user config dir
  (`reasonix/archive/<timestamp>.jsonl`; see §5 for its per-OS location), one
  message per line, so the full history stays traceable.
- The read-only `history` tool gives the agent on-demand BM25 retrieval over
  saved session JSONL files. `scope="project"` searches the current controller's
  session directory; `scope="global"` also searches the user-global session
  directory and compacted-history archives. `operation="around"` can then read a
  bounded transcript window around a returned hit. Search keeps the best hit and
  trims trailing common-word-only noise with a relative score floor; a 0-result
  response tells the agent how to retry with rarer terms or widen scope.
- The read-only `memory` tool gives the agent on-demand search/list/read access
  to saved auto-memory files. It complements the writer tools: `memory` checks
  what already exists, `remember` saves or updates a fact, and `forget` removes
  a stale one from the active index while archiving the file for traceability.
  Archived memory files are visible in local management surfaces (`/memory`,
  TUI, desktop panel) but are excluded from active-memory retrieval. Memory
  search uses the same relative BM25 floor and guides the agent to fall back to
  history when exact original wording or tool output matters.
- Agent-initiated `remember` and `forget` calls require a fresh human approval
  each time, even when tool auto-approval or YOLO/full-access mode is enabled.
  Guardian/safety review cannot answer these prompts on the user's behalf. In
  non-interactive headless runs or sub-agents, these tools are refused rather
  than auto-approved. The approval request includes a compact preview of the
  memory being saved or archived, while external notification hooks only receive
  the tool name.
  User-initiated memory edits in the local UI are already explicit user actions.
  See [`SESSION_MEMORY_RETRIEVAL.md`](SESSION_MEMORY_RETRIEVAL.md) for the
  detailed implementation contract.

**What survives a fold.** A fact the user states in a normal-sized turn is kept
verbatim and is never summarized away — at any point in the session, across any
number of compactions. A digest, once written, is likewise kept verbatim rather
than re-summarized, so facts it captured are not lost to drift. The one
**best-effort** boundary: a fact buried inside a single oversized message (a
large paste, over the per-turn pin budget) folds with the rest, so its survival
depends on the summarizer catching it while compressing bulk. There is no
reliable way to auto-detect an arbitrary fact in bulk, so durable facts belong in
their own turn rather than buried in a large paste; the raw oversized content is
still archived and recoverable either way.

This is the **only** point where the prompt prefix changes — a deliberate, rare
"cache-reset point". Between compactions the session grows prepend-only and
stays cache-friendly, so cache hit rate (the key observability signal) stays
high. `context_window = 0` disables compaction for an instance.

### 3.7 Permissions (`internal/permission`) — per-call gating

A coding agent runs shell commands and edits files autonomously. The permission
layer decides, **per tool call**, whether to allow it, deny it, or ask the user
first. It is independent of the model and of the CLI — the agent consults a
`Gate` interface at execute time; the gate is built from a static `Policy` plus
an optional interactive `Approver`.

```go
type Decision int            // permission package
const (Allow Decision = iota; Ask; Deny)

// Policy evaluates static rules against a tool call. Pure, no I/O.
type Policy struct { Mode Decision; Allow, Ask, Deny []Rule }
func (p Policy) Decide(toolName string, readOnly bool, args json.RawMessage) Decision
```

- **Rule syntax.** A rule is `Tool` (matches any call in that tool family) or
  `Tool(specifier)` (matches when the call's *subject* matches the specifier).
  Bash and file mutation approvals use Claude Code-style families such as
  `Bash(npm run build)`, `Bash(npm run test:*)`, and `Edit(docs/**)`. Built-in
  file mutations include writes, edits, notebook edits, symbol/range deletes,
  and `move_file` renames/moves. Legacy
  lowercase tool IDs and `tool=literal` rules still load for compatibility. The
  `:*` suffix marks a Bash command-prefix approval; generated prefix rules also
  reject later commands that introduce shell operators, so `Bash(go test:*)`
  does not cover `go test ./... && rm -rf tmp`.
  Legacy `Bash(go test *)` prefix rules still load, but new rules are saved as
  `Bash(go test:*)`. The subject is extracted generically from the call's JSON
  args by a small set of
  known keys — `command` (bash), `path` / `file_path` (file tools), `pattern`
  (grep/glob) — so tools need not change. A rule whose subject the args don't
  expose only matches in its bare `Tool` form.
- **Precedence.** `deny` > `ask` > `allow` > fallback. Fallback is `Allow` for
  read-only tools and `Mode` (default `Ask`) for writers. `deny` always wins, so
  a broad `allow = ["Bash"]` can still be carved by `deny = ["Bash(rm -rf*)"]`;
  conversely `ask` overrides a broad `allow` to force a prompt on a risky subset.
- **Resolving `Ask`.** The interactive front-end (the chat TUI) prompts the user
  — allow once / allow this approval scope for the session / always allow this
  approval scope / deny — via an `Approver`. For Bash, the default scope is the
  concrete command subject, and the user may choose a conservative command-prefix
  scope when available (for example `Bash(go test:*)`) so similar invocations in
  the same session or saved config do not prompt again. For file-mutation tools,
  a session grant covers editing for the rest of the session while a persisted
  grant is path-scoped when a path is available, stored as `Edit(<path>)` so all
  built-in file-mutating tools share it. A
  non-interactive run
  (`reasonix run`, a sub-agent, anything with no TTY / no approver) cannot prompt, so
  it resolves `Ask` to **allow** — preserving autonomous behaviour. A `Deny` is a
  hard block in *every* mode: the tool never executes and the model receives a
  "blocked" result it can adapt to (the same shape as a plan-mode refusal).
- **Relationship to plan mode.** Plan mode (§3.4) is an orthogonal, coarser gate
  checked before the permission layer. Its boundary is fail-closed for untrusted
  tools: while planning, a tool runs only if it reports a *trustworthy*
  `ReadOnly()==true` — a built-in, a first-party MCP `ReadOnlyToolNames`
  override, a plugin-level `trusted_read_only_tools` declaration, or a concrete
  MCP name listed in `[agent].plan_mode_allowed_tools` — or self-reports
  plan-safe via `tool.PlanModeClassifier`. An MCP tool's `ReadOnly()` may
  instead come from the server's self-reported `readOnlyHint`, which plan mode
  treats as untrusted (`tool.PlanModeUntrustedReadOnly`): interactive
  controllers may ask once before executing it and may remember a persistent
  approval as `trusted_read_only_tools`. This trust prompt is a fresh user
  decision: `auto`, `yolo`, and the approved-plan execution window do not answer
  it, but an explicit session grant still prevents repeat prompts for the same
  tool. Non-interactive sessions and declined approvals remain fail-closed.
  Bash is gated separately: built-in read-only commands and concrete prefixes
  declared in `[agent].plan_mode_read_only_commands` may run. Interactive
  controllers may also ask once before running an unknown query-shaped prefix
  and may remember a persistent approval as the same
  `plan_mode_read_only_commands` entry. This bash trust prompt is also a fresh
  user decision: `auto`, `yolo`, and the approved-plan execution window do not
  answer it, while explicit session/persistent trust prevents repeat prompts for
  that prefix. Shell operators, background execution, shell interpreters, and
  unsafe arguments stay blocked while planning. Writers, installers, memory
  mutation, process control, and `complete_step` (read-only yet post-approval only, so it
  self-reports plan-unsafe) are refused; the enforced invariant is
  PlanSafe ⇒ ReadOnly. An untrusted read-only MCP/plugin tool is therefore
  blocked until the user approves or pre-trusts it, and it is excluded from
  planner/read-only research sub-agents until the tool is part of the trusted
  read-only registry. Plan mode still allows `read_only_task` and
  `read_only_skill`, whose sub-agents receive only read-only research tools and
  safe foreground bash; writer-capable `task` delegation and full skill execution
  remain blocked. The desktop MCP panel writes the same
  `trusted_read_only_tools` raw-name list as an advanced management surface:
  **Pre-trust read-only** adds currently listed `readOnlyHint` tools, per-tool
  **Pre-trust** adds an audited reader manually, and **Untrust** removes it
  again. These UI actions do not make MCP `readOnlyHint` globally trusted by
  default.
- **User decisions are separate from tool approvals.** Runtime tool approval has
  three user-facing postures: `ask` ("需要批准"), `auto` ("自动批准"), and
  `yolo` ("Yolo批准"). `auto` lets the permission policy auto-approve the writer
  fallback while preserving explicit ask/deny rules; `yolo` skips all tool
  permission approvals for approval-gated tools such as writers and Bash.
  Neither posture answers `ask` questions, approves `exit_plan_mode` plans, or
  confirms MCP read-only trust prompts for the user.
  Auto-plan is also a separate feature flag: when enabled, a complex task may
  still enter plan mode in any tool approval posture. After a user approves a
  plan, the controller opens a short `approvedPlanAutoApproveTools` execution
  window so the model can perform the approved writes without re-prompting; that
  transient window still does not auto-approve future plans. In headless `ask`
  execution, any fallback answer is labelled as a model assumption, not as a
  user decision.

- **Collaboration mode is separate from tool approval.** The desktop composer
  presents collaboration as `normal` ("正常模式"), `plan` ("计划模式"), and
  `goal` ("目标模式"). `/goal <objective>` starts an autonomous, session-scoped
  active goal: the controller prepends goal context to user turns outside the
  cache-stable system prompt and keeps issuing continuation turns until the
  model reports completion, repeats the same blocked state three times, the user
  stops it, or the safety continuation limit is reached. Blocked-state matching
  is normalized for casing, whitespace, and punctuation so minor wording drift
  does not reset the audit; restarting a goal begins a fresh blocked audit. A
  goal is treated as a task contract: if the objective includes Context,
  Request, Output format, Constraints, or Pause policy sections, those sections
  define the autonomous work boundary. When they are absent, the model infers a
  lightweight contract from the conversation and workspace. The injected goal
  block tells the model to pause only for irreversible or externally visible
  operations, scope changes, or information only the user can provide; ordinary
  uncertainty should be handled with sensible defaults and reported as an
  assumption. Completion requires the concrete request, output format,
  constraints, and relevant verification expectations to be satisfied or
  explicitly reported as unverified.
  Goals that look like long-horizon research, debugging, optimization, or
  implementation work automatically add an AutoResearch protocol to the same
  transient active-goal user block. AutoResearch is a Goal strategy, not a
  standalone global skill: it writes project-local state under
  `.reasonix/autoresearch/YYYYMMDD-HHMMSS-slug/` and keeps dynamic run state out
  of `REASONIX.md`, `AGENTS.md`, project memory, tool schemas, and the
  cache-stable system prompt. `/goal --research <objective>` forces that
  strategy; `/goal --simple <objective>` forces lightweight Goal. Outside goal
  mode, an ordinary prompt with a very strong AutoResearch signal is upgraded by
  the host into the equivalent of `/goal --research <original prompt>`; the
  ordinary-prompt classifier is intentionally stricter than `/goal`'s internal
  classification so weak words such as "long term", "optimize", "research", or
  "verify" do not create durable task state by themselves. `/goal clear` removes
  the active goal. Switching into plan/normal mode clears the active goal in the
  desktop UI so the collaboration mode remains one of the three choices, while
  the underlying tool approval posture is preserved.

| Tool approval posture | Tool approvals | Plan approval | MCP read-only trust | `ask` questions |
| --- | --- | --- | --- | --- |
| Need approval / `ask` | Follow permission policy (`Ask` prompts interactively) | Waits for user | Waits for user unless session-granted | Waits for user |
| Auto approve / `auto` | Writer fallback auto-allowed; explicit ask/deny rules still apply | Waits for user | Waits for user unless session-granted | Waits for user |
| YOLO approval / `yolo` | Approval prompts auto-allowed unless denied | Waits for user | Waits for user unless session-granted | Waits for user |
| Approved-plan execution window | Approved plan's tool calls auto-allowed unless denied | Future plans still wait | Waits for user unless session-granted | Waits for user |

Out of the box (`mode = "ask"`, no rules) `reasonix run` behaves exactly as before
(writers resolve `Ask`→allow with no TTY), while `reasonix` now prompts before
each writer/bash call. `deny` rules harden both modes.

### 3.8 Slash commands (`internal/command`)

The chat TUI accepts `/command` input. Three kinds share one dispatch:

- **Built-in actions** (`/compact`, `/new`, `/clear`, `/effort`, `/mcp`, `/help`) manipulate session
  state locally and never reach the model. `/new` starts a new session while
  saving the previous transcript for resume/history. `/clear` requires
  confirmation, then discards the current context without saving it; it does not
  delete project memory.
- **Custom commands** are Markdown files under `.reasonix/commands/` (project) and
  the user config dir, e.g. `~/.reasonix/commands/` on macOS/Linux; the project dir overrides the user dir on a
  name clash. A file `review.md` becomes `/review`; a subdirectory namespaces it
  (`git/commit.md` → `/git:commit`). Invoking one renders its body and sends the
  result as the next user turn.
- **MCP prompts** (§3.3) appear as `/mcp__<server>__<prompt>`.

```markdown
---
description: Review the staged diff
argument-hint: [focus-area]
---
Review the staged diff. Focus on $ARGUMENTS, list bugs with file:line.
```

- Frontmatter is an optional `---`-fenced block of simple `key: value` lines;
  `description` and `argument-hint` are recognised (no YAML dependency — Reasonix
  stays lean). The remainder is the body template.
- Substitution in the body: `$ARGUMENTS` (all args, space-joined), `$1`…`$N`
  (positional, empty when absent), `$$` (a literal `$`). Arguments are the
  space-separated tokens after the command.
- Loading is pure (`command.Load(dirs...)`) and tested; a malformed file is
  skipped, not fatal. Custom and MCP-prompt commands both resolve to text and
  reuse the same "start a turn" path as a typed message.

#### CLI modal/composer ownership

The Bubble Tea chat TUI has one bottom composer. A slash-command overlay must
declare whether it owns keyboard input:

- **Modal overlays** own navigation/confirm/cancel keys and must hide the
  composer while open. Examples: `/mcp`, `/resume`, `/rewind`, approval prompts,
  and non-typing `ask` choice cards.
- **Input-owned overlays** are attached to the textarea and must keep the
  composer visible. Examples: slash/@ autocomplete and `ask` free-text mode.

New CLI overlays must update `chat_tui.hideComposer()` and add/extend layout
tests so `bottomRows()` accounts for either `panel + status` or
`panel + composer + status`. This prevents inactive chat input boxes from being
rendered under modal panels.

### 3.9 Chat references (`@`)

A chat message may embed `@` references; before the turn is sent, each is
resolved and prepended to the message as a tagged block the model can read.

- `@<server>:<uri>` where `<server>` is a connected MCP server → an MCP
  resource (`resources/read`), wrapped `<resource ref="…">…</resource>`.
- `@<path>` otherwise → a **local file or directory**, but only when the path
  actually exists on disk. This existence gate is the disambiguator: an ordinary
  `@mention` or an email address resolves to no file and stays literal text. A
  file is wrapped `<file path="…">…</file>` (size-capped, binary files noted not
  dumped); a directory becomes a recursive listing (depth-first, skipping common
  noise like `.git` and `node_modules`).
- Resolution is asynchronous (off the TUI event loop); a fetch failure surfaces
  as a notice but doesn't block the turn. Reads are user-initiated and read-only
  — they do **not** pass the permission gate (§3.7).
- Typing `/` or `@` opens an autocomplete menu above the input. The `@` menu
  navigates **one directory level at a time** (`os.ReadDir`, never a recursive
  walk — bounded for huge directories): a directory entry descends, a file
  completes, and MCP resources appear alongside top-level entries. The
  bottom-region menu changes height only on these discrete actions, never per
  streamed token, so scrollback stays clean (§ rendering).

### 3.10 Subagent profiles and explicit CLI execution

A subagent profile is a Skill with `runAs: subagent` and, for profiles managed
by the desktop or CLI editors, `invocation: manual`. Profiles reuse the existing
project/global Skill files; they do not introduce another state format or
database. Manual invocation excludes a profile from the pinned Skill index so
the model cannot discover it implicitly, while explicit `/<name> <task>`
invocation remains available.

Interactive slash invocation and `Controller.RunSubagentProfile` both execute
the profile with the Boot-wired Skill runners. Each run gets an isolated child
session and returns only its final answer to the caller. The headless contract is
explicit:

- `reasonix subagent try <name> ... <task>` uses the read-only Skill runner;
- `reasonix subagent run <name> ... <task>` uses the normal permission and
  sandbox path; and
- ordinary `Controller.Run` / `reasonix run` remains unchanged and does not
  reinterpret slash-prefixed input as a subagent command.

Desktop and CLI profile mutations share
`skill.ValidateEditableSubagentProfile`. Only simple manual project/global
profiles can be rewritten or deleted. Custom-scope Skills, unmanaged
frontmatter, and Skill directories containing `references/` or `scripts/` are
refused so an editor cannot silently flatten or discard rich Skill content.
Built-in profiles support configuration overrides but have no writable file.

Effective model and effort precedence is: per-profile
`agent.subagent_models` / `agent.subagent_efforts`, profile frontmatter,
`agent.subagent_model` / `agent.subagent_effort`, then executor/default model
configuration. See [Subagent profiles](./SUBAGENT_PROFILES.md) for the user-facing
command and file-format contract.

## 4. Data Types (`internal/provider`)

```go
type Role string
const (RoleSystem Role = "system"; RoleUser Role = "user"
       RoleAssistant Role = "assistant"; RoleTool Role = "tool")

type Message struct {
    Role       Role       `json:"role"`
    Content    string     `json:"content,omitempty"`
    ToolCalls  []ToolCall `json:"tool_calls,omitempty"`
    ToolCallID string     `json:"tool_call_id,omitempty"`
    Name       string     `json:"name,omitempty"`
}

type ToolCall   struct { ID, Name, Arguments string }              // Arguments: raw JSON
type ToolSchema struct { Name, Description string; Parameters json.RawMessage }
type Request    struct { Messages []Message; Tools []ToolSchema; Temperature float64; MaxTokens int }

type ChunkType int
const (ChunkText ChunkType = iota; ChunkToolCall; ChunkDone; ChunkError)

type Chunk struct {
    Type     ChunkType
    Text     string    // ChunkText
    ToolCall *ToolCall // ChunkToolCall
    Err      error     // ChunkError
}
```

## 5. Configuration (TOML)

Resolution order: **flag > project `./reasonix.toml` > the user config file
> built-in defaults**. Starting with **Reasonix v1.8.1**, the user config lives
at `~/.reasonix/config.toml` on macOS/Linux and
`%AppData%\reasonix\config.toml` on Windows. See
[Configuration paths](./CONFIG_PATHS.md) for migration and related data paths.
Fields marked user/global only, including agent step limits, are not overridden
by project `reasonix.toml`.
Provider entries name secrets with `api_key_env`; saved key values live in
Reasonix's global `<Reasonix home>/.env`, shared by CLI and desktop. Project
`.env`, home `.env`, inherited shell environment variables, legacy credentials,
and the OS keyring are not provider-key runtime fallbacks. Project `.env` still
feeds workspace-scoped, non-provider `${VAR}` expansion for MCP/plugin settings
without importing provider keys or Reasonix control variables. Step-limit
preferences belong in the user config.
Project `reasonix.toml` does not override `agent.max_steps` or
`agent.planner_max_steps`, and it does not override the user-level Memory v5
compiler switch.

```toml
default_model = "deepseek"   # provider name (→ its default model) or "provider/model"
# language    = "zh"                # ui language tag; empty = auto-detect from $LANG / $REASONIX_LANG

[ui]
# shortcut_layout = "desktop"       # classic|desktop; compatibility setting
# cursor_shape = "underline"        # CLI/TUI textarea cursor: underline|block|bar

[agent]
system_prompt = "You are Reasonix, a coding agent..."  # or system_prompt_file = "..."
max_steps         = 0    # user/global only; executor tool-call rounds; 0 = no limit
planner_max_steps = 0    # user/global only; planner read-only tool-call rounds; 0 = no limit
temperature       = 0.0
memory_compiler = { enabled = true, verbosity = "observe" }   # user/global only; observe|compact; CLI: reasonix config memory-v5 off|observe|compact|on|status
reasoning_language = "auto"       # visible reasoning text: auto|zh|en
# plan_mode_allowed_tools = ["custom_reader"]   # extra read-only declarations for custom tools;
#                                                # cannot unlock known blocked tools or unsafe bash
# plan_mode_read_only_commands = ["gh issue view", "gh pr diff"]   # extra read-only shell prefixes for plan mode
# planner_model = "deepseek-pro"   # optional: two-model collaboration (low-frequency planner)
# subagent_model = "deepseek-pro"   # optional default for runAs=subagent skills
# subagent_effort = "high"           # optional default reasoning effort for subagents
# subagent_models = { review = "deepseek-pro", security_review = "deepseek-pro" }
# subagent_efforts = { review = "max", security_review = "high" }

# A vendor endpoint exposing several models under one base_url/key.
[[providers]]
name           = "deepseek"
kind           = "openai"
base_url       = "https://api.deepseek.com"
# chat_url     = "https://proxy.example.com/v1/chat/completions"   # optional full chat request URL
# models_url   = "https://proxy.example.com/v1/models"             # optional model discovery URL
models         = ["deepseek-v4-flash", "deepseek-v4-pro"]
default        = "deepseek-v4-flash"   # optional; defaults to models[0]
api_key_env    = "DEEPSEEK_API_KEY"
context_window = 1000000   # tokens; harness compacts older history near this limit (0 disables)

# A single-model entry still works for custom OpenAI-compatible endpoints.

[environment]
enabled = true   # inject a stable startup summary of OS, shell, and common tool versions

# Optional trusted executable paths shown to the model when PATH probing is not enough.
# Workspace-local paths are listed but not auto-executed during startup probing.
# [environment.tools]
# go = "/opt/homebrew/bin/go"

[tools]
enabled = []   # omit/empty = all built-ins
bash_timeout_seconds = 120   # foreground safety cap; set 0 for no tool-local cap
mcp_call_timeout_seconds = 300   # default MCP call safety cap; plugin/tool overrides may raise it

[tools.shell]
prefer = "auto"   # auto (default) | bash | powershell | pwsh — force the shell tool's interpreter
# path = "C:\\Program Files\\PowerShell\\7\\pwsh.exe"   # explicit executable for the chosen shell

[skills]
# paths = ["~/my-skills", "../shared/skills"]   # extra custom skill roots
# excluded_paths = ["~/.agents/skills"]         # hide convention roots without deleting folders
# disabled_skills = ["review"]                  # hidden from prompt, slash invocation, and skill tools

[permissions]
mode  = "ask"                              # writer fallback when no rule matches: ask|allow|deny
deny  = ["Bash(rm -rf*)", "Bash(git push*)"]   # hard-blocked in every mode
allow = ["Bash(go test:*)", "Bash(git status:*)"]  # never prompted
ask   = []                                 # force a prompt even if otherwise allowed

[sandbox]
# workspace_root = ""          # file-writers confined here; empty = cwd
# allow_write    = ["/tmp"]    # extra dirs write_file/edit_file/multi_edit/move_file may modify
# forbid_read    = ["${HOME}/.ssh"]   # dirs read/list/search tools and sandboxed bash may not inspect

[serve]
auth_mode = "none"             # none|token|password; use auth before binding beyond localhost
# token = ""                   # optional fixed token; empty token mode generates one at startup
# password_hash = ""           # bcrypt hash generated with reasonix serve --hash-password --password '...'
# behind_proxy = false         # trust X-Forwarded-* only behind a trusted reverse proxy

[[plugins]]
name    = "example"            # type defaults to "stdio"
command = "reasonix-plugin-example"
args    = []
# env   = { FOO = "bar" }
# call_timeout_seconds = 600            # per-server MCP call timeout; 0 = global/default cap
# tool_timeout_seconds = { "generate_video" = 1800 }   # raw MCP tool names
# trusted_read_only_tools = ["search"]   # optional pre-seeded MCP read-only trust

# [[plugins]]                   # a remote MCP server over Streamable HTTP
# name    = "stripe"
# type    = "http"             # "stdio" (default) | "http" | "sse"
# url     = "https://mcp.stripe.com"
# headers = { Authorization = "Bearer ${STRIPE_KEY}" }   # ${VAR} / ${VAR:-default} expanded
```

`reasonix setup` writes this default config so the CLI is usable out of the box.

`[ui].cursor_shape` is normalized to `underline`, `block`, or `bar`; empty or
unknown values fall back to `underline`. It applies to the Bubble Tea CLI/TUI
textarea only, while desktop and browser inputs keep their platform-native
cursor behavior.

`[serve]` controls the HTTP browser frontend used by `reasonix serve`. The
default `auth_mode = "none"` is intended for the loopback default
`127.0.0.1:8787`; deployments reachable from another machine must use `token` or
`password`. Password mode requires either a startup `--password` or a stored
bcrypt `password_hash`. `behind_proxy` must stay false unless the server is
behind a trusted proxy that owns the `X-Forwarded-For` and `X-Forwarded-Proto`
headers.

MCP servers may also be declared in a project-root `.mcp.json` using Claude
Code's exact `mcpServers` schema (`command`/`args`/`env`, `type`/`url`/`headers`,
`${VAR}` expansion). It is read after the TOML files and merged into
`[[plugins]]`; on a name collision `reasonix.toml` wins (it is the more explicit,
Reasonix-specific source). This lets a server already configured for Claude work in
Reasonix unchanged.

```json
{ "mcpServers": {
  "stripe": { "type": "http", "url": "https://mcp.stripe.com",
              "headers": { "Authorization": "Bearer ${STRIPE_KEY}" } }
} }
```

`[sandbox]` is the *enforcement* layer beneath permissions (which are *policy*).
Phase 0 confines the file-writing built-ins (`write_file`, `edit_file`,
`multi_edit`, `move_file`) to `workspace_root` (default cwd), the Reasonix user
config dir, plus `allow_write`: a write whose target — resolved to an absolute,
symlink-free path so a symlinked dir or `..` cannot tunnel out — falls outside
every root is refused, and the error is fed back to the model. Confinement is on
by default (root = cwd), so edits stay in the project while the agent can still
update its own global config. `forbid_read` lists directories the agent should
not read, list, or search; entries support `${VAR}` / `${VAR:-default}` expansion
and should be absolute, or use `${HOME}` for home-relative secrets such as
`${HOME}/.ssh`. `bash` is itself jailed by default when an OS sandbox is
available (`[sandbox] bash = "enforce"`: Seatbelt on macOS, bubblewrap on
Linux, and a native helper on Windows): each command is allowed to write only
the same roots plus platform-specific command temp/cache roots, denied reads
under `forbid_read`, and allowed to reach the network only when
`network = true`.
**Current status:** stable builds force the effective Bash sandbox mode to
`off` on Windows — even an explicit `bash = "enforce"` resolves to `off` (and
`reasonix doctor` flags the ignored setting) — because the native backend
described below still breaks common Git Bash/MSYS2, Docker, and git workflows.
The description is kept as the design of record until the backend is reliable
enough to re-enable.
The native Windows helper uses Reasonix's bundled Windows sandbox backend:
AppContainer for read-only commands and a low-integrity token for writable
commands, with temporary ACL grants for
writable roots and tool executables, a per-command temp root instead of mutating
the global Temp directory, temporary deny ACEs for `forbid_read` (files and
directories), best-effort restoration from pre-run DACL snapshots for touched
directories, and a kill-on-close Job Object. Because the sandbox works by
temporarily mutating shared-path ACLs and integrity labels, concurrent commands
against the same root are serialized with a per-root lock, and residue from a
force-killed command (a lingering low-integrity label or `forbid_read` deny ACE)
is swept by the next run so a crash cannot durably lower a workspace's integrity
or lock the user out of a `forbid_read` path. A writable command runs under a
low-integrity token, so beyond the configured roots it retains write access to
the narrow set of locations Windows leaves writable to any low-integrity process
(e.g. `AppData\LocalLow`); the workspace boundary and `forbid_read` denials are
unaffected. Read-only AppContainer commands omit network capabilities when
networking is disabled; writable Windows commands fail closed when
`network = false` because the low-integrity token does not provide a reliable
per-process network block without elevated firewall/WFP setup.
When no OS sandbox is available, `bash = "enforce"` refuses bash execution
instead of running unconfined. Install the platform sandbox backend
(bubblewrap/`bwrap` on Linux, `sandbox-exec` on macOS) or set
`[sandbox] bash = "off"` to explicitly restore the pre-1.16 unconfined shell
behavior. The escape-prompt and broader OS support are Phase 1's remainder (§9).

## 6. Error Handling

- Library code wraps with `fmt.Errorf("...: %w", err)` and returns; it never
  prints or calls `os.Exit`.
- Only `cli` / `main` decide exit codes and user-facing messages.
- Tool execution errors are fed back to the model, not fatal.
- Network layer should apply bounded exponential backoff on 429 / 5xx
  (interface reserved; implementation may follow).

## 7. Code Style

- `gofmt` + `go vet` must be clean; package names lowercase; exported
  identifiers documented; comments explain *why*, not *what*.
- No premature generalization. Prefer clear and direct.

## 8. Distribution

- Build: `CGO_ENABLED=0 go build -ldflags "-s -w -X main.version=$(VERSION)" -o reasonix ./cmd/reasonix`
- Cross matrix: `darwin|linux|windows` × `amd64|arm64`.
- Version injected via ldflags (`git describe --tags --always`).
- Install: prebuilt binary / `go install` / future `brew tap`.

## 9. Roadmap (not in current scope)

- Sandbox Phase 1: an OS-level jail for `bash` so commands — not just the
  file-writer built-ins (Phase 0) — are confined to the workspace. **Seatbelt on
  macOS, bubblewrap on Linux, and a native Windows helper ship, on by default
  when available** (see §5).
  Remaining: (a)
  the escape-prompt — detect sandbox-unavailable or sandbox-denied failures and
  offer an explicit, permission-gated unconfined rerun (in `reasonix run`, the
  command just fails and the model adapts), which completes the "allow inside the
  box, prompt at its edge" model; (b) an optional elevated Windows backend with a
  dedicated sandbox user for enterprise hardening. Shells out to OS tooling so
  the binary stays dependency-free. With this in
  place, "always allow" rule persistence becomes optional rather than load-bearing.
- MCP long tail (deferred deliberately — no consumer / no foundation yet): OAuth
  2.0 + `headersHelper` auth for remote servers; the remaining `.mcp.json` scopes
  (local / user — project scope shipped, see §5); tool-search deferral;
  `list_changed` live updates; channels / elicitation / roots; plugins that
  provide *providers*, not just tools.
- An Anthropic-native provider `kind` (native prompt-cache control), proving the
  registry generalises beyond one wire format.
- "Always allow" persistence writing learned rules back to project config; a
  per-session permission override flag for `reasonix run`.
