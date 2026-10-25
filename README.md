# Task Execution Protocol

## Core Assumptions

- There are execution units (task), each task is a sandboxed stateful environment.
- Each task knows how to resume from a given `session_id`, `task_id`, and `seq_id`.
- Each task reports its aliveness status through some probe.
- Each task can call one or more other tasks (`CallMany`). Execution halts on the caller task until all called tasks finish. The caller task is resumed automatically when they are done (including if any callee fails).
- All tasks in a call tree share the same `session_id`.
- The runtime automatically suspends and resumes the underlying sandbox.

---

## 1. Core Execution Model

- **Session (`session_id`)**: Identifies an entire call tree. It is assigned once at the root invocation (by the external entrypoint) and propagated unchanged to every task the tree spawns. A task never chooses or changes its `session_id`.
- **Task (`task_id`)**: Identifies one stateful sandbox within a session. A `task_id` maps 1:1 to a sandbox and is unique within its `session_id`. The caller always passes the `task_id` when calling a task, so it can either name a fresh sandbox or re-enter an existing one.
- **Sequence (`seq_id`)**: A monotonically increasing integer (`0, 1, 2, ...`) tracking checkpoints within a `task_id`. Owned by the runtime.
- **Sandbox-Native State**: The physical sandbox snapshot (memory + filesystem/disk state) is the source of truth for `(session_id, task_id, seq_id)`.
  - Suspending a task at `(session_id, task_id, seq_id)` snapshots/pauses its sandbox at that exact boundary.
  - Resuming from `(session_id, task_id, seq_id)` restores that exact sandbox snapshot and delivers the resume payload into the waking task.
  - If a task crashes or fails liveness probes while executing from `(session_id, task_id, N)`, the runtime discards the dirty sandbox, restores the snapshot at `(session_id, task_id, N)`, and re-delivers the input for step `N`.
- **Single-Writer & Cycle Prevention**: A `task_id` can only have one active execution turn at a time. Calling a `task_id` that is already `RUNNING` or `WAITING` (e.g., duplicate target in `CallMany` or a call cycle) immediately resolves that call entry with a `TASK_BUSY` error without mutating the target sandbox.
- **Session Isolation**: A task may only call `task_id`s within its own `session_id`. Cross-session calls are rejected with `SESSION_MISMATCH`.

---

## 2. Task & Sandbox Lifecycle

| Logical Task State | Physical Sandbox State | Description |
| :--- | :--- | :--- |
| **`RUNNING`** | **Active** (CPU/mem allocated, probed) | Task is actively executing a turn starting from `(session_id, task_id, seq_id)`. |
| **`WAITING`** | **Suspended** (snapshotted / paused) | Task emitted `YIELD_CALL`. Runtime snapshots the sandbox at `seq_id + 1` and suspends it until all child calls finish. |
| **`IDLE` (Completed Turn)** | **Suspended** (snapshotted at `seq_id + 1`) | Task emitted `YIELD_DONE`. Sandbox is suspended at `seq_id + 1` and can be resumed by future calls to the same `task_id`. |
| **`UNHEALTHY`** | **Reclaimed / Rolled Back** | Liveness probe failed or timed out while `RUNNING`. Runtime kills the active sandbox and either retries from `(session_id, task_id, seq_id)` or returns an error to the caller. |

---

## 3. Protocol Primitives (Task <-> Runtime)

### 3.1 `RESUME` *(Runtime -> Task)*

Restores the sandbox for `(session_id, task_id, seq_id)` (or boots at `seq_id = 0` if the task is new) and delivers either the caller's `input` or the `call_results` from a completed `YIELD_CALL`:

```json
{
  "type": "RESUME",
  "session_id": "sess_1",
  "task_id": "task_a",
  "seq_id": 1,
  "input": null,
  "call_results": [
    {
      "task_id": "task_b",
      "output": "<payload>",
      "error": null
    },
    {
      "task_id": "task_c",
      "output": null,
      "error": {
        "code": "STATUS_TIMEOUT",
        "message": "Task task_c failed liveness checks"
      }
    }
  ]
}
```

- When invoked by a parent task (or external entrypoint), `input` carries the invocation arguments and `call_results` is empty.
- When waking up after `YIELD_CALL`, `call_results` is populated in the exact order of the requested `calls`. Each entry has exactly one of `output` or `error` set, mirroring the callee's `YIELD_DONE` (or a runtime-generated error such as `STATUS_TIMEOUT` or `TASK_BUSY`).

### 3.2 `STATUS` *(Runtime <-> Task)*

The runtime periodically requests a task's status. The response reports the task's lifecycle state:

```json
{
  "type": "STATUS",
  "session_id": "sess_1",
  "task_id": "task_a",
  "seq_id": 1,
  "status": "RUNNING"
}
```

`status` is one of:

| Status | Meaning |
| :--- | :--- |
| `RUNNING` | Sandbox is active and the task is executing a turn from `seq_id`. |
| `SUSPENDED` | Sandbox is snapshotted and paused at `seq_id` (task is `WAITING` or `IDLE`). |
| `FAILED` | Task is `UNHEALTHY`; the sandbox was reclaimed and a rollback to `seq_id` is pending or has been abandoned. |

- **Fencing**: Including `(session_id, task_id, seq_id)` in `STATUS` responses allows the runtime to fence stale/zombie task instances if a turn was already rolled back and resumed elsewhere.
- **Only `RUNNING` sandboxes are queried directly**: A `SUSPENDED` or `FAILED` status is answered by the runtime from its own records; the paused sandbox is never woken to answer a `STATUS` request. A `RUNNING` task that fails to respond within the status timeout is transitioned to `FAILED`.


### 3.3 `YIELD` *(Task -> Runtime)*

A running turn always ends with the task emitting a yield frame, which triggers an immediate sandbox snapshot & suspend. The task does not send a sequence number: the runtime owns `seq_id` and records the new checkpoint as `seq_id + 1`, where `seq_id` is the value it delivered in the preceding `RESUME`.

#### `YIELD_CALL`

Halts the caller task, snapshots and suspends its sandbox, and invokes one or more target tasks in parallel. All targets inherit the caller's `session_id`:

```json
{
  "type": "YIELD_CALL",
  "session_id": "sess_1",
  "task_id": "task_a",
  "calls": [
    { "task_id": "task_b", "task": "worker_type_1", "input": "<args_1>" },
    { "task_id": "task_c", "task": "worker_type_2", "input": "<args_2>" }
  ]
}
```

- If a target `task_id` does not exist yet in the session, the runtime creates its sandbox at `seq_id = 0`.
- If a target `task_id` already exists and is `IDLE`, the runtime resumes its suspended sandbox from its latest `seq_id` with the new `input`.

#### `YIELD_DONE` (Turn Complete)

Completes the current invocation, snapshots and suspends the sandbox, and returns the result to the waiting caller. Exactly one of `output` or `error` is set, and the pair is forwarded verbatim into the caller's `call_results`:

```json
{
  "type": "YIELD_DONE",
  "session_id": "sess_1",
  "task_id": "task_a",
  "output": "<final_result>",
  "error": null
}
```

---

## 4. End-to-End Call Sequence

All frames below carry `session_id = sess_1`; it is omitted for brevity.

```text
task_a                            Runtime                   task_b                 task_c
------                            -------                   ------                 ------
  |<-- RESUME(task_a, seq=0) --------|
  |<-- STATUS() --------------------->|
  |                                  |
  |--- YIELD_CALL(calls=[task_b, -->|
  |     task_c])                     |
[Snapshot & Suspend (task_a, 1)]     |
                                     |--- RESUME(task_b, latest_seq, in) ->|
                                     |--- RESUME(task_c, latest_seq, in) ----------------------->|
                                     |<-- STATUS() ------------------------>|                     |
                                     |<-- STATUS() ---------------------------------------------->X (timeout!)
                                     |                                     |       [Rollback/Kill task_c]
                                     |<-- YIELD_DONE(task_b, out) ---------|
                                     |                       [Snapshot & Suspend (task_b)]
                                     |
[Restore & Unsuspend (task_a, 1)]    |
  |<-- RESUME(task_a, seq=1, --------|
  |     results=[                    |
  |       {task_b: out},             |
  |       {task_c: err}              |
  |     ])                           |
  |                                  |
  | (task_a handles task_c error)    |
  |--- YIELD_DONE(out) ------------->|
[Snapshot & Suspend (task_a, 2)]
```

### 4.1 The Same Sequence as HTTP Calls

The protocol is transport-agnostic. For illustration, assume this HTTP binding:

| Direction | Frame | Endpoint |
| :--- | :--- | :--- |
| Client -> Runtime | start root task | `POST /sessions/{session_id}/tasks/{task_id}/call` |
| Runtime -> Task | `RESUME` | `POST http://<sandbox>/resume` |
| Runtime -> Task | `STATUS` | `GET http://<sandbox>/healthz` |
| Task -> Runtime | `YIELD_CALL` / `YIELD_DONE` | `POST /sessions/{session_id}/tasks/{task_id}/yield` |
| Client -> Runtime | inspect status | `GET /sessions/{session_id}/tasks/{task_id}` |

`$RUNTIME` is the runtime's base URL. `$TASK_A`, `$TASK_B`, `$TASK_C` are the sandbox-local addresses the runtime uses to reach each task; tasks never call each other directly.

**1. A client starts the root task.** This creates `sess_1` and `task_a` at `seq_id = 0`.

```sh
curl -X POST $RUNTIME/sessions/sess_1/tasks/task_a/call \
  -H 'Content-Type: application/json' \
  -d '{ "task": "planner", "input": { "goal": "summarize repo" } }'
```

**2. Runtime boots `task_a`'s sandbox and delivers `RESUME`.**

```sh
curl -X POST $TASK_A/resume \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "RESUME",
    "session_id": "sess_1",
    "task_id": "task_a",
    "seq_id": 0,
    "input": { "goal": "summarize repo" },
    "call_results": []
  }'
```

**3. Runtime requests `task_a`'s status while it runs.**

```sh
curl $TASK_A/healthz
# {"type":"STATUS","session_id":"sess_1","task_id":"task_a","seq_id":0,"status":"RUNNING"}
```

**4. `task_a` fans out to `task_b` and `task_c`.** The runtime snapshots `task_a` at `seq_id = 1` and suspends it.

```sh
curl -X POST $RUNTIME/sessions/sess_1/tasks/task_a/yield \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "YIELD_CALL",
    "session_id": "sess_1",
    "task_id": "task_a",
    "calls": [
      { "task_id": "task_b", "task": "reader", "input": { "path": "src/" } },
      { "task_id": "task_c", "task": "reader", "input": { "path": "docs/" } }
    ]
  }'
```

**5. Runtime boots both children and delivers `RESUME` to each.**

```sh
curl -X POST $TASK_B/resume \
  -H 'Content-Type: application/json' \
  -d '{ "type": "RESUME", "session_id": "sess_1", "task_id": "task_b", "seq_id": 0,
        "input": { "path": "src/" }, "call_results": [] }'

curl -X POST $TASK_C/resume \
  -H 'Content-Type: application/json' \
  -d '{ "type": "RESUME", "session_id": "sess_1", "task_id": "task_c", "seq_id": 0,
        "input": { "path": "docs/" }, "call_results": [] }'
```

**6. Runtime requests the children's status.** `task_b` answers; `task_c` times out and is marked `FAILED`.

```sh
curl $TASK_B/healthz
# {"type":"STATUS","session_id":"sess_1","task_id":"task_b","seq_id":0,"status":"RUNNING"}

curl --max-time 5 $TASK_C/healthz
# curl: (28) Operation timed out
```

**7. `task_b` completes.** The runtime snapshots `task_b` at `seq_id = 1` and suspends it.

```sh
curl -X POST $RUNTIME/sessions/sess_1/tasks/task_b/yield \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "YIELD_DONE",
    "session_id": "sess_1",
    "task_id": "task_b",
    "output": { "files": 42 },
    "error": null
  }'
```

**8. A client inspecting the session at this point sees the collapsed statuses.** The runtime answers these from its own records without waking any sandbox.

```sh
curl $RUNTIME/sessions/sess_1/tasks/task_a
# {"session_id":"sess_1","task_id":"task_a","seq_id":1,"status":"SUSPENDED"}

curl $RUNTIME/sessions/sess_1/tasks/task_c
# {"session_id":"sess_1","task_id":"task_c","seq_id":0,"status":"FAILED"}
```

**9. All children are settled, so the runtime restores `task_a` at `seq_id = 1` and resumes it with the results.** The `call_results` array is in the same order as the `calls` array from step 4.

```sh
curl -X POST $TASK_A/resume \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "RESUME",
    "session_id": "sess_1",
    "task_id": "task_a",
    "seq_id": 1,
    "input": null,
    "call_results": [
      { "task_id": "task_b", "output": { "files": 42 }, "error": null },
      { "task_id": "task_c", "output": null,
        "error": { "code": "STATUS_TIMEOUT", "message": "Task task_c failed liveness checks" } }
    ]
  }'
```

**10. `task_a` handles the partial failure and completes.** The runtime snapshots `task_a` at `seq_id = 2`, suspends it, and returns the output to the original client call from step 1.

```sh
curl -X POST $RUNTIME/sessions/sess_1/tasks/task_a/yield \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "YIELD_DONE",
    "session_id": "sess_1",
    "task_id": "task_a",
    "output": { "summary": "42 source files; docs unavailable" },
    "error": null
  }'
```

**11. Re-entering an existing task.** Because `task_a` is now `SUSPENDED` at `seq_id = 2`, a later call with the same `session_id` and `task_id` resumes that snapshot rather than starting fresh:

```sh
curl -X POST $RUNTIME/sessions/sess_1/tasks/task_a/call \
  -H 'Content-Type: application/json' \
  -d '{ "task": "planner", "input": { "goal": "now include tests/" } }'
# runtime -> task_a: RESUME with seq_id = 2 and the new input
```
