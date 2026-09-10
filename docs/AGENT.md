# AI beside your mail

OmamailAI reads the default AI selected in Omarchy. There is no OmamailAI settings
page or separate Agent page. The background adapter supports Claude and Codex;
other defaults produce an inline explanation without opening a terminal or picker.
The installed CLI uses its normal system login and model/provider configuration. OmamailAI does not store AI keys or change the global agent selection.

Choose the outline **AI icon** button beside Compose in the window header, the
message menu, or `Alt+G` in the list, reader or composer. The right dock displays
the conversation, aligned to the bottom with older turns above. Drag its left
edge to resize; double-click the divider to restore the default width. User messages
have a background and a › marker; AI replies have no background and each offers
a copy icon after generation finishes that copies the original reply. Bold and code are formatted
through an escaping formatter that cannot create links or remote resources. Execution status appears just above the input, which starts at one line and
grows with newlines.
Type `/` to show commands, then use Up/Down and Return or click a suggestion.
Selecting a command only fills editable instructions. **Enter** sends;
**Shift+Enter** inserts a newline. Ctrl+Enter also sends. While running, the
stop icon ends the request. The **…** menu contains **New chat** and **History...**
for the current mail or draft. The header AI button closes the dock without stopping an active request.
While running, a timed Working line stays above the input and Escape interrupts
the request; when idle, Escape closes the dock.

While AI is working, Enter adds another message to a Pending queue and clears
the input immediately. Messages run in order in the same conversation after
each successful reply. Click a pending message to bring it back into an empty
input for editing, or remove it with ×. A failed start retains the message;
interrupting or a failed reply pauses the queue. Pending messages are held only
for this application session, with up to 20 messages and bounded text size.
They never switch to another mail or conversation. The queue for a request still
preparing its first turn waits until that conversation can be identified.

The worker runs silently in the background. Text appears progressively, along
with public status events such as reading a file or finishing a tool. Raw tool
arguments/results, diagnostics, and internal reasoning are not displayed.
Completed requests can be followed up in the same native agent conversation.
The **New chat** action reads the current mail or draft into a fresh conversation.
Follow-ups keep the original context; a notice identifies a draft edited since
that context was captured.

A request can cover at most 20 messages from one mailbox. The owning provider's
normal read interface supplies complete bodies without selecting or marking mail
read. Results stay bound to their account, messages and draft identity. You can
select history text or copy each answer. Translation and rewriting commands act
only on mail titles and bodies, excluding addresses and metadata. Their results
separate Title and Body; draft insertion takes only the Body section, so a
translated title is not accidentally inserted into the body. In a draft, **Insert at cursor**
and **Replace body** apply only a completed successful reply; neither sends mail.
Replacement is two text edits and can require two undo steps. The mail list has no AI icon. The header AI button stays static without a
breathing animation.

## Background bridge

`AgentContext.qml` loads bodies; `Agent.js` builds context and matches identity.
`AgentRunner.qml` starts a Python worker and polls validated display snapshots.
`scripts/agent-job.py` reads requests on stdin and launches the installed Claude or Codex
CLI with non-interactive streaming JSON output. Mail and questions reach the CLI
through stdin, never process arguments. No terminal launcher is invoked.

Each turn has a private 0700 directory under
`$XDG_STATE_HOME/omamailai/assistant/<turn-id>/`; files are 0600. The parser imports
only bounded, validated UTF-8 public text/status events. It rejects malformed or
incomplete streams and never treats a partial response as a successful draft
suggestion. Per-turn answers are limited to 64 KiB; conversation snapshots have
bounded entries and bytes. Oversized history requires a new chat instead of
silently dropping context. At most four requests run and 32 turns are retained.
Retention deletes only validated directory basenames.

A follow-up accepts only the parent turn ID and the new question. Account,
message and draft identity cannot be overridden. A successful native session is
continued with a fork, so branching from retained turns does not mix histories.
Continuation stays on the parent's provider even if the system default changes.
Old terminal-based jobs cannot be continued; start a new chat for them.

Claude runs with non-interactive `dontAsk` permissions: no hidden approval prompt
can leave the panel waiting for input in another window. Permission or login
failures appear in the panel. OmamailAI does not copy the interactive launcher's
auto-approval flags. Existing system configuration and tool permissions still
apply; this bridge is not a sandbox. Treating mail as untrusted context is an AI
instruction, not a technical restriction on its tools. Supplied content goes to
the provider configured for the system AI.

Cancellation and the request deadline stop the worker's child process group.
Tools that detached or submitted work to an existing daemon may continue. Raw
stderr is discarded rather than displayed or persisted, to avoid leaking tool
or login diagnostics. OmamailAI never automatically sends a message or applies AI
text.

## Verification

Backend tests use synthetic Claude and Codex streams and inspect actual process arguments,
stdin and child lifetime. They cover progressive output, native continuation,
absence of terminal launch, failure/limits, safe retention and ownership. QML
tests cover Enter/Shift+Enter submission, history selection while streaming, scroll,
errors, draft insertion and account/context ownership. The fork was checked with Qt 6.11.2 offscreen tests and a 420-pixel-wide QML preview using synthetic data and the repository’s UI stubs. This is not a full test of a personal mailbox in the running shell.

On 2026-09-10, `make validate` passed, including the main QML suite, source regressions, JavaScript/Python tests, QML lint and manifest validation. Targeted final checks covered reply-topic interaction and the 17 synthetic process tests. A live Codex 0.154.0 smoke test used the existing ChatGPT login for two synthetic turns; the forked second turn recalled the first answer. No real mailbox content was used. Claude process behavior was regression-tested with synthetic streams; the upstream documentation reports its own earlier live Claude smoke test.

## Reply topics

For one message, **Reply topics** starts a new contextual request in the existing right dock. Nothing is generated automatically when opening mail. The result contains up to four directions in the language of the message. Click a direction to generate a reply; **Use as reply...** then opens an ordinary reply draft with that text above the existing signature and quotation. It never sends mail. The action checks the original account and selected message again immediately before opening compose; changing either cannot apply the old result to a different message. Existing drafts retain their own Insert/Replace controls. Invalid or incomplete topic output is not actionable.

Topics are for individual messages. Multiple-message assistance and draft commands continue to use the existing free-form conversation. The new controls use the existing English interface; generated topics and replies follow the mail's language.

## Codex adapter

Codex uses `exec --json`, prompts on stdin, and `exec fork` for follow-ups. The JSONL parser imports completed agent messages and fixed status labels only; reasoning, raw commands, MCP results and diagnostics are discarded. A successful `turn.completed` plus nonempty output and a successful process exit are required. Codex may publish text at message boundaries rather than token by token. Native sessions remain in the CLI's own history; forgetting an OmaMail job does not delete that native history.

The adapter requests read-only execution, no approval prompts, disabled shell tool and disabled web search for this invocation. It does not modify saved configuration or claim to sandbox separately configured MCP servers and plugins. As with the existing Claude path, normal agent configuration remains relevant. A missing login or unsupported model is reported in the dock.

Omarchy lazy launcher scripts can install or update software. The bridge detects the mise launcher and uses `mise which` to resolve an already installed executable without invoking `mise use`. Other selected agents, including Kimi, Antigravity and Ori, fail with an explicit unsupported-adapter message; no substitute agent is started. The adapter was verified with installed Codex 0.154.0. Its fork/JSON flags are a compatibility requirement.


## Security review of this extension

**PASS for the changed transport, display and draft-ownership boundaries.** Mail and instructions are JSON-encoded and written to child stdin; they are not interpolated into a shell command. CLI arguments are fixed except for a validated native session UUID and the locally resolved executable. Tests inspect argv/stdin and verify that hostile shell-looking mail text does not create a file. The new Codex parser imports bounded public text, requires a successful completed turn, rejects malformed/incomplete output, and does not persist reasoning or raw tool results. Reply-topic cards use plain text. Draft application requires a completed job and rechecks the account and selected message; tests reject mismatched owners/messages and verify signature/quote preservation.

This is a scoped review, not a security audit of configured agents, MCP servers or plugins. The existing agent tool permissions remain relevant as described above. No additional mail credentials, API keys, automatic provider fallback or automatic sending were introduced.

## Provenance and maintenance

The reply-topic workflow was adapted conceptually from [goarstne/ai-mail-assistant](https://github.com/goarstne/ai-mail-assistant/tree/85a85f3f307a3f35a9376e3beb23c7f58481d180). The QML-native implementation is in `agent/ReplyTopics.js` and `components/AgentPrompt.qml`; no Thunderbird runtime is required. OmaMail retains its original MIT license and copyright notice.

The initial fork is based on OmaMail commit `77db4b19085898ffb58b15fd9ff9b0ae878b36ce`. Keep the original repository as the `upstream` remote and this fork as `origin`. Run `make validate` on Omarchy before publishing behavior changes; use `make test-js test-shell-portable` for portable backend checks. Do not claim the source-only or offscreen checks are live mailbox tests. The inherited release workflow publishes only on a version tag; a normal code push does not publish a release.

## Independent installation (2026-09-10)

OmamailAI is a separate `omamailai` plugin and desktop application. Settings, caches, draft recovery, notification icons, temporary transport files and AI jobs use the fork namespace; keyring compatibility probes also stay within that namespace. The installer never migrates `omamail` or `gmail.omarchy`, and only an explicit `--claim-default` changes the default mail handler. Original accounts are not imported automatically. HEY and AI CLI authentication remain external shared system configuration.

**PASS — scoped coexistence review:** registration and installer tests preserve original plugin folders, the original desktop entry and the default mail handler; config tests write only the fork directory. Credential attribute tests cover the separate keyring namespace. No new mail transport, HTML rendering or agent permissions are introduced by this rename. Live account authentication in the new namespace is **NOT VERIFIED**; no credentials are copied for this check.
