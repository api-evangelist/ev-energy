---
name: ev-energy-onboard-driver-vehicle
description: Onboard an ev.energy driver and connect their electric vehicle so it becomes eligible for smart charging.
api: ev.energy v2 API
base_url: https://api.ev.energy/v2
operations:
  - post-users
  - get-vehicle_compatibility-check
  - get-vehicle_onboarding
  - post-vehicle_onboarding-complete
  - post-vehicle_onboarding-onboarding_link
  - get-vehicles
  - get-vehicles-vehicle_id
  - post-vehicle_waitlist
source: https://developers.ev.energy/docs/howto/onboarding_users.md
---

# Onboard a driver and connect their vehicle

Every operation requires an OAuth 2.0 bearer token. Under `client_credentials` you must also set
`EvEnergy-User` to the **absolute URL** of the user resource on any operation that needs single-user
context; those operations return `400` without it.

## Steps

1. **Create the user.** `post-users` — `POST /users`. Requires the `user:write` scope. The response
   carries the user's absolute URL and a `user`-prefixed ULID.
2. **Check the vehicle is supported before you ask for credentials.** `get-vehicle_compatibility-check`
   — `GET /vehicle_compatibility/check`, scope `vehicle_catalogue` / `vehicle:read`. If the model is not
   supported, call `post-vehicle_waitlist` (`POST /vehicle_waitlist`) instead of failing the flow.
3. **Start the connection.** `get-vehicle_onboarding` — `GET /vehicle_onboarding`. This returns the
   manufacturer authorization surface the driver must complete. On the `response_type=json` path a
   `422 application/problem+json` is returned when the integration prerequisites are not met — read the
   `type` field, do not branch on the message string.
   To hand the flow to the driver on their own device instead, use
   `post-vehicle_onboarding-onboarding_link` (`POST /vehicle_onboarding/onboarding_link`).
4. **Complete the connection.** `post-vehicle_onboarding-complete` —
   `POST /vehicle_onboarding/complete` with the `authorization_request` identifier. A `404` means no
   `AccountAuthorizationRequest` matches; a `422` means the request expired or the integration
   prerequisites failed. Both are RFC 9457 problem documents.
5. **Confirm.** `get-vehicles` / `get-vehicles-vehicle_id` — the vehicle now appears with a `vhcl`-prefixed
   ULID and a `trim` reference.

## Rules

- **No idempotency contract exists.** ev.energy publishes no idempotency key header and no replay
  window. Treat every `POST` in this flow as *not* safe to blindly retry: on a timeout, re-read with
  `get-vehicles` before retrying step 4 or 5, or you will create duplicate state.
- Related resources are referenced by **absolute URL**, not bare ID. Send
  `{"vehicle": "https://api.ev.energy/v2/vehicle/vhcl.../"}`, not `{"vehicle": "vhcl..."}`.
- Errors are RFC 9457 `application/problem+json`. On `400` read the ev.energy `field_errors` extension —
  an object mapping request field names to arrays of validation strings.
- Rate limit is **1000 requests/hour rolling** per OAuth application + user. Read
  `x-ratelimit-remaining`; on `429` honour `retry-after` (seconds).
