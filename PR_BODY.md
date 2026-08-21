## Summary

The gateway turn-inactivity watchdog (`gateway/run.py::_watch_gateway_turn_inactivity`, default timeout 30 min) abandons a turn once `seconds_since_activity` exceeds the inactivity timeout. Activity was only stamped when a tool **started** and when it **completed**, so a tool call that runs silently for 30+ minutes — quiet builds, long pytest suites, large model/download operations, network waits that emit no output — left the activity clock frozen at "executing tool: X". The watchdog then hard-abandoned a turn that was still making real progress, reaping the tool's processes mid-execution (issue #84491).

This PR adds a lightweight daemon-thread heartbeat inside `_run_agent_tool_execution_middleware` that refreshes the agent's activity clock every 30s while a tool call is in flight, until the call returns.

## Motivation

This is a real, recurring failure mode in live gateway operation. Our own filed bug [#84491](https://github.com/NousResearch/hermes-agent/issues/84491) documents turns being abandoned during long tool calls. The symptom is invisible until it bites: a 45-minute build or a large test run appears to "hang" to the gateway, the turn is reaped, and the user gets a stale/empty response while the spawned subprocess is killed. The fix keeps a turn marked live for exactly as long as a tool is legitimately executing.

The heartbeat touches `agent._touch_activity`, which updates `self._last_activity_ts` (`run_agent.py:3788`); `get_activity_summary` -> `build_activity_snapshot` derives `seconds_since_activity` from that timestamp (`agent/session_activity.py:92`), the exact field the watchdog reads (`gateway/run.py:3055`). So the heartbeat drives the correct clock.

## Changes Made

- `agent/tool_executor.py`
  - New module constant `_TOOL_ACTIVITY_HEARTBEAT_INTERVAL_S = 30.0` (far below the 1800s default timeout).
  - New helper `_run_tool_activity_heartbeat(agent, stop_event, label, interval)` — a daemon thread that touches `agent._touch_activity(label)` every interval until `stop_event` is set; swallows all exceptions so a heartbeat can never break the agent loop.
  - In `_run_agent_tool_execution_middleware`, wraps the `execute(final_args)` call: starts the heartbeat thread, returns the result, and stops the thread in a `finally` (covers normal return, exceptions, and the case where `execute` is never reached because a guardrail/authorization block short-circuits before dispatch — the heartbeat is only started after those gates clear).
  - Both the sequential and the concurrent tool-execution paths funnel through this single middleware, so one heartbeat covers every tool.

## Verification / Testing

- Independent code review (fresh subagent, no shared context) returned `passed: true` with no security or logic concerns.
- Static security scan: no hardcoded secrets, no `os.system`/`subprocess(shell=True)`, no `eval`/`exec`, no `pickle.loads`. `ruff check` clean.
- New tests in `tests/run_agent/test_tool_activity_heartbeat.py` (5 tests, all passing on Python 3.12, which matches upstream CI):
  - `test_heartbeat_touches_periodically_and_stops` — unit test of the heartbeat thread lifecycle.
  - `test_slow_tool_call_refreshes_activity_during_execution` — integration: a slow tool gets mid-call activity stamps.
  - `test_fast_tool_call_does_not_leave_stray_heartbeat` — fast tools don't leave the thread running.
  - `test_heartbeat_stops_when_execute_raises` — exception-safety: the thread tears down on raise.
  - `test_concurrent_tool_call_heartbeat` — the concurrent execution path also stamps via the shared chokepoint.

Command: `python -m pytest tests/run_agent/test_tool_activity_heartbeat.py -o 'addopts=' -q`
Result: `5 passed in 4.09s`

Note on scope: a custom tool with no internal timeout will now keep the gateway turn alive via the heartbeat, so the 30-min gateway backstop no longer fires for a genuinely hung tool. This is bounded by the tool layer's own timeouts (terminal `timeout` default 180s, concurrent batch deadline ~420s). All production tools retain a timeout, so this is an accepted, documented trade-off — the heartbeat extends turn life only while the tool is legitimately running.
