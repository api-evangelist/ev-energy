---
name: ev-energy-read-charging-sessions
description: Retrieve and interpret ev.energy charging sessions, sub-sessions, energy usage and smart-charging assessment data.
api: ev.energy v2 API
base_url: https://api.ev.energy/v2
operations:
  - get-charging_sessions
  - get-charging_sessions-charging_session_id
  - get-charging_sub_sessions
  - get-charging_sub_sessions-charging_sub_session_id
  - get-charging_sub_sessions-charging_sub_session_id-energy_usage
  - get-charging_sub_sessions-charging_sub_session_id-assessment
  - get-charging_sub_sessions-charging_sub_session_id-flags
  - get-charging_sub_sessions-charging_sub_session_id-schedules
source: https://developers.ev.energy/docs/understanding/charging_sessions.md
---

# Read charging session data

Scope required: `charging_session:read`. All operations are `GET`.

## Model

A **charging session** (`cses` prefix) is one plug-in-to-unplug event and references a `vehicle` and an
`evse`. It contains one or more **charging sub-sessions** (`csub` prefix), each representing a single
contiguous period of charging *in one mode*. Energy, carbon and cost analysis hangs off the
sub-session, not the session.

## Steps

1. `get-charging_sessions` — `GET /charging_sessions`, filtered and paginated.
2. `get-charging_sessions-charging_session_id` — `GET /charging_sessions/{charging_session_id}` for one
   session plus its `sub_sessions`.
3. For each sub-session, choose the projection you need:
   - `get-charging_sub_sessions-charging_sub_session_id-energy_usage` — periodic energy intervals.
   - `get-charging_sub_sessions-charging_sub_session_id-assessment` — the smart-charging assessment,
     with `metrics` and `ratings`.
   - `get-charging_sub_sessions-charging_sub_session_id-flags` — data-quality and behaviour flags.
   - `get-charging_sub_sessions-charging_sub_session_id-schedules` — the schedule that drove this period.

## Rules

- **Paginate by the `Link` header, never by constructing URLs.** RFC 5988 `rel="previous"` / `rel="next"`;
  `about:blank` marks the ends. `page_size` defaults to 25, max 100. Collections use *either* cursor
  (`page_after` / `page_before`) or page-number (`page`) schemes — following `Link` works for both.
- Do not poll for session completion. Subscribe to the `charging_sub_session.created` and
  `charging_sub_session.ended` webhooks instead (see `asyncapi/ev-energy-webhooks.yml`); the webhook
  `data` field carries exactly the same representation as the equivalent `GET`. ev.energy states webhook
  delivery is not guaranteed, so keep a periodic `GET` reconciliation as a fallback.
- Errors are RFC 9457 problem documents. `429` carries
  `type: https://api.ev.energy/v2/problems/rate-limit-exceeded/` and a `retry-after` header.
