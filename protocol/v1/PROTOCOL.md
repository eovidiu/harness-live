# Harness Live protocol, version 1 (draft 2)

Status: draft for review. Not implemented.

Draft 2 includes the results of a check against real session history:
185 lead sessions, 4,013 subagent sessions and 364,857 Live envelopes.
Section 10 lists the changes and the evidence.

This protocol connects an omp session (the client) to the Harness Live server.
The harness extension in each session is the client. It connects to the server.
The server never searches for sessions.

## 1. Transport

- Unix domain socket: `~/.config/vv-omp-harness/live.sock`, file mode `0600`.
  The parent directory has mode `0700`.
- One connection for each session. A lead session, each subagent and each advisor have their own connection.
- Encoding: UTF-8, one JSON object on each line (NDJSON). No line breaks inside an object.
- Maximum line length: 4 MiB. The receiver closes the connection if a line is longer.
- Each message has a field `t` (the message type).
- A receiver ignores fields that it does not know. A receiver ignores message types that it does not know.
- The server accepts a minimum of 512 connections at the same time.
  One omp process can hold more than 100 connections.

Only a session with a harness root (a `.harness` directory) connects.
A session with no harness root does not connect.

## 2. Connection sequence

```
client                                server
  | ---- connect ---------------------> |
  | ---- hello -----------------------> |
  | <--- welcome  (or reject, close) -- |
  | ---- status ----------------------> |   first status, always
  | ---- view ------------------------> |   first view, lead and provider only
  | ---- event / status / view -------> |
  | <--- cmd -------------------------- |
  | ---- cmd_result ------------------> |
  | ---- bye  (optional) -------------> |
  | ---- close -----------------------> |
```

- The client sends `hello` first. The server answers with `welcome` or `reject`.
- After `welcome`, the client sends one `status`. A lead session and a provider also send one `view`.
- A subagent and an advisor send no `view`. The view of their lead is the state of the project.
- A closed socket means that the session ended. A close with no `bye` is an unclean end.
- The server does not send a heartbeat. The operating system closes the socket when the process stops.

## 3. Messages from the client

### 3.1 `hello`

```json
{
  "t": "hello",
  "protocols": [1],
  "session": {
    "id": "01a1c3f9-42ec-7360-98c8-1060c6215f33",
    "lead": "01a1c3f9-42ec-7360-98c8-1060c6215f33",
    "role": "lead",
    "agent": "lead",
    "parent": null,
    "file": "/sessions/acme-api/2026-11-22_01a1c3f9.jsonl"
  },
  "project": {
    "key": "9f2c4e1ab07d3355",
    "common_key": "41c0d2a97be85f10",
    "root": "/work/acme-api",
    "name": "acme-api"
  },
  "cwd": "/work/acme-api",
  "harness": {
    "version": "2.45.0",
    "build": "2.45.0+9c37708a1b2c",
    "provider": ["bun", "/plugins/vv-omp-harness/bin/provider.ts"]
  },
  "omp": { "version": "19.1.0", "mode": "tui" },
  "pid": 48211,
  "started_at": 1795340000000,
  "verbs": ["session.prompt", "session.abort", "session.shutdown", "agent.cancel", "view.refresh", "ledger.sync"]
}
```

| Field | Rule |
|---|---|
| `protocols` | The protocol versions that the client supports. |
| `session.id` | The id of this session. |
| `session.lead` | The id of the lead session. A subagent gives the id of its lead. |
| `session.role` | `lead`, `subagent`, `advisor` or `provider`. |
| `session.agent` | `lead`, or the task name of the subagent. |
| `session.parent` | The agent that started this one. `null` for a lead. A subagent can start subagents. |
| `session.file` | The path of the session file, or `null`. |
| `project.key` | First 16 hexadecimal characters of SHA-256 of the real path of the harness root. Different for each worktree. |
| `project.common_key` | The same hash of the git common directory. Equal for all worktrees of one repository. The server uses it to group worktrees that share one ledger. |
| `harness.build` | Version and commit, or `null`. |
| `harness.provider` | The command that builds one view for this project. The server stores it and uses it when no session runs. |
| `omp.version` | The omp version, or `null` if the session cannot read it. |
| `omp.mode` | `tui`, `rpc`, `json` or `print`. |
| `pid` | The process id. The server compares it with the peer id from the operating system. A lead and its subagents can have the same `pid`. |
| `verbs` | The commands that this session accepts. The server sends no other verb. |

### 3.2 `status`

The complete state of one session. Each `status` replaces the previous one.
It is small. The client sends it when a value changes.

```json
{
  "t": "status",
  "ts": 1795340001000,
  "rev": 7,
  "idle": false,
  "model": "claude-fable-5-2",
  "waiting_on": {
    "kind": "ask",
    "call": "toolu_01AbC",
    "since": 1795340000900,
    "feature": "F012",
    "topic": "review",
    "fields": 1
  }
}
```

| Field | Rule |
|---|---|
| `rev` | Increases by 1 for each status on this connection. |
| `idle` | `true` when the agent waits for a user prompt. The server uses it to show the session state. |
| `waiting_on` | `null`, or a short record of the open question that a person must answer. |
| `waiting_on.kind` | `ask` in version 1. |
| `waiting_on.call` | The id of the `ask` tool call. |
| `waiting_on.since` | The time when the question opened. |
| `waiting_on.topic` | The harness prefix of the question, or `null`: `review`, `ops`, `spec`, `flow_approval`, `flow_move`, `preflight`, `harness-config`, and others. |
| `waiting_on.feature` | The feature id, or `null`. |
| `waiting_on.fields` | The number of fields in the question. |

Version 1 does not send the content of a question and has no verb to answer one.
The server shows that a session waits. The person answers:

- in the terminal, for a session that the person started;
- on the tracker issue, for a session that the server started. The harness posts the question as a comment on the issue and reads the reply there. The server takes no part in this.

### 3.3 `view`

The complete state of one project. Each `view` replaces the previous one.
Only a lead session and a provider send it.

```json
{
  "t": "view",
  "ts": 1795340000500,
  "rev": 12,
  "view": {
    "generated_at": 1795340000480,
    "gates": "enforce",
    "ledger": {
      "backend": "linear", "container": "Linear project vv-harness", "label": "vv-harness",
      "fetched_at": 1795339990000, "generation": 41, "error": null
    },
    "flow": {
      "approved": true,
      "default": "standard",
      "by_scope": [ { "glob": "src/live/ui/**", "flow": "web" } ],
      "flows": { "standard": ["spec", "implement", "focused_test", "review", "impact_test"] },
      "steps": {
        "spec": { "kind": "agent", "agent": "harness-spec-verification", "description": "the spec is complete and testable" },
        "review": { "kind": "panel", "agent": "harness-reviewer", "description": "every lens votes APPROVE on the same code", "max_rounds": 5 }
      }
    },
    "features": [
      {
        "id": "F012",
        "title": "Redact tokens in tool output",
        "description": "Tokens in tool output must not reach the log.",
        "status": "in_progress",
        "priority": 2,
        "url": "https://linear.app/example/issue/OVI-260",
        "flow": "standard",
        "scope": ["src/live/redact.ts"],
        "depends_on": [],
        "split_of": null,
        "failure_reason": null,
        "spec": { "verdict": "PASS", "verified_at": 1795330000000 },
        "claimed_by": { "session": "01a1c3f9-42ec-7360-98c8-1060c6215f33", "branch": "f012-redact" },
        "progress": [
          { "step": "spec", "state": "done", "at": 1795330000000 },
          { "step": "implement", "state": "done", "at": 1795335000000 },
          { "step": "focused_test", "state": "running" },
          { "step": "review", "state": "pending", "origin": { "kind": "rule", "rule": "risk_paths", "cause": "paths" } },
          { "step": "impact_test", "state": "pending" }
        ],
        "waiting": null,
        "panel": { "round": 2, "max_rounds": 5, "lenses": [ { "lens": "correctness", "vote": "APPROVE" }, { "lens": "tests", "vote": "CHANGES" } ], "risk": "high" }
      }
    ],
    "decisions": [],
    "drift": [],
    "errors": []
  }
}
```

| Field | Rule |
|---|---|
| `rev` | Increases by 1 for each view on this connection. |
| `view.flow.approved` | `false` when the flow file changed and a person did not approve it. |
| `view.features[].status` | `pending`, `in_progress`, `blocked`, `passing`, `failed`, `removed`. The harness maps its ledger values to these. |
| `view.features[].description` | A maximum of 4,000 characters. |
| `view.features[].progress[].state` | `pending`, `running`, `done`, `failed`, `waiting`. |
| `view.features[].progress[].override` | Optional. Present when a person or a rule replaced the usual result of the step. |
| `view.features[].progress[].origin` | Optional. Present when a rule or the lead added the step. |
| `view.features[].waiting` | `null`, or the text of what a person must settle for this feature. |
| `view.features[].panel` | `null`, or the state of the review panel. |
| `view.decisions` | The latest decision-model results that affect the state. |
| `view.errors` | Text for each part of the state that the harness could not read. |

All text passes through the harness `redact` function before the client sends it.

The harness sends a `view`:

- after `welcome`;
- after a change of the flow, the ledger, the evidence or the panel state, also when a different process made the change;
- after the command `view.refresh`.

The client sends a maximum of one `view` each second. A newer view replaces a view that waits.

### 3.4 `event`

One activity record. Events can be lost. State must not depend on events.

```json
{
  "t": "event",
  "seq": 418,
  "ts": 1795340003120,
  "kind": "harness.gate",
  "feature": "F012",
  "dropped": 0,
  "data": { "gate": "scope", "blocked": true, "reason": "path is outside the feature scope", "call": "toolu_01" }
}
```

| Field | Rule |
|---|---|
| `seq` | Starts at 0 on each connection and increases by 1. |
| `kind` | One of the kinds below. |
| `feature` | The feature id, if the event belongs to one feature. |
| `dropped` | The number of events that the client dropped before this one. |

Kinds in version 1:

| Group | Kinds |
|---|---|
| Agent | `agent.spawn`, `agent.progress`, `agent.end`, `agent.brief` |
| Turn | `turn.start`, `turn.end` |
| Tool | `tool.start`, `tool.output`, `tool.end` |
| Text | `text.delta` |
| User | `user.prompt` |
| Session | `session.compaction` |
| Harness | `harness.gate`, `harness.test`, `harness.ledger`, `harness.decision`, `harness.panel` |

- The `data` object of each current kind is the same as in the current Live envelope
  (`src/live/protocol.ts` in `vv-omp-harness`).
- `user.prompt` has `length` and `steering`. It does not contain the text.
- `session.compaction` has no required data.
- The client makes `harness.decision` and `harness.panel`. The server made them before, from project files.
- The fields `assignment` and `context` of `agent.spawn` have a maximum of 8 KiB each.
- The client combines fast `text.delta`, `tool.output` and `agent.progress` updates, as the current tap does.

These current kinds are not in version 1:

- `session.start`, `session.end`: replaced by `hello`, `bye` and the socket state.
- `harness.features`, `harness.flow`, `harness.flow_steps`, `harness.flow_unreadable`: replaced by `view`.
- `lead.idle`: replaced by `status.idle`.
- `stall`, `stall.clear`: the server makes them from the events and the status. It needs no project files.

### 3.5 `cmd_result`

```json
{ "t": "cmd_result", "id": "c-7f3a", "ok": true, "data": { "delivered_as": "steer" } }
{ "t": "cmd_result", "id": "c-7f3b", "ok": false, "error": { "code": "refused", "message": "a question is open" } }
```

Error codes: `unknown_verb`, `bad_args`, `refused`, `failed`, `timeout`.

### 3.6 `bye`

```json
{ "t": "bye", "reason": "signal", "signal": "SIGTERM", "pending_calls": 2 }
```

| Field | Rule |
|---|---|
| `reason` | `session_shutdown`, `provider_done`, `signal`, `fatal`, `error`. |
| `signal` | The signal name, for the reason `signal`. |
| `pending_calls` | The number of tool calls that did not end. |

## 4. Messages from the server

### 4.1 `welcome`

```json
{ "t": "welcome", "protocol": 1, "server": { "version": "0.1.0" } }
```

`protocol` is the highest version that the client and the server support.

### 4.2 `reject`

```json
{ "t": "reject", "code": "no_common_protocol", "message": "the server supports 2 to 3" }
```

Codes: `no_common_protocol`, `bad_hello`, `peer_mismatch`, `duplicate_session`.
The server closes the connection after `reject`. The client does not try again in this session.

Rule for a `session.id` that the server already knows:

- The old connection is closed: the server accepts the new one. This is a resumed session.
- The old connection is open and has the same `pid`: the new connection replaces the old one.
- The old connection is open and has a different `pid`: the server sends `reject` with `duplicate_session`.

### 4.3 `cmd`

```json
{ "t": "cmd", "id": "c-7f3a", "verb": "session.prompt", "args": { "text": "Continue with F012.", "deliver_as": "auto" }, "deadline_ms": 10000 }
```

- `id` is unique for each command. The client gives the same `id` in `cmd_result`.
- The client answers each `cmd` with one `cmd_result`.
- If no `cmd_result` arrives before `deadline_ms`, the server records `timeout`.

Verbs in version 1:

| Verb | Arguments | Effect in the session | Accepted by |
|---|---|---|---|
| `session.prompt` | `text`, `deliver_as` | Adds a user message. The gates apply as usual. | `tui` lead sessions |
| `session.abort` | none | Stops the current agent operation. | `tui` lead sessions |
| `session.shutdown` | none | Stops the session. | `tui` lead sessions |
| `agent.cancel` | `agent` | Stops one subagent. | lead sessions |
| `view.refresh` | none | Builds and sends a new `view`. | lead sessions, providers |
| `ledger.sync` | none | Syncs the ledger with the tracker, then sends a new `view`. | lead sessions |

- `deliver_as`: `auto`, `steer` or `follow_up`. With `auto`, omp starts a prompt if the agent is idle and queues a steer if the agent is busy. The result gives `delivered_as`.
- While `status.waiting_on` is set, the client answers `session.prompt` with `refused`. A message that arrives during an open question closes the question as an error in omp.
- A session that the server started in RPC mode (`rpc-ui`) gets prompt, steer, abort and stop through RPC on stdio. It does not list the `session.*` verbs in `hello`.

## 5. Rules for the client

1. The client never blocks an omp hook. It writes to a queue in memory.
2. The queue for each connection has a limit of 2,500 events or 2 MiB. When it is full, the client drops the oldest `event` and counts it in `dropped`.
3. The client never drops a `view` or a `status`. A newer one replaces one that waits in the queue.
4. If the connection fails, the client drops events. It tries to connect again on a timer: after 1 second, then with a delay that doubles to a maximum of 30 seconds.
5. After each new connection, the client sends `hello`, then a full `status`, then a full `view` (lead and provider).
6. The client runs only the verbs that it listed in `hello`.

## 6. Rules for the server

1. The server accepts connections only from the same operating system user.
2. The server compares `hello.pid` with the peer process id, where the operating system gives one.
3. The server keeps the latest `view` for each project, the latest `status` for each session, and the events for each session.
4. When a socket closes, the server marks the session as ended. With no `bye` before the close, it marks an unclean end.
5. The server sends a `cmd` only for a verb in the `verbs` list of that session.
6. The server reads no project files and calls no tracker.

## 7. Provider command

For a project with no session, the server starts the command from `harness.provider`
with the project root as the working directory. The provider:

1. connects and sends `hello` with `session.role` = `provider`;
2. sends one `status` and one `view`;
3. sends `bye` with reason `provider_done` and closes.

## 8. Version rules

- Version 1 accepts only additions: new optional fields, new event kinds, new verbs.
- A change that breaks an old client or an old server starts version 2.
- The server supports the current version and the previous version.

## 9. Open points

1. The field list of `view.features[].panel` and `view.decisions`. This draft shows the minimum.
2. Raw hook rates are not known. The queue limit uses rates after the tap combined updates.
3. For a session that the server started: the design of "question as a comment on the issue, reply on the issue". This is harness work, not protocol work.

Tested on omp 19.1.0:

- An extension can read the omp version (`pi.pi.VERSION`), the mode (`ctx.mode`) and the idle state (`ctx.isIdle()`).
- `sendUserMessage` starts a prompt when the agent is idle and queues a steer when it is busy. Text that a person typed stays in the input box.
- `sendUserMessage` during an open question closes the question as an error.
- The `ask` tool exists only in `tui` and `rpc-ui` modes. In omp, `ctx.mode` gives `rpc` for both RPC modes.
- An extension can answer an open question, but only through omp internals. Version 1 does not use this.

## 10. Changes in draft 2, with evidence

| Change | Evidence from the history |
|---|---|
| New message `status` with `idle` and a short `waiting_on` record | 601 `ask` calls, all from lead sessions. Median wait 308 seconds, maximum 25 hours. `lead.idle` occurred 17 times against 37,769 `turn.end`. |
| New verb `agent.cancel` | 270 subagent messages ended with `aborted`. No verb could address one subagent. |
| `deliver_as` on `session.prompt`; error `busy` removed | 129 prompts arrived while the agent was busy. omp queues them as steers. |
| Client makes `harness.decision` and `harness.panel`; panel state in `view` | The server made 716 and 361 of these from project files. Rule 6.6 forbids that. |
| Status values `failed` and `removed`; `canceled` removed | Ledgers: 10 `failed`, 1 `removed`, 0 `canceled`. |
| More feature and flow fields in `view` | 176 of 265 flow-step records had fields with no place in draft 1. |
| Kinds `user.prompt` and `session.compaction` | 1,718 lead user messages, 104 steers, 1,751 compactions. |
| `stall` made by the server | 1,221 `stall` envelopes. The rules need only events and status. |
| Reconnect on a timer | Median prompt cycle 162 seconds; maximum active time 105 hours. |
| Queue limit 2,500 events or 2 MiB | Peak 2,420 events in 60 seconds for one lead group. 99th percentile event size 552 bytes. |
| `hello`: `json` mode, `cwd`, `session.file`, `harness.build`, role `advisor`, `project.common_key` | omp has a `json` mode. 50 advisor sessions. 19 lead sessions ran in worktrees. |
| Sessions with no harness root do not connect | 67 of 185 lead sessions had no harness. |
| `bye` reasons `signal` and `fatal`; `pending_calls` | 46 runs ended by a signal, 2 by a fatal error, 26 with no exit record. |
| Rule for a known `session.id` | 185 lead sessions had 341 process runs with the same id. |
| Minimum of 512 connections | 134 sessions wrote in the same minute. |

Decided after the check:

- Only a lead session and a provider send a `view`.
- No verb to answer a question in version 1, and no question content on the wire. The server shows only that a session waits. Reasons: on omp 19.1.0 each question is an HTML form; an answer from an extension needs omp internals; the person is at the same computer as the terminal.
- A session that the server started gets its answers on the tracker issue, not through the server.

Not accepted from the check:

- A partial view update (`view.patch`). The largest view was 66 KB and the highest rate was 156 changes each hour. That is approximately 10 MB each hour on a local socket. The limit of one view each second is sufficient for version 1.
- A kind for model changes. `agent.progress` and `status` already give the model.
