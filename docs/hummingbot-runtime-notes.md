# Hummingbot Runtime Notes

These notes capture repo-specific runtime behavior that has been verified in the codebase.

## Runtime Shape

- A single Hummingbot bot instance runs at most one active strategy or script at a time.
- Strategy V2 can still orchestrate multiple controllers from that one strategy.
- The usual hierarchy is:

```text
bot process
  -> 0 or 1 running strategy/script
      -> 0..N controllers
          -> 0..N executors
```

If multiple independent strategies must run concurrently, use multiple Hummingbot instances/processes.

## Strategy V2 Controllers

- `StrategyV2Base.__init__()` initializes controllers when the config is a `StrategyV2ConfigBase`.
- `StrategyV2Base.start()` starts all configured controllers unconditionally.
- `StrategyV2Base.on_tick()` updates executor info, reloads controller configs, and then executes strategy-level executor actions when market data is ready.
- `ControllerBase.start()` sets controller status to `RUNNING`, schedules `control_loop()`, and initializes candles.
- `ControllerBase.update_config()` only applies fields marked with `json_schema_extra={"is_updatable": True}`.

Important files:

- `hummingbot/strategy/strategy_v2_base.py`
- `hummingbot/strategy_v2/controllers/controller_base.py`
- `hummingbot/strategy_v2/runnable_base.py`

## Controller `manual_kill_switch`

- `manual_kill_switch` is defined on `ControllerConfigBase` and is updatable.
- In `scripts/v2_with_controllers.py`, it is enforced by `check_manual_kill_switch()`.
- Current behavior: a controller with `manual_kill_switch: true` may still be started first, then stopped on a later strategy tick.
- Reason: `StrategyV2Base.start()` starts every controller without checking `manual_kill_switch`, and `V2WithControllers.on_tick()` calls `super().on_tick()` before `check_manual_kill_switch()`.
- Therefore, `manual_kill_switch: true` currently means "stop/cash out this running controller on tick", not "never start this controller".

If changing this behavior, consider both:

- guarding controller startup in `StrategyV2Base.start()` or a subclass; and
- checking `manual_kill_switch` before `super().on_tick()` in scripts that use it.

## Scheduling / Cron

- There is no generic built-in cron scheduler for arbitrary bot or strategy jobs.
- Some strategies have time-window support through strategy-specific config.
- `hummingbot/strategy/conditional_execution_state.py` supports running between start/end datetimes or daily start/end times.
- Custom V2 scripts can implement scheduling inside `on_tick()`.
- For process-level automation, use external scheduling such as cron, systemd timers, Docker/Kubernetes scheduling, or MQTT commands.

## MQTT

- The MQTT bridge lives in `hummingbot/remote_iface/mqtt.py`.
- When MQTT is enabled and started, `MQTTGateway` creates a heartbeat publisher.
- Heartbeat topic format:

```text
{mqtt_namespace}/{instance_id}/hb
```

- The default namespace is `hbot`, so the default heartbeat topic is:

```text
hbot/<instance_id>/hb
```

- Related MQTT topics include:

```text
hbot/<instance_id>/status_updates
hbot/<instance_id>/log
hbot/<instance_id>/events
hbot/<instance_id>/notify
```

- `MQTTGateway` also has an internal health monitor that checks transport connectivity and attempts reconnects. This is separate from the heartbeat publisher.
