# Deploying RLN with RTR

This compose file runs an RGB Lightning Node and RTR together. It is the reference way to run RTR: the node's API port only exists on the internal compose network, so every call from outside goes through RTR.

## Supported setups

There are exactly two supported ways to run a node: RLN alone, or RLN behind RTR with every call going through RTR. Mixing them is not supported: RTR's database is only correct if it observes every operation. Using RTR is the only secure way to interact with a node that has one. Make sure the node's API port is reachable only from RTR.

## Run

```sh
cd deploy
cp rtr.toml.example rtr.toml
# fill [webhook] url and secret
docker compose up --build --wait
```

Both images are built from source; the RLN build takes a while the first time. `RLN_SRC` points at the RLN checkout used as build context (default `../../rgb-lightning-node/dev`); that checkout must have its `rust-lightning` submodule populated (`git clone --recurse-submodules` or `git submodule update --init`), otherwise the image build fails on the path patches. `healthy` means RTR is serving, not that the node is unlocked: read `node` in the health body. `RTR_BIND` and `RTR_PORT` change the host address and port RTR is published on (default `127.0.0.1:3101`).

The node reads `rln.toml`, mounted read-only into the container and passed with `--config`. It is RLN's own config format, so any key from RLN's `sample-config.toml` can be added there (network, ports, auth, default indexer and proxy, channel limits). The file is committed as a template and meant to be edited in place: it ships with `network = "Testnet"`, which must match `[rln] network` in `rtr.toml`. On a mismatch RTR refuses to start if the node is already unlocked; otherwise the first reconcile after `/unlock` marks the node `misconfigured` in `/rtr/health` (`status` turns `degraded`) and all background work pauses until the mismatch is fixed.

Then talk to RTR for everything: RLN routes are proxied 1:1 and RTR-native routes live under `/rtr/*`.

```sh
curl -s 127.0.0.1:3101/rtr/health
curl -s -X POST 127.0.0.1:3101/init -H 'content-type: application/json' -d '{"password":"..."}'
curl -s -X POST 127.0.0.1:3101/unlock -H 'content-type: application/json' -d '{...}'
```

## Ports

Published on the host:

- `127.0.0.1:3101` -> RTR API (RLN proxy plus `/rtr/*`)
- `9735` on all interfaces -> node Lightning peer port

Not published: the node's API port `3001`. It is reachable only as `http://rln:3001` from containers on the `internal` network. Do not add a port mapping for it; the isolation rule holds by construction.

## Data

- `rln-data` volume -> node storage (`/data` in the container)
- `rtr-db` volume -> RTR SQLite mirror (`/var/lib/rtr`)

`docker compose down` keeps both volumes; `docker compose down -v` deletes them, including the node's wallet data.

## Biscuit authentication

The default `rln.toml` sets `disable_authentication = true` and publishes RTR on `127.0.0.1` only, so anyone who can reach the host's loopback can drive the node. `[service] auth_token` protects `/rtr/*` only, not the proxied RLN routes. To expose RTR beyond loopback (`RTR_BIND=0.0.0.0`), switch the node to Biscuit auth:

1. In `rln.toml` set `disable_authentication = false` and `root_public_key = "<hex>"` under `[auth]`.
2. Set `[rln] token` in `rtr.toml` to a Biscuit the engine can use for its own calls (`/nodeinfo`, `/networkinfo`, `/listtransfers`, `/listassets`, `/refreshtransfers`, `/failtransfers`).
3. Operators keep sending their own Biscuit in the `Authorization` header; proxied calls forward it to the node unchanged and RTR never adds one on their behalf.

If `rln.token` is missing or lacks the rights the engine needs, RTR exits at startup (its probe of the node fails with `node misconfigured: ...`) and `restart: unless-stopped` restarts it in a loop; check `docker compose logs rtr`.

## Webhooks

- Delivery is at-least-once: dedupe on the `x-rtr-event-id` header.
- Events are delivered strictly in order. A failing delivery blocks the ones behind it until it succeeds or is parked after `[webhook] max_attempts` attempts.
- Parked events are counted as `parked_events` in `/rtr/health`, and `status` turns `degraded` while any are parked.
