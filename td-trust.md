# Network Rail TD/TRUST feed worker

This worker connects to [Network Rail's open data feeds](https://wiki.openraildata.com/index.php/About_the_Network_Rail_feeds) (STOMP), receives TD (Train Describer) and/or TRUST (Train Movements) messages, and forwards them to the webapp API.

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
- `WEBAPP_URL` – Base URL of the webapp (e.g. `http://nginx:80`) for posting messages; omit to only print to stdout.
- `WEBAPP_FEED_SECRET` – Secret for `X-Feed-Secret` when posting to the webapp; omit if not posting.
- `FEED_TYPE` – `td`, `trust`, or `both` (default: `both`).
- `DURABLE` – Set to `1` (or `true`/`yes`) to use a durable subscription.

## Run

```bash
pip install -r requirements.txt
python main.py
# Optional: durable subscription
python main.py --durable
```

In Docker, the webapp `docker-compose` runs this worker as the `td-trust-worker` service.
