# Task Execution Protocol

## Core Assumptions

- There are execution units (task), each task is a sandboxed stateful environment.
- Each task knows how to resume from a given `session_id` and a `seq_id`.
- Each task reports its aliveness status through some probe.
- Each task can call one or more other tasks (`CallMany`). Execution halts on the caller task until all called tasks finish. The caller task is resumed automatically when they are done (including if any callee fails).
- The runtime automatically suspends and resumes the underlying sandbox.

---

## 1. Core Model & State Invariants

- **Session (`session_id`)**: Every task execution belongs to an explicit `session_id` (always passed by the caller or external entrypoint). A `session_id` maps 1:1 to a stateful sandbox.
- **Sequence (`seq_id`)**: A monotonically increasing integer (`0, 1, 2, ...`) tracking checkpoints within a `session_id`.
- **Sandbox-Native State**: The physical sandbox snapshot (memory + filesystem/disk state) is the source of truth for `(session_id, seq_id)`.
  - Suspending a task at `(session_id, seq_id)` snapshots/pauses its sandbox at that exact boundary.
  - Resuming from `(session_id, seq_id)` restores that exact sandbox snapshot and delivers the resume payload into the waking task.
  - If a task crashes or fails liveness probes while executing from `(session_id, N)`, the runtime discards the dirty sandbox, restores the snapshot at `(session_id, N)`, and re-delivers the input for step `N`.
- **Single-Writer & Cycle Prevention**: A `session_id` can only have one active execution turn at a time. Calling a `session_id` that is already `RUNNING` or `WAITING` (e.g., duplicate target in `CallMany` or a call cycle) immediately resolves that call entry with a `SESSION_BUSY` error without mutating the target sandbox.

---

## 2. Task & Sandbox Lifecycle

| Logical Task State | Physical Sandbox State | Description |
| :--- | :--- | :--- |
| **`RUNNING`** | **Active** (CPU/mem allocated, probed) | Task is actively executing a turn starting from `(session_id, seq_id)`. |
| **`WAITING`** | **Suspended** (snapshotted / paused) | Task emitted `YIELD_CALL` advancing to `next_seq_id`. Sandbox is suspended until all child calls finish. |
| **`IDLE` (Completed Turn)** | **Suspended** (snapshotted at `next_seq_id`) | Task emitted `YIELD_DONE`. Sandbox is suspended at `next_seq_id` and can be resumed by future calls to `session_id`. |
| **`UNHEALTHY`** | **Reclaimed / Rolled Back** | Liveness probe failed or timed out while `RUNNING`. Runtime kills the active sandbox and either retries from `(session_id, seq_id)` or returns an error to the caller. |

---

## 3. Protocol Primitives (Task <-> Runtime)

### 3.1 `RESUME` *(Runtime -> Task)*

Restores the sandbox for `(session_id, seq_id)` (or boots at `seq_id = 0` if the session is new) and delivers either the caller's `input` or the `call_results` from a completed `YIELD_CALL`:

```json
{
  "type": "RESUME",
  "session_id": "sess_a",
  "seq_id": 1,
  "input": null,
  "call_results": [
    {
      "session_id": "sess_b",
      "status": "OK",
      "output": "<payload>"
    },
    {
      "session_id": "sess_c",
      "status": "ERROR",
      "error": {
        "code": "PROBE_TIMEOUT",
        "message": "Task sess_c failed liveness checks"
      }
    }
  ]
}
```

- When invoked by a parent task (or external entrypoint), `input` carries the invocation arguments and `call_results` is empty.
- When waking up after `YIELD_CALL`, `call_results` is populated in the exact order of the requested `calls`.

### 3.2 `PROBE` *(Runtime <-> Task)*

While a task is `RUNNING`, the runtime periodically probes the task for liveness:

```json
{
  "type": "PROBE_STATUS",
  "session_id": "sess_a",
  "seq_id": 1,
  "status": "ALIVE"
}
```

- **Fencing**: Including `(session_id, seq_id)` in probe responses allows the runtime to fence stale/zombie task instances if a turn was already rolled back and resumed elsewhere.
- **No Probing When Suspended**: Tasks in `WAITING` or `IDLE` have suspended sandboxes and are not probed.

### 3.3 `YIELD` *(Task -> Runtime)*

A running turn always ends with the task emitting a yield frame, which advances its checkpoint to `next_seq_id = seq_id + 1` and triggers an immediate sandbox snapshot & suspend.

#### `YIELD_CALL` (Fan-Out / `CallMany`)

Halts the caller task, snapshots and suspends its sandbox at `next_seq_id`, and invokes one or more target sessions in parallel:

```json
{
  "type": "YIELD_CALL",
  "session_id": "sess_a",
  "next_seq_id": 1,
  "calls": [
    { "session_id": "sess_b", "task": "worker_type_1", "input": "<args_1>" },
    { "session_id": "sess_c", "task": "worker_type_2", "input": "<args_2>" }
  ]
}
```

- If a target `session_id` does not exist yet, the runtime creates its sandbox at `seq_id = 0`.
- If a target `session_id` already exists and is `IDLE`, the runtime resumes its suspended sandbox from its latest `seq_id` with the new `input`.

#### `YIELD_DONE` (Turn Complete)

Completes the current invocation, snapshots and suspends the sandbox at `next_seq_id`, and returns the result (or unhandled task error) to the waiting caller:

```json
{
  "type": "YIELD_DONE",
  "session_id": "sess_a",
  "next_seq_id": 2,
  "status": "OK",
  "output": "<final_result>",
  "error": null
}
```

---

## 4. End-to-End Call Sequence

```text
Task A ("sess_a")                 Runtime                   Task B ("sess_b")      Task C ("sess_c")
-----------------                 -------                   -----------------      -----------------
  |<-- RESUME(sess_a, seq=0) --------|
  |<-- PROBE() --------------------->|
  |                                  |
  |--- YIELD_CALL(next_seq=1, ------>|
  |     calls=[sess_b, sess_c])      |
[Snapshot & Suspend (sess_a, 1)]     |
                                     |--- RESUME(sess_b, latest_seq, in) ->|
                                     |--- RESUME(sess_c, latest_seq, in) ----------------------->|
                                     |<-- PROBE() ------------------------>|                     |
                                     |<-- PROBE() ---------------------------------------------->X (timeout!)
                                     |                                     |       [Rollback/Kill sess_c]
                                     |<-- YIELD_DONE(sess_b, OK, out) -----|
                                     |                       [Snapshot & Suspend (sess_b)]
                                     |
[Restore & Unsuspend (sess_a, 1)]    |
  |<-- RESUME(sess_a, seq=1, --------|
  |     results=[                    |
  |       {sess_b: OK, out},         |
  |       {sess_c: ERROR, err}       |
  |     ])                           |
  |                                  |
  | (Task A handles sess_c error)    |
  |--- YIELD_DONE(next_seq=2, OK) -->|
[Snapshot & Suspend (sess_a, 2)]
```

