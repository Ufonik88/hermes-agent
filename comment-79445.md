Hit the same failure on our gateway (hermes-agent v0.20.0, Linux):

```
2026-08-05 22:34:37,654 ERROR hermes_plugins.discord_platform.adapter: [Discord] Discord Gateway WebSocket remained unhealthy (ack_stale); forcing reconnect
2026-08-05 22:34:37,818 ERROR gateway.run: Fatal discord adapter error (discord_websocket_health_stale): Discord Gateway WebSocket health check failed: ack_stale
```

Recovery was automatic after the forced reconnect in our case. We also saw the same `ack_stale` warning (without the fatal) on 2026-07-26 22:23. Sharing in case frequency or conditions help with reproduction.
