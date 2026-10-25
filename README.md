# Task Execution Protocol

- There are execution units (task), each task is a sandboxed stateful envrionment.
- Each task knows how to resume from a given "session_id" and a seq id.
- Each task reports its aliveness status through some probe.
- Each task can call another task. We want execution to halt on the caller task until the
  other task is finished. Caller task is resumed automatically when it's done.
- We will automaticlly suspend and resume the underlying sandbox.
