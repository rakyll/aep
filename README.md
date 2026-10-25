# Task Execution Protocol

- There are execution units (task), each task is a sandboxed stateful envrionment.
- Each task knows how to resume from a given "session_id".
- Each task reports its aliveness status through some probe.
