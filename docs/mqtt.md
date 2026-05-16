# Hummingbot MQTT Reference

This document captures detailed MQTT bridge behavior verified in code.

## Topic Prefix

```text
{mqtt_namespace}/{instance_id}
```

- Default namespace is `hbot`.
- Namespace values ending with `/` or `.` are normalized (trailing character removed).

## Bridge Control from CLI

- `mqtt start [-t/--timeout <seconds>]`
- `mqtt stop`
- `mqtt restart [-t/--timeout <seconds>]`

Headless mode expects MQTT enabled (`mqtt_autostart: true`) so the bot can be controlled remotely.

## MQTT Flow

1. **Bridge availability**
   - On startup, gateway publishes `online` to `.../status_updates`.
   - On shutdown, gateway publishes `offline` to `.../status_updates`.
   - Heartbeat publisher emits to `.../hb`.
2. **Command/query RPC**
   - Commands are RPC services bound to `.../<command>`.
   - Client sends request payload to command topic and receives response on client-provided reply topic.
3. **Streaming topics**
   - Logs, notifications, status updates, and internal market events are pub/sub streams.
4. **External event ingress**
   - Gateway subscribes to `.../external/event/*` and dispatches to registered in-bot listeners.

## Core Topics

- `hbot/<instance_id>/hb`
- `hbot/<instance_id>/status_updates`
- `hbot/<instance_id>/log`
- `hbot/<instance_id>/events`
- `hbot/<instance_id>/notify`
- `hbot/<instance_id>/external/event/*`

RPC command/query topics:

- `hbot/<instance_id>/start`
- `hbot/<instance_id>/stop`
- `hbot/<instance_id>/config`
- `hbot/<instance_id>/import`
- `hbot/<instance_id>/status`
- `hbot/<instance_id>/history`
- `hbot/<instance_id>/balance/limit`
- `hbot/<instance_id>/balance/paper`

## RPC Command/Query Payloads

- `.../start`
  - Request keys: `log_level`, `v2_conf`, `is_quickstart`, `async_backend`
  - Backward-compatible request key: `conf` is treated as `v2_conf`
  - Response keys: `status`, `msg`
- `.../stop`
  - Request keys: `skip_order_cancellation`, `async_backend`
  - Response keys: `status`, `msg`
- `.../config`
  - Request keys: `params` (list of key/value tuples)
  - Response keys: `changes`, `config`, `status`, `msg`
- `.../import`
  - Request keys: `strategy`
  - Response keys: `status`, `msg`
- `.../status`
  - Request keys: `async_backend`
  - Response keys: `status`, `msg`, `data`
- `.../history`
  - Request keys: `days`, `verbose`, `precision`, `async_backend`
  - Response keys: `status`, `msg`, `trades`
- `.../balance/limit`
  - Request keys: `exchange`, `asset`, `amount`
  - Response keys: `status`, `msg`, `data`
- `.../balance/paper`
  - Request keys: `asset`, `amount`
  - Response keys: `status`, `msg`, `data`

Status codes:

- `200` success
- `400` error

Notes:

- `async_backend=true` usually behaves as fire-and-forget.
- `async_backend=false` returns synchronous output where supported, subject to timeout paths.
- Response routing for RPC depends on request metadata such as `reply_to`.

## Query Examples with mosquitto

Read streams:

```text
mosquitto_sub -h <broker_host> -p <broker_port> -t 'hbot/<instance_id>/status_updates' -v
mosquitto_sub -h <broker_host> -p <broker_port> -t 'hbot/<instance_id>/events' -v
mosquitto_sub -h <broker_host> -p <broker_port> -t 'hbot/<instance_id>/log' -v
```

Publish external event to listener topics:

```text
mosquitto_pub -h <broker_host> -p <broker_port> \
  -t 'hbot/<instance_id>/external/event/order/market' \
  -m '{"timestamp":1730000000000,"sequence":1,"type":"eevent","data":{"type":"buy","amount":"0.01"}}'
```

For command RPC, use an MQTT RPC-capable client that sets reply routing metadata. Plain `mosquitto_pub` is suitable for pub/sub topics but not full request/reply RPC workflows.
