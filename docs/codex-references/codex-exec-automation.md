# Codex Exec for Automation – Detailed Guide

## Overview

`codex exec` is the headless edition of the Codex CLI. It runs an entire Codex session to completion without the interactive TUI, making it suitable for CI pipelines and other automated workflows.

## Installation, Authentication, and Invocation

Codex publishes prebuilt binaries via `@openai/codex` on npm; install globally, authenticate with `codex login`, and invoke `codex exec` in your scripts (see the GitHub Action sketch in the advanced guide).

Running `codex exec PROMPT` (or piping a prompt on stdin) launches a non-interactive session that exits when the agent is done; `RUST_LOG` controls logging (defaults to `error`).

## CLI Options and Configuration Controls

All primary flags are defined in `exec/src/cli.rs`. Key options include:
- **Prompt delivery** – pass as an argument or `-` to read stdin. The runner refuses to hang waiting for stdin unless forced with `-`.
- **Image attachments** – `--image/-i` accepts one or more file paths; the runner uploads them before sending the text prompt.
- **Model selection** – `--model/-m` overrides the configured model; `--oss` enables the built-in open-source provider and ensures the default OSS model is installed.
- **Sandboxing** – `--sandbox/-s` selects a sandbox policy; `--full-auto` is shorthand for `workspace-write`, while `--dangerously-bypass-approvals-and-sandbox` disables protections (mutually exclusive). Regardless of flags, automation mode sets the approval policy to `never`, so Codex auto-runs commands.
- **Working directory** – `--cd/-C DIR` sets the session root; by default, the runner insists on a Git repo unless `--skip-git-repo-check` is supplied.
- **Color and human output** – `--color` (auto/always/never) controls ANSI when not using JSON.
- **JSON mode** – `--json` switches to machine-readable output; `--output-last-message` writes the final assistant reply to a file.
- **Config overrides** – `-c/--config key=value` applies ad-hoc overrides (values parsed as TOML/JSON) before loading `~/.codex/config.toml`. Profiles can be selected via `--profile/-p` too.

During initialization `run_main` loads configuration with the chosen overrides, ensures OSS models are present when requested, prints a configuration summary, and spawns an async event loop to drain Codex events until shutdown.

Keyboard interrupts trigger an `Op::Interrupt` submission to gracefully cancel outstanding work.

## JSON Output Contract

### Prologue Lines

In JSON mode the runner emits two preface lines:
1. A JSON object summarizing the effective configuration (workdir, model, provider, approval mode, sandbox description, and optional reasoning settings).
2. `{"prompt": "<text>"}` so automation can log or audit the exact instructions sent.

### Event Stream Format

Subsequent lines are JSON objects representing Codex protocol events. Each payload matches the `Event` struct and `EventMsg` enum, serialized with a `type` discriminator inside `msg`. A regression test confirms the structure (`{ "id": "...", "msg": { "type": "...", ... } }`).

Streaming deltas (`agent_message_delta`, `agent_reasoning_delta`) are suppressed in JSON mode to avoid partial tokens; everything else, including reasoning sections and tool output, is forwarded verbatim.

The event `id` matches the submission that produced it (e.g., initial prompt, follow-up image upload). Once the agent emits `task_complete`, the runner writes the optional last message file, submits `Op::Shutdown`, and exits when `shutdown_complete` arrives.

## Event Catalogue

Below is a comprehensive map of the `EventMsg` variants you may receive, grouped by concern. Use the `type` field to switch on behavior.

### Session & Conversation Metadata
- `session_configured`: reports the assigned session UUID, model, optional reasoning effort, history log id/count, any seed events for resumed sessions, and the rollout file path. Use this to persist conversation IDs.
- `conversation_path`: reveals the filesystem path to the live rollout and the conversation id (same type as above).
- `plan_update`: carries a structured plan (steps plus statuses and optional explanation).
- `turn_aborted`: indicates that the current turn was interrupted or replaced (e.g., by Ctrl+C).
- `shutdown_complete`: final confirmation that the agent stopped; treat as end-of-session.

### User/Agent Messaging and Reasoning
- `user_message`: echoes text sent to the model, optionally annotated with `kind` (plain vs. XML instructions/environment) and inline image data (data URLs).
- `agent_message`: assistant messages emitted as full strings. Deltas are suppressed in JSON mode.
- `agent_reasoning`, `agent_reasoning_raw_content`, `agent_reasoning_raw_content_delta`, `agent_reasoning_section_break`: expose reasoning traces when enabled. Only reasoning deltas are suppressed; raw content deltas remain, so clients can reconstruct chain-of-thought segments if desired.

### Task Lifecycle & Token Usage
- `task_started`: optionally reports the model’s context window size for the run.
- `token_count`: supplies aggregate usage via `TokenUsageInfo` (totals and last-turn stats). Each `TokenUsage` breaks down input, cached input, output, reasoning output, and total tokens; helper methods annotate cached vs. non-cached input and estimate context-window remaining percentage.
- `task_complete`: signals that Codex believes the task is done and includes the final assistant message, if any.

### Tool Invocation & Automation
- `mcp_tool_call_begin` / `mcp_tool_call_end`: wrap Model Context Protocol tool calls with server/tool names, arguments, durations, and success/error results. The `McpInvocation` object contains the server and tool names, and `result` is either a `CallToolResult` or an error string.
- `web_search_begin` / `web_search_end`: capture agent-initiated web searches (call id plus final query string).
- `exec_command_begin`: announces a shell command with its parsed argument vector, working directory, and a semantic classification (`ParsedCommand` differentiates reads, file listings, searches, etc.).
- `exec_command_output_delta`: streams raw bytes (base64-encoded) from stdout/stderr; a test suite ensures the chunk is base64 encoded for safe transport.
- `exec_command_end`: supplies the aggregated output, formatted output (what the model sees), stdout/stderr captures, exit code, and duration.
- `exec_approval_request`: even though automation mode auto-approves, the event schema exists—command vector, working directory, reason string—allowing tooling to audit or enforce policies.
- `background_event`, `stream_error`: surface informational background messages or retry notices from the Codex runtime.

### File Modification & Patch Workflow
- `apply_patch_approval_request`: describes requested patch diffs (per-path `FileChange` objects) and optional grant roots or explanations.
- `patch_apply_begin` / `patch_apply_end`: mirror exec events for patch application, including auto-approval status and resulting stdout/stderr/success flag.
- `turn_diff`: provides a unified diff summarizing the net changes for the turn.

### Plan & Review Mode
- `plan_update`: described above; carries multi-step plan updates.
- `entered_review_mode` / `exited_review_mode`: signal that a nested review session started/finished, with structured findings (`ReviewOutputEvent`) including confidence, correctness, and per-file issues.

### History, Discovery, and Customization
- `get_history_entry_response`: returns a single historical entry, if available, for incremental transcript retrieval.
- `mcp_list_tools_response`: lists MCP tool definitions currently available to Codex.
- `list_custom_prompts_response`: enumerates any configured custom prompts.
- `initial_messages` (embedded in `session_configured`): for resumed runs the server can replay earlier `EventMsg`s to seed clients. Supporting types like `ResumedHistory` and `InitialHistory` document how history is bundled.

### Errors
- `error`: wraps high-level failures with a human-readable message.

## Tool Execution Semantics

The combination of `exec_command_*` events lets automation trace every shell command Codex runs: when it started, structured intent (`ParsedCommand`), incremental output (raw bytes labeled stdout/stderr), final formatted output, duration, and exit status.

Base64 encoding ensures binary-safe streaming; the test suite confirms encode/decode symmetry.

Similar begin/end events exist for patch application and MCP tools, giving consistent lifecycle hooks for every automated action.

Even though `codex exec` auto-approves by default, `exec_approval_request` and `apply_patch_approval_request` events still appear when Codex wants to log its decision. They record command vectors, reasons, diff payloads, and optional session-level write permissions so policy engines can monitor or override behavior.

## Reasoning and Streaming Notes

Reasoning output can arrive in several flavors: summarized text (`agent_reasoning`), raw chain-of-thought (`agent_reasoning_raw_content`), incremental deltas (`agent_reasoning_raw_content_delta`), and section breaks for structured summaries. Only the delta stream is muted to keep JSON clean; full reasoning payloads remain available.

Standard assistant text appears via `agent_message`; partial `agent_message_delta` updates are filtered for the same reason.

## Session Management and Resume

`codex exec resume` continues an existing rollout, sharing the same conversation id and appending new events. The CLI accepts `SESSION_ID`, `--last`, and an optional follow-up prompt (or `-` for stdin).

At runtime, `resolve_resume_path` locates the requested rollout by listing conversations or matching IDs, then the `ConversationManager` spins up a resumed session with the prior history.

The advanced docs describe equivalent commands and note that resuming retains the original rollout file and conversation id.

Every session id uses the `ConversationId` wrapper (UUID as string), so clients can safely treat it as an opaque string for storage and routing.

## Token Usage and Caching Metrics

`TokenUsage` tracks five counters: `input_tokens`, `cached_input_tokens` (tokens pulled from cache), `output_tokens`, `reasoning_output_tokens`, and `total_tokens`. Helper methods break out non-cached input, compute a blended total (non-cached input + output), and estimate how much of the model’s context window remains after subtracting a baseline for system prompts/tools.

`TokenUsageInfo` pairs cumulative totals with the last turn’s usage and echoes the model context window size so dashboards can track trends.

`TokenCount` events wrap this data (nullable when the model doesn’t report usage), giving automation hooks to watch for context exhaustion or to meter costs.

## Last Message Capture

`TaskComplete` includes the final assistant message; when `--output-last-message` is provided, the runner writes that string (or an empty file if absent) and warns when no message was available.

After emitting `task_complete` the runner sends `Op::Shutdown` and exits once `shutdown_complete` appears.

## Images and Attachments

Image paths passed via `--image` are converted to `InputItem::LocalImage` entries, uploaded first, and acknowledged with a dedicated `TaskComplete` before the text prompt runs. This ensures the agent has the image context before processing the main request.

## Configuration Summary Semantics

The config summary printed at the start helps log automation context. Keys include `workdir`, `model`, `provider`, `approval`, `sandbox`, and optional `reasoning effort` / `reasoning summaries` when the provider supports reasoning summaries (Responses API). Sandbox summaries enumerate writable roots and whether network access is enabled.

## Example Automation Workflow

1. Install Codex (`npm i -g @openai/codex`), authenticate, and provision secrets in your CI environment.
2. Prepare a working directory (optionally pass `--cd`). Ensure you’re inside a Git repo or add `--skip-git-repo-check` if the workspace is intentionally bare.
3. Launch `codex exec --json` with the desired prompt, sandbox, model overrides, images, and config overrides as needed. Set `RUST_LOG` if you need verbose traces.
4. Read the first JSON line to capture the effective configuration and the second for the prompt. Store them with your job logs for auditing.
5. Stream events line-by-line, branching on `msg.type`. Handle tool events (`exec_command_*`, MCP calls, patch events) to mirror Codex’s actions in your logs, respond to review or plan updates, and record token usage for metering.
6. Treat `task_complete` as the signal that the run finished. If `--output-last-message` was provided, read the saved file; otherwise, use the embedded `last_agent_message`. Wait for `shutdown_complete` before terminating your job, ensuring all streams were drained.
7. Persist the `session_configured.session_id` and `rollout_path` if you plan to resume the conversation later with `codex exec resume` or to ingest the rollout file offline.

With these hooks you can fully automate Codex runs, track every tool invocation, capture reasoning, monitor cache usage, and safely resume long-lived sessions.
