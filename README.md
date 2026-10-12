# Metasquare Oracle (`mso`)

Self-hosted multi-asset price oracle with REST and gRPC APIs, built to run on a Raspberry Pi 5 behind a Cloudflare Tunnel.

[![CI](https://github.com/Niraj-Kamdar/mso/actions/workflows/ci.yml/badge.svg)](https://github.com/Niraj-Kamdar/mso/actions/workflows/ci.yml)
[![Docker](https://github.com/Niraj-Kamdar/mso/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/Niraj-Kamdar/mso/actions/workflows/docker-publish.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Container Image](https://img.shields.io/badge/ghcr.io-niraj--kamdar%2Fmso-blue?logo=docker)](https://github.com/Niraj-Kamdar/mso/pkgs/container/mso)

> **Part of Moor** — leveraged funds on Solana. The protocol hub, moor-program (private), holds the program, SDK
> and docs. Public entry points: the web app at [app.moor.fund](https://app.moor.fund) and the
> [whitepaper](https://github.com/moor-protocol/whitepaper). Related: moor-daemon (private) · mockamm (private)

## Where it fits

mso is a general-purpose oracle: any application can run it and read prices over REST or gRPC with an API key. Moor
uses it to feed its devnet prices: on the operator's Raspberry Pi, moor-daemon reads prices from mso over the local
Docker network (`http://mso:4000`) and pushes them to Solana devnet. Moor is not deployed to mainnet.

## Quickstart

Multi-arch images (`linux/amd64`, `linux/arm64`) are published to GitHub Packages:

```bash
docker run -d \
  --name metasquare-oracle \
  --restart unless-stopped \
  -p 4000:4000 \
  -p 50051:50051 \
  -v mso-data:/data \
  ghcr.io/niraj-kamdar/mso:latest
```

Docker Compose:

```bash
git clone https://github.com/Niraj-Kamdar/mso.git
cd mso
docker compose up -d
```

From source (Rust 1.80+ and `protoc`; on Debian/Ubuntu/Raspberry Pi OS: `sudo apt install -y protobuf-compiler`):

```bash
cargo build --release
./target/release/mso serve   # starts feeds, rollups, REST and gRPC servers
```

## Features

- **Ingestion tiers**
  - Fast (1 s): `SOL/USD` from the Binance WebSocket stream, with Hyperliquid fallback.
  - Standard (60 s batch): `BTC/USD`, `ETH/USD`, `ZEC/USD`, `PAXG/USD` (tokenized gold via Binance/Jupiter) and
    Backed xStocks (`SPY/USD`, `NVDA/USD`, `GOOG/USD`, `QQQ/USD`, `TSLA/USD` via Jupiter, Yahoo Finance fallback).
- **Rollups**: raw ticks → `candles_1m` (7-day retention) → `candles_1h` (90-day retention) → `candles_1d`
  (permanent). Storage stays under 30 MB.
- **In-memory cache**: responses served from memory; client `max_age_ms` is clamped to lower and upper bounds.
- **API keys**: scoped, with TTL, hashed with SHA-256 and cached in memory; hard cap of 100 active keys.
- **Interfaces**: gRPC (Tonic) unary and streaming ticks on port `50051`; REST (Axum) plus the Scalar API reference
  on port `4000`.

## Configuration

Assets, poll frequency and retention come from a TOML file. Without one, the built-in
[`src/config.default.toml`](src/config.default.toml) is used (it is also the reference). A custom file only needs the
keys it changes; `assets` replaces the default list when present.

```toml
# mso.toml
[poll_secs]
binance = 10          # poll Binance every 10s instead of 60s

[[assets]]
symbol = "SOL/USD"
source = "binance-ws" # binance-ws | binance | hyperliquid | jupiter | yahoo
id = "SOLUSDT"
fallback = { source = "hyperliquid", id = "SOL" }

[[assets]]
symbol = "DOGE/USD"
source = "hyperliquid"
id = "DOGE"
```

```bash
./target/release/mso serve --config mso.toml   # or MSO_CONFIG=mso.toml
# Docker: mount the file and set MSO_CONFIG
docker run ... -v $PWD/mso.toml:/config/mso.toml:ro -e MSO_CONFIG=/config/mso.toml ghcr.io/niraj-kamdar/mso:latest
```

Unknown keys, duplicate symbols, unknown sources and zero intervals are rejected at startup.

## API keys

```bash
# Docker:
docker exec -it metasquare-oracle mso key create --app pnl-backend --ttl-days 30

# Local:
./target/release/mso key create --app pnl-backend --ttl-days 30
./target/release/mso key list               # keys and quota status
./target/release/mso key revoke --id <KEY_ID>
```

## API reference

Send an active key as `x-api-key: <KEY>`, `Authorization: Bearer <KEY>`, or `?key=<KEY>`.

| Endpoint | Method | Parameters | Description |
|---|---|---|---|
| `/health` | GET | None (public) | Uptime and service health |
| `/api/v1/price` | GET | `symbol`, `max_age_ms` | Latest price tick with cache indicator |
| `/api/v1/ticks` | GET | `symbol`, `limit` (1..300) | Recent ticks in chronological order, for charts and sparklines |
| `/api/v1/candles` | GET | `symbol`, `interval` (`1s`,`1m`,`1h`,`1d`), `limit` | OHLC + TWAP candles |
| `/api/v1/stats` | GET | `symbol`, `window_ms` (10s..24h) | Windowed high/low, TWAP and return % |
| `/` or `/docs` | GET | None (public) | Interactive Scalar API reference |
| `/openapi.json` | GET | None (public) | OpenAPI 3.1.0 spec |

## Deployment on Raspberry Pi 5 with Cloudflare Tunnel

Systemd service (`/etc/systemd/system/mso.service`):

```ini
[Unit]
Description=Metasquare Oracle (mso)
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/mso
ExecStart=/home/pi/mso/target/release/mso serve --db /home/pi/mso/oracle.db
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Cloudflare Tunnel config (`~/.cloudflared/config.yml`):

```yaml
tunnel: <TUNNEL_UUID>
credentials-file: /home/pi/.cloudflared/<TUNNEL_UUID>.json

ingress:
  - hostname: oracle.metasquare.tech
    service: http://localhost:4000
  - service: http_status:404
```

## Documentation

- Interactive API reference at `/` or `/docs` on port `4000`; OpenAPI spec at `/openapi.json`
- Default configuration (reference for every key): [`src/config.default.toml`](src/config.default.toml)
- gRPC service definition: [`proto/oracle.proto`](proto/oracle.proto)
- Docker Compose file: [`docker-compose.yml`](docker-compose.yml)

## License

[MIT](LICENSE).
