# Network Rail TD/TRUST feed worker

The worker lives in [`network-rail-data`](../../network-rail-data): it connects to [Network Rail's open data feeds](https://wiki.openraildata.com/index.php/About_the_Network_Rail_feeds) (STOMP), receives TD (Train Describer) and/or TRUST (Train Movements) messages, and **writes them directly to MySQL** (`` `jb.app_network_rail` ``).

## Good practice (Open Rail Data wiki)

This implementation follows the [Good Practice](https://wiki.openraildata.com/index.php/About_the_Network_Rail_feeds#Good_Practice) guidance:

| Practice | Implementation |
|----------|----------------|
| **One account, connect once** | Single STOMP connection; multiple feeds (TD and/or TRUST) via multiple subscriptions on that connection. |
| **Don't leave a failed client running** | On **authentication or authorization errors**, the process **exits** and does **not** retry (avoids hammering the service). |
| **Handle failures with exponential backoff** | On disconnect or connection failure, the worker waits **1s, 2s, 4s, 8s, 16s…** (capped at 5 minutes) before reconnecting, so the service can recover. |
| **Durable subscriber** | Optional: set `DURABLE=1` or use `--durable` so the broker can queue messages briefly if you disconnect. |
| **Stomp heartbeats** | Heartbeats `(5000, 5000)` ms are used to detect network problems. |

References:

- [About the Network Rail feeds](https://wiki.openraildata.com/index.php/About_the_Network_Rail_feeds)
- [Durable Subscription](https://wiki.openraildata.com/index.php/Durable_Subscription)

## Environment

- `NETWORK_RAIL_FEED_EMAIL` – Feed account email (required).
- `NETWORK_RAIL_FEED_PASSWORD` – Feed account password (required).
- `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` – Target database (see [`network-rail-data/README.md`](../../network-rail-data/README.md)).
- `FEED_TYPE` – `td`, `trust`, or `both` (default: `both`).
- `DURABLE` – Set to `1` (or `true`/`yes`) to use a durable subscription.

## Run

See [`network-rail-data`](../../network-rail-data): `docker compose up -d --build` on the `webapp-shared` network.
