---
name: ev-energy-dispatch-vpp-event
description: Create and cancel an ev.energy virtual-power-plant dispatch event against a dispatch coordinator, and audit its effect.
api: ev.energy v2 API
base_url: https://api.ev.energy/v2
operations:
  - get-dispatch_coordinators
  - get-dispatch_coordinators-dispatch_coordinator_id
  - post-dispatch_coordinators-dispatch_coordinator_id-dispatch_events
  - post-dispatch_coordinators-dispatch_coordinator_id-dispatch_events-dispatch_event_id-cancel
  - get-dispatch_coordinators-dispatch_coordinator_id-dispatch_logs
  - get-evses-evse_id-vpp_modes
  - get-vehicles-vehicle_id-vpp_modes
source: openapi/ev-energy-api-v2-openapi.yaml
---

# Dispatch a VPP event

Scopes: `dispatch_coordinator:read` to read coordinators and logs, `dispatch_event:write` to create or
cancel an event.

**This flow moves real load on real hardware.** A dispatch event changes when connected vehicles and
chargers draw power. Treat every write here as high-consequence and require explicit human confirmation
of the coordinator, window and target before calling step 3.

## Steps

1. `get-dispatch_coordinators` — `GET /dispatch_coordinators`. Identify the coordinator (`devt`-family
   resources hang off it) whose fleet you intend to move.
2. `get-dispatch_coordinators-dispatch_coordinator_id` — confirm the coordinator's configuration before
   writing.
3. `post-dispatch_coordinators-dispatch_coordinator_id-dispatch_events` —
   `POST /dispatch_coordinators/{dispatch_coordinator_id}/dispatch_events`. Body is a `DispatchEventWrite`.
4. To stand the event down:
   `post-dispatch_coordinators-dispatch_coordinator_id-dispatch_events-dispatch_event_id-cancel` —
   `POST .../dispatch_events/{dispatch_event_id}/cancel`, returning a `DispatchEventCancelResponse`.
5. Audit with `get-dispatch_coordinators-dispatch_coordinator_id-dispatch_logs`, and check per-asset
   participation with `get-evses-evse_id-vpp_modes` and `get-vehicles-vehicle_id-vpp_modes`.

## Rules

- **There is no idempotency key on this API.** A retried `POST` to step 3 can create a *second* dispatch
  event. On a timeout or ambiguous failure, do not retry — read
  `get-dispatch_coordinators-dispatch_coordinator_id-dispatch_logs` first and only then decide.
- Cancel is the compensating action for create. There is no update.
- `429` responses are common on bulk audits: 204 of 210 operations declare one. Honour `retry-after`.
- All errors are RFC 9457 `application/problem+json`; branch on `type`, not on `title` or `detail`.
