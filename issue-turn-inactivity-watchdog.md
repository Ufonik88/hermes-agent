### Bug Description

When a turn's tool call (e.g. `terminal`) runs longer than the gateway turn-inactivity timeout without producing intermediate activity, the watchdog treats the agent as idle and hard-abandons the turn mid-execution, reaping the tool's processes.

`gateway/run.py` `_watch_gateway_turn_inactivity()` polls `agent.get_activity_summary()["seconds_since_activity"]` and calls `_abandon_timed_out_gateway_turn()` (hard interrupt + process reap) once idle >= timeout (default 1800s). Activity is stamped when a tool **starts** (`agent/tool_executor.py` `_touch_activity("executing tool: <name>")`) but is not refreshed while the tool runs unless the tool itself pushes updates. A tool that runs silently for 30+ minutes (quiet builds, long pytest suites, large downloads, network waits) therefore looks idle to the watchdog.

### Evidence (live gateway log)

```
2026-08-03 14:42:18,225 ERROR gateway.run: Agent idle for 1801s (timeout 1800s) in session agent:main:discord:thread:1500021549741244547:1500021549741244547 | last_activity=executing tool: terminal | iteration=57/90 | tool=terminal
```

Note the log itself reports `last_activity=executing tool: terminal`: the system knows a tool is in flight, yet the turn is abandoned anyway.

### Impact

- Legitimate long-running tool calls are killed mid-flight (partial builds, interrupted downloads, orphaned side effects from reaped processes).
- The agent turn is lost with a generic idle-timeout error, which looks like a crash to the user.

### Suggested fix directions

1. Refresh the activity stamp while a tool call is in flight (heartbeat every N seconds while `_current_tool` is set).
2. Exclude in-flight tool execution from the idle window, with a separate (and configurable) hard ceiling for genuinely hung tools.
3. Or expose the turn-inactivity timeout as a gateway config knob so long-tool workloads can raise it.

### Environment

- hermes-agent v0.20.0 gateway (Linux), Python 3.14, Discord platform.
