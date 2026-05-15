# Improvements

## MQTT First-Class Startup And Bot State Reporting

### Problem

MQTT is the primary remote control and status-reporting medium for headless Hummingbot deployments, but current startup treats MQTT as a background service. `HummingbotApplication.__init__()` calls `mqtt_start()`, which schedules `start_mqtt_async()` without awaiting readiness. Quickstart can continue into strategy loading and startup before MQTT is healthy.

This creates a risk where strategy failures, bootstrap status, and early runtime state are not reliably visible over MQTT.

### Target Behavior

MQTT should become a first-class prerequisite service whenever `mqtt_autostart` is enabled.

- In headless mode, force `mqtt_autostart = true`.
- In headless mode, read only the config required to connect MQTT before initializing Hummingbot runtime services.
- In headless mode, connect the MQTT transport and status publisher before creating/running trading, Gateway, UI, or strategy services.
- In CLI mode, apply the same MQTT-first behavior when `mqtt_autostart` is enabled.
- If MQTT cannot connect, block and retry indefinitely instead of starting the strategy in a partially observable state.
- Publish lifecycle state through `status_updates`.
- Publish detailed errors through `/log` and `/notify`.

### Proposed Startup Order

1. Parse CLI args and environment variables.
2. Unlock encrypted config with `CONFIG_PASSWORD`.
3. Load the minimal client config needed for MQTT connection settings.
4. Enable `mqtt_autostart` for headless mode.
5. Establish the MQTT transport before initializing Hummingbot runtime services.
6. Publish MQTT lifecycle status: `bootstrapping`.
7. Create `HummingbotApplication` using the established MQTT gateway/session.
8. Initialize TradingCore, Gateway monitor, UI/headless mode, and full application services.
9. Initialize basic local logging and patch MQTT log handlers.
10. Load full system config.
11. Bind MQTT commands, notifier, external events, and market events after their app dependencies exist.
12. Load and validate strategy config.
13. Wait for Gateway readiness if the strategy requires Gateway.
14. Start strategy.
15. Start market event forwarding.
16. Publish lifecycle status: `strategy running`.
17. Enter UI or headless run loop.

### Implementation Notes

- Replace fire-and-forget MQTT startup with an awaitable readiness gate.
- Add an application-level helper such as `ensure_mqtt_ready()`.
- Split MQTT connection bootstrap from full application bootstrap so the MQTT config can be loaded and connected first.
- Split `MQTTGateway` initialization into two phases:
  - Bootstrap phase: connection parameters, heartbeat, and `status_updates` publisher.
  - Application phase: log handler, notifier, RPC commands, external events, and market event forwarding.
- Avoid calling `mqtt_start()` from `HummingbotApplication.__init__()`.
- Call the MQTT readiness gate from quickstart before strategy loading.
- Call the MQTT readiness gate from CLI strategy start when `mqtt_autostart` is enabled.
- Re-patch MQTT loggers after every `init_logging(...)` call.
- Start market event forwarding only after strategy markets are initialized.

### Status Reporting Contract

Continue using:

```text
hbot/<instance_id>/status_updates
```

Use `StatusUpdateMessage` for lifecycle state. Keep existing fields backward compatible:

- `timestamp`
- `type`
- `msg`

Add optional structured data:

- `data`

Recommended lifecycle types:

- `bootstrap`
- `strategy`
- `gateway`
- `shutdown`

Example messages:

```json
{"type": "bootstrap", "msg": "bootstrapping"}
{"type": "strategy", "msg": "loading"}
{"type": "strategy", "msg": "running"}
{"type": "strategy", "msg": "failed"}
{"type": "shutdown", "msg": "offline"}
```

### Test Plan

- Verify headless startup waits for MQTT before loading a strategy.
- Verify CLI startup waits for MQTT only when `mqtt_autostart` is enabled.
- Verify strategy startup is not attempted while MQTT is unhealthy.
- Verify MQTT reconnect retry does not start the strategy prematurely.
- Verify lifecycle status messages are published in order.
- Verify startup failures are visible on `/log` and `status_updates`.
- Verify logging reinitialization does not remove MQTT log forwarding.
- Verify existing behavior is preserved when CLI mode has `mqtt_autostart: false`.
