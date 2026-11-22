# Harness Live protocol, version 1 (draft)

Status: draft for review. Not implemented.

This protocol connects an omp session (the client) to the Harness Live server.
The harness extension in each session is the client. It connects to the server.
The server never searches for sessions.

## 1. Transport

- Unix domain socket: `~/.config/vv-omp-harness/live.sock`, file mode `0600`.
  The parent directory has mode `0700`.
- One connection for each session. A lead session and each subagent have their own connection.
- Encoding: UTF-8, one JSON object on each line (NDJSON). No line breaks inside an object.
- Maximum line length: 4 MiB. The receiver closes the connection if a line is longer.
- Each message has a field `t` (the message type).
- A receiver ignores fields that it does not know. A receiver ignores message types that it does not know.

## 2. Connection sequence

```
client                                server
  | ---- connect ---------------------> |
  | ---- hello -----------------------> |
  | <--- welcome  (or reject, close) -- |
  | ---- view ------------------------> |   first view, always
  | ---- event / view  (many) --------> |
  | <--- cmd -------------------------- |
  | ---- cmd_result ------------------> |
  | ---- bye  (optional) -------------> |
  | ---- close -----------------------> |
```

- The client sends `hello` first. The server answers with `welcome` or `reject`.
- After `welcome`, the client sends one `view` before other messages.
- A closed socket means that the session ended. `bye` only gives the reason.
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
    "parent": null
  },
  "project": {
    "key": "9f2c4e1ab07d3355",
    "root": "/work/acme-api",
    "name": "acme-api"
  },
  "harness": { "version": "2.45.0", "provider": ["bun", "/plugins/vv-omp-harness/bin/provider.ts"] },
  "omp": { "version": "19.1.0", "mode": "tui" },
  "pid": 48211,
  "started_at": 1795340000000,
  "verbs": ["session.prompt", "session.abort", "session.shutdown", "view.refresh", "ledger.sync"]
}
```

| Field | Rule |
|---|---|
| `protocols` | The protocol versions that the client supports. |
| `session.id` | The id of this session. Unique for each connection. |
| `session.lead` | The id of the lead session. A subagent gives the id of its lead. |
| `session.role` | `lead`, `subagent` or `provider`. |
| `session.agent` | `lead`, or the task name of the subagent. |
| `project.key` | First 16 hexadecimal characters of SHA-256 of the real path of the root. |
| `harness.provider` | The command that builds one view for this project. The server stores it and uses it when no session runs. |
| `omp.mode` | `tui`, `rpc` or `print`. |
| `pid` | The process id. The server compares it with the peer id from the operating system. |
| `verbs` | The commands that this session accepts. The server sends no other verb. |

### 3.2 `view`

The complete state of one project. Each `view` replaces the previous one.

```json
{
  "t": "view",
  "ts": 1795340000500,
  "rev": 12,
  "view": {
    "generated_at": 1795340000480,
    "gates": "enforce",
    "ledger": { "backend": "linear", "container": "Linear project vv-harness", "fetched_at": 1795339990000, "error": null },
    "flows": {
      "standard": ["spec", "implement", "focused_test", "review", "impact_test"]
    },
    "steps": {
      "spec": { "kind": "agent", "description": "the spec is complete and testable" },
      "review": { "kind": "panel", "description": "every lens votes APPROVE on the same code" }
    },
    "features": [
      {
        "id": "F012",
        "title": "Redact tokens in tool output",
        "status": "in_progress",
        "url": "https://linear.app/example/issue/OVI-260",
        "flow": "standard",
        "scope": ["src/live/redact.ts"],
        "depends_on": [],
        "claimed_by": { "session": "01a1c3f9-42ec-7360-98c8-1060c6215f33", "branch": "f012-redact" },
        "progress": [
          { "step": "spec", "state": "done" },
          { "step": "implement", "state": "done" },
          { "step": "focused_test", "state": "running" },
          { "step": "review", "state": "pending" },
          { "step": "impact_test", "state": "pending" }
        ],
        "waiting_on": null
      }
    ],
    "drift": [],
    "errors": []
  }
}
```

| Field | Rule |
|---|---|
| `rev` | Increases by 1 for each view on this connection. |
| `view.features[].status` | `pending`, `in_progress`, `blocked`, `passing`, `canceled`. |
| `view.features[].progress[].state` | `pending`, `running`, `done`, `failed`, `waiting`. |
| `view.features[].waiting_on` | `null`, or an object that describes the decision that a person must make. |
| `view.errors` | Text for each part of the state that the harness could not read. |

All text passes through the harness `redact` function before the client sends it.

The harness sends a `view`:

- after `welcome`;
- after a change of the flow, the ledger or the evidence;
- after the command `view.refresh`.

### 3.3 `event`

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
| Turn | `turn.start`, `turn.end`, `lead.idle` |
| Tool | `tool.start`, `tool.output`, `tool.end` |
| Text | `text.delta` |
| Harness | `harness.gate`, `harness.test`, `harness.ledger`, `harness.decision`, `harness.panel` |

The `data` object of each kind is the same as in the current Live envelope
(`src/live/protocol.ts` in `vv-omp-harness`). These current kinds are not in version 1:

- `session.start`, `session.end`: replaced by `hello`, `bye` and the socket state.
- `harness.features`, `harness.flow`, `harness.flow_steps`, `harness.flow_unreadable`: replaced by `view`.
- `stall.clear`: the server makes it.

### 3.4 `cmd_result`

```json
{ "t": "cmd_result", "id": "c-7f3a", "ok": true, "data": {} }
{ "t": "cmd_result", "id": "c-7f3b", "ok": false, "error": { "code": "busy", "message": "the agent is not idle" } }
```

Error codes: `unknown_verb`, `bad_args`, `busy`, `refused`, `failed`, `timeout`.

### 3.5 `bye`

```json
{ "t": "bye", "reason": "session_shutdown" }
```

Reasons: `session_shutdown`, `provider_done`, `error`.

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

### 4.3 `cmd`

```json
{ "t": "cmd", "id": "c-7f3a", "verb": "session.prompt", "args": { "text": "Continue with F012." }, "deadline_ms": 10000 }
```

- `id` is unique for each command. The client gives the same `id` in `cmd_result`.
- The client answers each `cmd` with one `cmd_result`.
- If no `cmd_result` arrives before `deadline_ms`, the server records `timeout`.

Verbs in version 1:

| Verb | Arguments | Effect in the session | Accepted by |
|---|---|---|---|
| `session.prompt` | `text` | Adds a user message. The gates apply as usual. | `tui` sessions |
| `session.abort` | none | Stops the current agent operation. | `tui` sessions |
| `session.shutdown` | none | Stops the session. | `tui` sessions |
| `view.refresh` | none | Builds and sends a new `view`. | all sessions |
| `ledger.sync` | none | Syncs the ledger with the tracker, then sends a new `view`. | lead sessions |

A session that the server started in RPC mode gets prompt, steer, abort and stop through RPC on stdio.
It does not list the `session.*` verbs in `hello`.

## 5. Rules for the client

1. The client never blocks an omp hook. It writes to a queue in memory.
2. The queue has a size limit. When it is full, the client drops the oldest `event` and counts it in `dropped`.
3. The client never drops a `view`. A new `view` replaces a `view` that is still in the queue.
4. If the connection fails, the client drops events. It tries to connect again before the next prompt.
5. After each new connection, the client sends `hello`, then a full `view`.
6. The client runs only the verbs that it listed in `hello`.

## 6. Rules for the server

1. The server accepts connections only from the same operating system user.
2. The server compares `hello.pid` with the peer process id, where the operating system gives one.
3. The server keeps the latest `view` for each project and the events for each session.
4. When a socket closes, the server marks the session as ended.
5. The server sends a `cmd` only for a verb in the `verbs` list of that session.
6. The server reads no project files and calls no tracker.

## 7. Provider command

For a project with no session, the server starts the command from `harness.provider`
with the project root as the working directory. The provider:

1. connects and sends `hello` with `session.role` = `provider`;
2. sends one `view`;
3. sends `bye` with reason `provider_done` and closes.

## 8. Version rules

- Version 1 accepts only additions: new optional fields, new event kinds, new verbs.
- A change that breaks an old client or an old server starts version 2.
- The server supports the current version and the previous version.

## 9. Open points

1. The full field list of `view.features[]`. This draft shows the minimum.
2. The shape of `waiting_on` (a decision that a person must make).
3. The size limit of the client queue.
4. A `session.steer` verb for `tui` sessions.
