# Hummingbot Architecture Overview

This document is a high-level overview of runtime behavior verified in this repository.

## Runtime Shape

- One Hummingbot process runs at most one active strategy or script.
- Strategy V2 can orchestrate multiple controllers from that one active strategy.
- Controllers can manage multiple executors.

```text
bot process
  -> 0 or 1 running strategy/script
      -> 0..N controllers
          -> 0..N executors
```

If multiple independent strategies must run concurrently, use multiple Hummingbot instances/processes.

## Strategy V2 Overview

- `StrategyV2Base` initializes and starts configured controllers.
- `ControllerBase` runs controller lifecycle and control loop logic.
- Strategy ticks coordinate executor updates and strategy-level actions.

Important files:

- `hummingbot/strategy/strategy_v2_base.py`
- `hummingbot/strategy_v2/controllers/controller_base.py`
- `hummingbot/strategy_v2/runnable_base.py`

## Runtime Notes

- `manual_kill_switch` is currently enforced at tick-time in scripts that implement checks for it.
- In current behavior, a controller may still start before being stopped by a later tick-level kill-switch check.

## Scheduling

- There is no generic built-in cron scheduler for arbitrary bot jobs.
- Some strategies provide time-window support.
- Custom V2 scripts can implement scheduling in `on_tick()`.
- Process-level scheduling is typically external (cron/systemd/Docker/Kubernetes) or remote via MQTT commands.

## Remote Interface

- MQTT bridge support is implemented in `hummingbot/remote_iface/mqtt.py`.
- Detailed command/query flow, topics, and payload conventions are in `docs/mqtt.md`.
