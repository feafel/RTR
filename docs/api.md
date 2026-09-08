# RTR endpoints

Everything under `/rtr/*` is answered by RTR itself from its mirror database; it never touches the node. Any other `/rtr/*` path is 404 and is never proxied.

## `GET /rtr/health`

```sh
curl -s 127.0.0.1:3101/rtr/health
{"status":"ok","node":"unlocked","pending_transfers":1,"pending_payments":0,"parked_events":0,"last_full_sync_at":1756640000}
```

| Field | Meaning |
| --- | --- |
| `status` | `ok` only when `node` is `unlocked` and `parked_events` is 0, otherwise `degraded` |
| `node` | `unknown`, `locked`, `unlocked`, `down` or `misconfigured` (node reachable and unlocked but failing the `rln.network` check; background work is paused), as last observed |
| `pending_transfers` | Mirrored rows in a `Waiting*` status |
| `pending_payments` | Mirrored LN payments in a non-terminal status (`Pending`, `Claimable`, `Claiming`) |
| `parked_events` | Webhook events that exhausted `webhook.max_attempts` and are no longer retried |
| `last_full_sync_at` | Unix time of the last completed full sync, or `null` |

Health always answers 200 while RTR is serving and never requires a token. For an orchestrator, a 200 means "RTR up", not "node ready": read `node`.

## `GET /rtr/transfers`

Lists mirrored transfers, newest first, as `{"transfers":[...]}` using the same transfer object as webhook payloads.

| Query | Values |
| --- | --- |
| `status` | `Initiated`, `WaitingCounterparty`, `WaitingSafeHeight`, `WaitingBroadcast`, `WaitingConfirmations`, `Settled`, `Failed` |
| `asset_id` | Exact asset id |
| `limit` | Default 100, capped at 1000 |

Invalid values give 400.

## `GET /rtr/transfers/{id}`

One transfer by RTR id, or 404.

## `GET /rtr/payments`

Lists mirrored LN payments, newest first, as `{"payments":[...]}` using the same payment object as webhook payloads.

| Query | Values |
| --- | --- |
| `status` | `Pending`, `Claimable`, `Claiming`, `Succeeded`, `Failed`, `Cancelled` |
| `limit` | Default 100, capped at 1000 |

Invalid values give 400.

## `GET /rtr/openapi.yaml`

The RLN spec passed with `--openapi`, or 404 when none was given.

## Auth

When `service.auth_token` is set, `/rtr/transfers`, `/rtr/transfers/{id}`, `/rtr/payments` and `/rtr/openapi.yaml` require `Authorization: Bearer <token>` (scheme case-insensitive) and answer 401 otherwise. It guards `/rtr/*` only, never the proxied RLN routes, which stay under RLN's own auth.

## Errors

Errors from RTR itself use RLN's error shape: `{"error":"...","code":<http status>,"name":"<ErrorName>"}`. Internal failures are reported as `internal error`; the cause goes to RTR's log.
