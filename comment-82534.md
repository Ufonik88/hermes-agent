Another data point for this failure mode, with opencode-go as the aux vision provider and no fallback attempted. Our gateway log (2026-08-12):

```
ERROR tools.vision_tools: Error analyzing image: Error code: 400 - {'error': {'param': '', 'type': 'server_error', 'message': 'Error from provider (Console Go): Upstream request failed: [404] No endpoints found that support image input'}}
```

Here the primary/auto-routed provider itself (Console Go / opencode-go) has no image-capable endpoint at all, so it fails before any main-model fallback runs, and the agent sees only "Error analyzing image" with no actionable message. A capability check in the auxiliary vision router (probe /models for image support or a capability flag) plus a clear error naming the provider and model would help a lot here.
