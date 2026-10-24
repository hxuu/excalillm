# Agent-agnostic transcript → JSON contract

Research for `excalillm` planning step ("transcript in, scenes JSON out") using the user's own coding agent.

**Method.** Local inventory with `--version` / `--help`, auth probes, and error-path runs on 2026-10-24 (Linux, non-TTY). Official 2026 docs for CLIs that are not installed. Claim tags:

- **[V]** verified locally (command output, exit code, or observed behaviour)
- **[D]** official vendor documentation
- **[I]** inferred from observed behaviour or third-party report; not confirmed on this machine

**Environment limits.** No agent on this machine can currently answer a prompt. Claude is authenticated but upstream returns `Model is unavailable`; Gemini has no auth; OpenCode has a stored credential but its provider connection is refused. Part 2 therefore produced no successful JSON generations; what follows is contract design grounded in local failure semantics plus docs. Details in §2.

---

## 1. Local inventory

| CLI | Installed | Version | Auth state | Verdict |
|---|---|---|---|---|
| `claude` | yes `/home/hxuu/.local/bin/claude` | 2.1.296 | `loggedIn: true`, `authMethod: api_key`, `apiKeySource: ANTHROPIC_API_KEY` [V] | usable interface, currently broken upstream |
| `codex` | no | — | — | docs only |
| `opencode` | yes `/usr/bin/opencode` | 2.0.26 | OpenCode Console credential stored [V]; provider transport refused [V] | interface usable with caveats |
| `aider` | no | — | — | docs only |
| `gemini` | yes `/home/hxuu/.npm-global/bin/gemini` | 0.47.0 | no auth method configured [V], exit 41 | interface usable, needs auth |
| `cursor-agent` | no | — | — | docs only |
| `ollama` | no | — | — | docs only; HTTP API preferred |

### 1.1 Claude Code 2.1.296

**Non-interactive:** `-p, --print` [V]. Help text: "starts an interactive session by default, use `-p/--print` for non-interactive output".

**Prompt delivery:** positional `[prompt]` plus stdin. Docs: "Non-interactive mode reads stdin", with a 10 MB cap; piped input is combined with the argv prompt [D]. Local pipe was accepted and the run proceeded to the API call [V].

**Output:**
- default `text`: final answer only [D]
- `--output-format json`: single JSON envelope with `result`, `session_id`, `is_error`, `subtype`, `total_cost_usd`, `usage` [V observed on error; D on success fields]
- `--output-format stream-json`: NDJSON events, terminal `result` [D]
- `--json-schema <schema>`: schema-constrained output in `structured_output` field; field is **absent** when generation fails or is declined [D]. Schema is passed inline, not as a file path [D].

**Files/permissions:** tools are available. `--allowedTools`, `--disallowedTools` [V help], `--permission-mode {acceptEdits,auto,bypassPermissions,manual,dontAsk,plan}` [V help], `--permission-prompts none` for unattended runs (docs), `--dangerously-skip-permissions` (do not use). `--no-session-persistence` [V help] avoids writing transcripts to `~/.claude/projects` — relevant because the input is a private transcript.

**Context pollution:** without `--bare`, `-p` loads `CLAUDE.md`, hooks, MCP servers, skills, and plugins from cwd and `~/.claude`; docs explicitly warn a `-p` session connects project MCP servers even in an untrusted folder. `--bare` disables auto-discovery but **also ignores OAuth/keychain** and requires `ANTHROPIC_API_KEY` [D]. This is a real config fork: subscription users cannot use `--bare`.

**Exit codes:** docs say 0 on success, non-zero on failure, and that in-run failures are printed as the result on stdout [D]. Locally: API error with `--output-format json` → **exit 1**, stdout JSON with `"is_error":true`, `"subtype":"success"` (contradictory fields), `result:"API Error: 400 ... Model is unavailable."` [V]. Plain `-p` without `--output-format json` hung past 120 s on the same failure [V] — so always set a timeout and always use JSON.

**Sandbox/prompt blocking:** `-p` has no trust dialog and no per-MCP approval prompt [D]. In Manual mode (the default when nothing sets a mode in newer versions), tool calls needing approval are **denied**, not prompted [D].

### 1.2 OpenCode 2.0.26

**Non-interactive:** `opencode run [message...]` [V help].

**Prompt delivery:** message on **argv only**. Piping stdin with no message produced `{"type":"error", ... "You must provide a message"}` [V]. `--file/-f` attaches a file to the message [V help]. For a 1,500-word Arabic prompt, put the instruction on argv and the transcript in an attached file, or pass the whole prompt on argv (size is fine; quoting is the risk).

**Output:**
- `--format default`: human/TUI-style output [I]
- `--format json`: **NDJSON event stream**, not one JSON document. Observed `step_start` events, then a terminal error event [V]. There is no documented "final JSON" mode, so the consumer must collect assistant text parts and ignore lifecycle events [I].

**Files/permissions:** `--auto` auto-approves non-denied permissions [V help]. No documented read-only switch. Agent can edit files in cwd [I].

**Exit codes:** **always 0 in every failure observed**: missing message → 0 [V], provider transport error → 0 [V]. Do not use exit status as the success signal.

**Latency:** transport failure took ~80 s of retries before the error event [V].

### 1.3 Gemini CLI 0.47.0

**Non-interactive:** `-p, --prompt` [V]. Help: "Run in non-interactive (headless) mode with the given prompt. Appended to input on stdin (if any)." So argv prompt + stdin transcript is the intended pipe shape [V help].

**Output:** `-o, --output-format {text,json,stream-json}` [V help]. JSON is a single object with `response` and `stats`, optional `error` [D]. `stream-json` is NDJSON with `init/message/tool_use/tool_result/error/result` [D].

**Files/permissions:** `--approval-mode {default,auto_edit,yolo,plan}` [V help]; `plan` is read-only [V help]. `--skip-trust` trusts the workspace for the session [V help] — needed to avoid a trust gate in headless runs. `--sandbox` exists [V help]. `--allowed-tools` is deprecated in favour of policy files [V help].

**Exit codes:** docs: `0` success, `1` general/API failure, `42` input error, `53` turn limit [D]. Locally missing auth → **exit 41**, which is **not in the documented table** [V]. Another reminder that the exit-code contract is version-specific.

**Auth:** requires `GEMINI_API_KEY`, Vertex, or Code Assist login; error text names `/home/hxuu/.gemini/settings.json` [V]. No `auth status` subcommand in help [V].

**Output purity:** JSON envelope wraps the answer, so a `.response` adapter is mandatory; `text` mode is clean but unvalidated [D/I].

### 1.4 Not installed

`codex`, `aider`, `cursor-agent`, `ollama` are absent from PATH [V]. Their facts in §3 come from vendor docs; no local binaries to probe.

---

## 2. Empirical runs (Part 2)

No CLI produced a successful generation, so the planned "tiny JSON prompt" and "1,500-word Arabic prompt" experiments were **skipped for lack of a working model**, not for quota reasons. What was measured is the failure path — which is the part of the contract most likely to bite.

| Attempt | Result | Exit | Latency | Parseable? |
|---|---|---|---|---|
| `claude -p --output-format json 'say OK'` | API 400 `Model is unavailable`; JSON envelope with `is_error:true` | 1 | 8.76 s (reported `duration_ms`) | envelope yes; no answer |
| `claude -p 'say OK'` (text) | no output; hung | killed at 120 s | >120 s | no |
| `claude -p` with stdin pipe | hung | killed | >120 s | no |
| `gemini -p ... -o json` | JSON `{session_id,error:{...auth...}}` | **41** | <1 s | envelope yes; no answer |
| `gemini` stdin+`-p` | same auth error | 41 | <1 s | yes |
| `opencode run '...' --format json` | NDJSON `step_start` ×10 then `provider.transport ConnectionRefused` | **0** | ~80 s | event stream yes; no answer |
| `opencode run --format json` (stdin only) | `{"type":"error","message":"You must provide a message"}` | **0** | <1 s | yes |

**Conclusions that matter for the contract**

1. **Exit code is not a success signal** for at least three of the seven CLIs (`opencode` always 0; Claude can emit `is_error` with exit 1 but a contradictory `subtype:"success"`; Codex is reported to exit 0 on task failure [I]).
2. **stdout framing differs three ways**: bare text, one JSON envelope, NDJSON event stream. A single parser cannot handle all three.
3. **Failure text can arrive on stdout** (Claude's `result`, Gemini's `error`), so `stderr` is not the error channel.
4. **Timeouts must be generous and enforced externally.** OpenCode burned 80 s retrying; Claude burned >120 s. A 30 s default would produce false failures.
5. **Auth checks can lie.** OpenCode reports a stored credential yet the provider refuses connections. Only a real probe (or a real job) proves usability.

> Consequence: the "prompt on stdin, JSON on stdout" pair is a good *default* but not a *contract*. The contract needs a per-agent output adapter and a validation gate. See §4.

---

## 3. Docs check for the missing CLIs

### 3.1 OpenAI Codex — `codex exec`

Sources: <https://learn.chatgpt.com/docs/non-interactive-mode>.

| Field | Fact | Tag |
|---|---|---|
| One-shot | `codex exec "<prompt>"` | D |
| Prompt input | argv; stdin as extra context when argv prompt present; `codex exec -` or no prompt = stdin is the prompt | D |
| stdout | **only final agent message**; progress on **stderr** | D |
| JSON | `--json` = JSONL events (`thread.started`, `turn.*`, `item.*`, `error`); final text is an `item.completed` agent message, not a top-level field | D |
| Schema | `--output-schema schema.json` + `-o/--output-last-message out.json`; writes final message to file and still prints to stdout | D |
| Structured-output caveat | Schema is reportedly ignored when tools/MCP servers are active (open issue #15451); malformed output observed | I |
| Sandbox | default **read-only**; `--sandbox workspace-write` / `danger-full-access`; `--full-auto` removed | D |
| Repo requirement | must run in a git repo unless `--skip-git-repo-check` | D |
| Context control | `--ephemeral` (no session files), `--ignore-user-config`, `--ignore-rules` | D |
| Exit codes | 0 on protocol success; reported 0 even when tool calls failed / requested artifacts missing (#15536, #37717) | I |
| Auth | reuses saved login; `CODEX_API_KEY` for automation | D |
| Prompt blocking | exec mode is non-interactive; no TUI prompts. MCP `required=true` init failure aborts the run | D |

**Fit:** good, if you consume the final message from `-o` or stdout, not `--json`. Schema support is the strongest of the non-Anthropic CLIs, but the tools-active bug means the schema cannot be the only gate.

### 3.2 Cursor CLI — `cursor-agent` / `agent`

Sources: <https://cursor.com/docs/cli/headless>, <https://cursor.com/docs/cli/reference/output-format>, <https://cursor.com/docs/cli/using>.

| Field | Fact | Tag |
|---|---|---|
| One-shot | `agent -p/--print "<prompt>"` | D |
| Prompt input | argv; print mode is also inferred for piped stdin / non-TTY stdout, which implies stdin prompts | D/I |
| Output | `text` (default, final assistant message only), `json`, `stream-json` (+`--stream-partial-output`) | D |
| JSON shape | `{type:"result", subtype:"success", is_error:false, result:"...", session_id, duration_ms}` | D |
| Failure | non-zero exit and stderr message; **no JSON object emitted** | D |
| Schema | none documented | D |
| Files | without `--force`/`--yolo`, edits are proposed not applied (headless page); the "Using" page says "Cursor has full write access in non-interactive mode" — docs conflict | D |
| Sandbox | no documented headless sandbox/allowlist; community request open | I |
| Auth | `CURSOR_API_KEY` or `cursor-agent login`; `cursor-agent status` | D |
| Context | reads `AGENTS.md`, `CLAUDE.md`, `.cursor/rules` from project root | D |

**Fit:** good output shape (`.result`, clean text). No schema enforcement → needs the extractor + validator. Contradictory write-permission docs mean: run in a throwaway directory and never pass `--force`.

### 3.3 Aider

Sources: <https://aider.chat/docs/scripting.html>, <https://aider.chat/docs/config/options.html>.

| Field | Fact | Tag |
|---|---|---|
| One-shot | `aider --message "..."` / `--message-file FILE` (`-f`) | D |
| Prompt input | argv or file; **no documented stdin prompt** | D |
| Output | chat transcript on stdout: prompt echo, streaming markdown, "Applied edit…" lines, token/cost summary | D/I |
| JSON | none | D |
| Schema | none | D |
| Files | edits files in the repo by default; `--dry-run`, `--no-auto-commits`, `--no-git` | D |
| Clean output knobs | `--no-pretty`, `--no-stream`; `--yes-always`; `--exit`; `--no-check-update` | D |
| Exit codes | not documented in the scripting page | — |
| Fit | **poor.** Its output channel is file edits plus decorated chat, not a final answer. Usable only with `--no-git --dry-run --no-pretty --message-file`, and even then the JSON must be scraped out of prose. | D/I |

### 3.4 Ollama

Sources: <https://docs.ollama.com/cli>, <https://docs.ollama.com/capabilities/structured-outputs>.

| Field | Fact | Tag |
|---|---|---|
| One-shot | `ollama run <model> "<prompt>"`, or prompt on stdin (`echo ... | ollama run model`) | D |
| CLI JSON flag | no `--format` documented for `ollama run`; JSON mode is an **API** field | D/I |
| API JSON | `POST /api/chat` with `"stream": false`, `"format": "json"` or a full JSON Schema | D |
| Schema | yes, via `format`; recommended to also paste the schema in the prompt; `temperature: 0` | D |
| Output | CLI prints raw model text; API returns `message.content` | D |
| Cloud | **structured outputs not supported on Ollama Cloud** | D |
| Auth | local daemon, no key; cloud sign-in for cloud models | D |
| Fit | best treated as an HTTP backend, not a CLI. If forced through the shell contract, use `curl` with a request file. | I |

### 3.5 Cross-check on installed CLIs (docs vs local)

- Claude: `--json-schema` is the only CLI here with native schema-constrained output *and* a documented `structured_output` field. [D]
- Gemini: no schema flag; envelope is `{response, stats, error}`; exit table omits the observed 41. [D/V]
- OpenCode: no schema flag; `--format json` is an event log. [V]

---

## 4. Comparison table

Legend: **P** prompt input; **JSON** is there a machine format; **Schema** native schema enforcement; **Pure** is stdout the answer alone; **Write** can it modify files; **Exit** can exit code be trusted; **Auth** how it authenticates.

| CLI | P argv | P stdin | P file | JSON | Schema | Pure stdout | Write by default | Exit trust | Auth | One-shot flag |
|---|---|---|---|---|---|---|---|---|---|---|
| claude | yes | yes (10 MB cap) | via prompt text | envelope / NDJSON | **yes** `--json-schema` | envelope only | tools allowed; deny via mode | partial (JSON `is_error`) | API key / OAuth | `-p` |
| codex | yes | yes (`-` ) | via stdin | NDJSON events / `-o` file | **yes** `--output-schema` | **yes** (final msg) | read-only sandbox default | **no** (0 on task failure) | saved login / `CODEX_API_KEY` | `exec` |
| gemini | yes | yes (appended) | no | envelope / NDJSON | no | envelope only | approval mode; `plan` = read-only | mostly (0/1/42/53 + 41) | API key / Vertex / login | `-p` |
| opencode | yes | **no** | `--file` attach | NDJSON events | no | no | yes | **no** (0 always) | Console/provider | `run` |
| cursor-agent | yes | inferred | no | envelope / NDJSON | no | `text` is clean | no `--force` = propose only | yes (non-zero, no JSON) | API key / login | `-p` |
| aider | yes | no | `--message-file` | **no** | no | no (chat + diffs) | yes (disable with `--dry-run --no-git`) | unknown | provider key | `-m` |
| ollama | yes | yes | no | API `format` | **yes** (API) | CLI text is raw | no | local: yes | none / cloud login | `run` |

**Read this table as:** only Claude, Codex, and Ollama can *enforce* a schema; only Codex and Cursor reliably put the final answer alone on stdout; only Claude and Codex can be made read-only with a documented flag. No single invocation works everywhere.

---

## 5. Recommended contract

### 5.1 Principles

1. **The tool owns the prompt, schema, validation, and repair.** The agent only transforms text. (Matches ADR 0002.)
2. **Config supplies a command template plus an output adapter.** The template is data, not code; the adapter is a small enum, not a plugin.
3. **Prompt goes in a file or on stdin, never concatenated into a shell string.** Arabic text, newlines, quotes, and `$` must never pass through shell parsing.
4. **A run succeeds only when the parsed JSON validates.** Exit code and "looks like JSON" are hints, not gates.
5. **Every run leaves artifacts** so a human can inspect and re-ingest without re-spending quota.
6. **Version-pin the agent and adapter.** Flags and exit codes moved between 2025 and 2026 (Gemini's 41, Codex's removed `--full-auto`, Claude's `--bare` auth split).

### 5.2 Config shape

TOML, project config (e.g. `excalillm.toml`). Template string per ADR 0002; `argv` variant for users who want no shell.

```toml
[plan]
agent = "claude"              # key into [agents.*]
timeout_seconds = 420         # per attempt, wall clock
max_repair_attempts = 2
scratch_dir = true            # run in an empty temp cwd, never the repo
schema = "schemas/scenes.schema.json"

[agents.claude]
# Placeholders are substituted by the tool with shell-quoted values.
# {prompt_file} {schema_file} {schema_json} {out_file} {cwd}
command = 'claude --bare -p --output-format json --json-schema {schema_json} --permission-mode dontAsk --disallowedTools "Bash,Edit,Write,Read,WebFetch,WebSearch" --no-session-persistence'
prompt = "stdin+argv"         # fixed instruction is in argv; transcript on stdin
output = "json:.structured_output|json:.result"
timeout_seconds = 420
# auth_check = "claude auth status"   # optional; may lie, see §2

[agents.codex]
command = 'codex exec --skip-git-repo-check --sandbox read-only --ephemeral --ignore-user-config --ignore-rules --output-schema {schema_file} -o {out_file} -'
prompt = "stdin"              # "-" makes stdin the prompt
output = "file:{out_file}|raw"
timeout_seconds = 600

[agents.gemini]
command = 'gemini -p "Return only the JSON object described by the schema in the prompt." --output-format json --approval-mode plan --skip-trust'
prompt = "stdin+argv"
output = "json:.response"
timeout_seconds = 420

[agents.cursor]
command = 'cursor-agent -p --output-format json'
prompt = "stdin"
output = "json:.result"
timeout_seconds = 420

[agents.opencode]
command = 'opencode run --format json --auto "Follow the instructions in {prompt_file} and write only the JSON answer."'
prompt = "file"               # opencode ignores stdin
output = "jsonl:last-text-part"
timeout_seconds = 600

[agents.aider]
command = 'aider --no-git --no-auto-commits --yes-always --no-pretty --no-stream --no-check-update --dry-run --message-file {prompt_file} --exit'
prompt = "file"
output = "raw"                # poetry: extractor must find the object in prose
timeout_seconds = 600

[agents.ollama]
command = 'curl -sS --max-time 400 http://localhost:11434/api/chat -H "Content-Type: application/json" --data-binary @{request_file}'
prompt = "request_file"       # tool writes the full messages+format JSON body
output = "json:.message.content"
timeout_seconds = 420
```

**Why a template and not only argv.** The ADR commits to a shell template, and it lets users add wrappers (`timeout`, `nice`, `env -i`, `firejail`, `ssh`). The danger is quoting. Mitigation: the tool performs substitution itself with `shlex.quote` on each placeholder and then runs `/bin/sh -c`. The transcript never appears in the template; it is a file or stdin. An optional `argv = [...]` form can be added for users who want zero shell; both compile to the same internal argv.

**Placeholders.** `{prompt_file}` is a UTF-8 file containing the complete prompt (instruction + transcript + schema + output rules). `{schema_file}` is the raw schema file. `{schema_json}` is the same schema compacted and shell-quoted (Claude needs inline). `{out_file}` is a path the tool will read after the process exits. `{cwd}` is the scratch directory.

### 5.3 Prompt delivery: stdin vs argv

| Payload | Delivery | Reason |
|---|---|---|
| instruction + schema + transcript (~15–25 KB Arabic) | **file**, then either stdin or a path in argv | immune to quoting, newlines, `$`, backticks; `ARG_MAX` is ~2 MB so size is not the issue, shell syntax is |
| short fixed instruction | argv in the template | readable, stable |
| transcript only | stdin when the CLI documents it | no temp-file lifetime race |

Rules:
- Always write `prompt.md` first, then feed it. Even in stdin mode, keep the file for the repair round and for human inspection.
- Prefer `prompt = "file"` when the CLI supports a file (`aider --message-file`, opencode `--file`, codex `--output-schema` is schema only). Otherwise `stdin`.
- Never build `command + " " + transcript`. If a user writes `{prompt}` in the template, refuse at config load and point them at `{prompt_file}`.
- Close stdin after writing. A CLI that ignores stdin (OpenCode) otherwise hangs until timeout [V].

### 5.4 Output adapters

```text
raw                     # stdout as the answer text
json:<dotted.path>      # parse stdout as one JSON doc, take path
json:<p1>|<p2>          # first present/non-null path
jsonl:<selector>        # NDJSON; select final answer text
file:<path>             # read answer text from a file the CLI wrote
```

`jsonl:last-text-part` is the only special selector initially: scan events, keep `part.text` where `part.type == "text"` (OpenCode) or `item.text` where `type == "item.completed"` and `item.type == "agent_message"` (Codex `--json`), return the last non-empty. Treat unknown event shapes as a hard error, not as empty output.

**Always run the extractor after the adapter.** Even `.structured_output` can be a string containing a fenced block in some versions; treat the adapted value as *candidate text*, then:

1. `json.loads(text.strip())` — accept BOM, trim whitespace.
2. Else find fenced blocks ` ```json ... ``` `; try each.
3. Else scan for the first balanced `{`…`}` / `[`…`]` with string/escape awareness (not regex, not greedy).
4. Else fail with `no_json_found`, storing the raw text.

Record which stage succeeded (`direct`, `fence`, `scan`) in `meta.json`. A high `scan` rate is a signal that the agent profile needs a stronger instruction or a different adapter.

### 5.5 Validation and repair

**Stage A — parse.** JSON syntax; reject duplicate keys if the parser allows detecting them; reject non-finite numbers.

**Stage B — schema.** `additionalProperties: false`; require every field; types; enums for transition (`cut|draw`); integer seconds or float with documented precision.

**Stage C — domain invariants** (things JSON Schema cannot say):
- scenes ordered, `start < end`, first start ≥ 0, last end ≤ audio duration + 0.5 s
- no overlaps unless the schema explicitly allows crossfades
- scene count within configured bounds
- every `text_ar` field non-empty and actually Arabic (contains Arabic codepoints); reject Latin-only placeholders
- every quoted transcript fragment appears in the source transcript (normalize diacritics/whitespace first); catches paraphrase-to-hallucination drift
- element IDs unique within a scene
- cue times fall inside their scene

**Repair loop.** Max 2 additional attempts (3 total). Each repair prompt contains: the schema, the validator's exact error (path + message), the previous output (or just the failing fragment if long), and "Return only the corrected JSON. Do not explain." If the agent profile supports native schema output, keep using it; otherwise the prompt must include the schema verbatim. On final failure: write `failure.json` with the validation error and raw response, exit non-zero, and print the artifact directory. Never return partially repaired JSON.

**No silent truncation.** If the response looks like valid JSON but scene count is below a configured floor, or the last scene lacks an end time, treat as truncated and repair rather than accept.

### 5.6 Timeouts, process control, and limits

- Per-attempt wall clock: 420 s default; 600–900 s for >1,000 words or slower local models. Observed retry ladders reached 80–120 s before any answer [V].
- Spawn in a new process group; on timeout send SIGTERM, wait 5 s, SIGKILL the group. Claude documents SIGTERM → exit 143 and a dropped partial result [D], so treat 143 as timeout, not a model error.
- Cap stdout/stderr at e.g. 8 MB each; stream to files; never `2>&1` when the answer is on stdout (Codex puts progress on stderr; merging corrupts the answer).
- Serialize runs per agent profile: CLIs share global config, session directories, and sometimes a single server (OpenCode background service). A lock file per profile avoids interleaved sessions.
- Scrub the environment: pass only `PATH`, `HOME`, locale, and the profile's declared env. Do not leak unrelated API keys into an agent that can run Bash.
- `cwd = scratch` is mandatory. All of these agents auto-load project rules (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules`, `GEMINI.md`), which can change tone, add preambles, or make the agent edit files instead of answering [D]. A temp dir with no repo, no `AGENTS.md`, and no `.mcp.json` is the only portable defense.

### 5.7 Exit-code handling

Success predicate:

```text
process_exited_within_timeout
AND output_adapter_produced_candidate_text
AND json_parse_stage_succeeded
AND schema_valid
AND domain_invariants_hold
```

Exit code is recorded but only used to short-circuit obvious failures:

| Agent | Observed/documented | Policy |
|---|---|---|
| claude | 0 ok; 1 on API error; 143 SIGTERM [D/V] | if JSON envelope has `is_error:true`, fail even on 0 |
| gemini | 0/1/42/53 [D], 41 observed [V] | any non-zero = fail; 41 = auth |
| opencode | 0 always [V] | ignore exit code entirely |
| codex | 0 even on task failure [I] | ignore; require `-o` content to validate |
| cursor | non-zero on failure, no JSON [D] | non-zero = fail |
| aider | undocumented | rely on output validation |
| ollama | curl exit / HTTP status | check HTTP status and `.error` |

### 5.8 File-convention escape hatch

For interactive-only agents, GUI agents, or a human who wants to use their own chat window. The tool creates a job directory and stops:

```text
.excalillm/jobs/2026-10-24T19-44-12/
  prompt.md        # complete instructions + transcript + schema + output rules
  input.json       # {job_id, audio_duration_s, transcript:[{t_start,t_end,speaker,text_ar}], schema_version}
  schema.json      # the contract
  README.txt       # one paragraph: "Run your agent on prompt.md, save the JSON as scenes.json"
  job.json         # {state:"awaiting_agent", created_at, agent_hint}
```

Human/agent writes `scenes.json` in that directory. Next tool invocation:

```bash
excalillm plan --ingest .excalillm/jobs/2026-10-24T19-44-12/
# or from a GUI chat: pbpaste | excalillm plan --ingest - --job <id>
```

Ingest rules: accept `scenes.json` only if it validates; move to `state:"validated"`; on failure write `validation.json`, keep state `awaiting_agent`, and print the exact error so the human can paste it back into the chat. This path has no timeout and no quota cost, and it is the only one that works with agents that cannot pipe (Claude Desktop, web chat, IDE chat).

Make this the **default** when no agent is configured; do not fail with "no agent found".

### 5.9 Example invocations

```bash
# Claude, schema-enforced, read-only tools, no session persistence
printf '%s' "$PROMPT" | claude --bare -p \
  --output-format json \
  --json-schema "$(jq -c . schema.json)" \
  --permission-mode dontAsk \
  --disallowedTools "Bash,Edit,Write,Read,WebFetch,WebSearch" \
  --no-session-persistence \
  | jq -e '.structured_output // empty'

# Codex, read-only sandbox outside a git repo, final message to file
codex exec - \
  --skip-git-repo-check --sandbox read-only --ephemeral \
  --ignore-user-config --ignore-rules \
  --output-schema schema.json -o scenes.json < prompt.md

# Gemini, headless, JSON envelope
gemini -p "Return only the JSON object. No prose." \
  --output-format json --approval-mode plan --skip-trust < prompt.md \
  | jq -er '.response | fromjson'

# Cursor, clean text in .result
cursor-agent -p --output-format json < prompt.md | jq -er '.result | fromjson'

# OpenCode, argv-only, prompt attached as a file, NDJSON event stream
opencode run --format json --auto \
  "Read prompt.md in this directory. Answer with JSON only." \
  --file prompt.md
```

(All of the above still pass through the tool's extractor and validator; `jq fromjson` is illustrative.)

---

## 6. Where a universal contract is impossible

1. **No common invocation shape.** Aider has no stdin prompt; OpenCode requires argv; Claude needs `-p`; Codex needs the `exec` subcommand. A single template cannot span them without per-agent branches — hence named profiles.
2. **No common output envelope.** Bare text, one JSON document, and NDJSON are all in use. "JSON on stdout" is a goal, not a property any vendor guarantees.
3. **No common failure signal.** Two CLIs exit 0 on failure [V/I]; one emits `subtype:"success"` alongside `is_error:true` [V]; Gemini uses an undocumented 41 [V]. Only validation can decide.
4. **No common schema enforcement.** Three CLIs have it, four do not; Codex's is reported to break when tools are active [I]. The validator is non-optional.
5. **No common sandbox.** Codex has read-only by default; Claude needs a permission mode and tool denylist; Cursor has no documented headless sandbox; Aider and OpenCode write freely. "Read-only" cannot be promised across agents, only requested per profile.
6. **No common context isolation.** Every agent auto-loads project rules from cwd. `--bare`, `--ignore-user-config`, `--skip-trust` exist in different combinations. A scratch cwd is the only portable mitigation, and it is a convention, not an interface.
7. **No common truncation signal.** Long Arabic output can hit output-token ceilings; some agents return valid-but-short JSON rather than an error. Only a semantic floor catches this.
8. **No stable interface over time.** Flags were removed (`--full-auto`), auth modes split (`--bare` vs OAuth), and exit codes changed within 2026. Any contract is a snapshot plus a capability probe.
9. **Agents are agents.** They may decide to read files, run Bash, or edit the repo instead of answering. A prompt can ask for a pure transform; it cannot make the process a pure function. The file-convention fallback and the scratch cwd are the honest concession.

**What *is* universal:** the *artifacts* — `prompt.md`, `input.json`, `schema.json`, `scenes.json` — and the *validator*. Any agent, including a human, can participate through those four files. That is the real contract; the shell template is an optimization for agents that can pipe.

---

## 7. Failure modes to design around

| # | Failure | Evidence | Mitigation |
|---|---|---|---|
| 1 | Exit 0 on failure | OpenCode always [V]; Codex reported [I] | validate output; ignore exit code for those profiles |
| 2 | Error text on stdout | Claude `result`, Gemini `error` [V] | parse envelope; check error fields before extracting |
| 3 | Fenced/prose-wrapped JSON | common LLM behaviour; Cursor `text` mode [D/I] | three-stage extractor; record stage |
| 4 | JSONL instead of JSON | Codex `--json`, OpenCode `--format json` [D/V] | jsonl adapter; never `json.loads(whole_stdout)` |
| 5 | Prompt ignored on stdin | OpenCode "You must provide a message" [V] | per-profile prompt mode; file attachment |
| 6 | Hang with no output | Claude text mode >120 s [V] | process-group timeout; JSON output format always |
| 7 | Auth looks present but isn't | OpenCode stored credential + refused transport [V] | real probe (`--probe`) or first-job failure surfaced clearly |
| 8 | Auth error codes undocumented | Gemini 41 [V] vs docs 0/1/42/53 [D] | map known codes, otherwise show raw stderr/stdout |
| 9 | Project rules change the answer | all agents load cwd rules [D] | empty scratch cwd; `--bare`/`--ignore-*` where available |
| 10 | Agent edits files instead of answering | Aider/OpenCode/Cursor write access [D] | read-only flags where possible; scratch cwd; denylist tools |
| 11 | Schema ignored when tools active | Codex #15451 [I] | validate; repair; try text-only profile |
| 12 | Truncated array | output-token ceilings [I] | semantic floor (min scenes, complete last scene); repair |
| 13 | Arabic gets transliterated or Latinized | observed class of LLM error [I] | require Arabic codepoints in arabic fields |
| 14 | Hallucinated timestamps | LLM arithmetic [I] | bounds check against audio duration; monotonicity |
| 15 | Duplicate/ambiguous scene IDs | model drift | schema pattern + uniqueness check |
| 16 | Repair loop diverges | model keeps same error | max 2 repairs; then fail with artifacts |
| 17 | Partial JSON accepted | naive `json.loads` on substring | balanced scan + full parse; never regex |
| 18 | Cost blow-up on retries | 1,500-word prompt × 3 attempts × large model | max attempts + `--max-budget-usd` where supported + per-job token log |
| 19 | Private transcript persisted by agent | Claude writes session JSONL by default [D] | `--no-session-persistence`; codex `--ephemeral`; scratch HOME where feasible |
| 20 | Concurrent runs corrupt session state | shared config dirs / background server [I] | per-profile lock; unique session ids |
| 21 | Windows/`sh` differences | shell template [I] | document POSIX-only for template profiles; offer `argv` mode |
| 22 | Template injection via prompt | shell template design | tool-side quoting; forbid `{prompt}` inline |

---

## 8. Support recommendation

**First-class profiles** (adapter + tested templates): `claude`, `codex`, `gemini`, `cursor-agent`. These have a documented one-shot mode, a machine output format, and a usable final-answer channel.

**Second-class**: `opencode` (argv-only, NDJSON, exit 0 always), `ollama` via HTTP API (schema support, but cloud lacks it).

**Last resort**: `aider` — no JSON mode, output is a chat transcript; support it only through the file-convention path or a `raw` extractor, and tell users it is not recommended.

**Always available**: the file convention. It is the only path that works for every agent, costs no quota, and keeps the human in the loop.

---

## 9. Open questions

1. Do we accept the ADR's shell template as the only config form, or ship `argv` alongside it? Template is required by the ADR; argv removes a class of quoting bugs.
2. Is `--bare` acceptable as the default Claude profile, given it breaks subscription/OAuth users? Alternative: scratch cwd without `--bare` (keeps OAuth, but still reads `~/.claude`).
3. Do we require a live probe before a job (`--probe`), given that auth status commands lie and probes cost quota?
4. Should `structured_output` be preferred over `.result` when both exist, and do we fall back on repair if the field is absent?
5. Is a minimum scene count a safe truncation heuristic for short transcripts, or does it reject legitimately sparse plans?
6. Which transcript normalization is acceptable for the "quoted fragment exists" hallucination check (diacritics, tatweel, whitespace, punctuation)?
7. Do we pin agent CLI versions in the job metadata for reproducibility, and fail when the version drifts?

---

## 10. Sources

Local: `claude --help`, `claude auth status`, `claude -p ... --output-format json`; `opencode --help`, `opencode run --help`, `opencode auth list`, `opencode run --format json`; `gemini --help`, `gemini -p ... -o json`; PATH probe for all seven. Commands run 2026-10-24.

Docs:
- Codex non-interactive: <https://learn.chatgpt.com/docs/non-interactive-mode>
- Codex issues (schema + tools, exit 0): <https://github.com/openai/codex/issues/15451>, <https://github.com/openai/codex/issues/15536>, <https://github.com/openai/codex/issues/37717>
- Cursor headless: <https://cursor.com/docs/cli/headless>; output format: <https://cursor.com/docs/cli/reference/output-format>; using: <https://cursor.com/docs/cli/using>
- Claude headless: <https://code.claude.com/docs/en/headless> (redirect from docs.claude.com)
- Gemini headless + exit codes: <https://geminicli.com/docs/cli/headless/>
- Aider scripting: <https://aider.chat/docs/scripting.html>; options: <https://aider.chat/docs/config/options.html>
- Ollama CLI: <https://docs.ollama.com/cli>; structured outputs: <https://docs.ollama.com/capabilities/structured-outputs>
