# Fibonsai

> Smart money finally grows on trees.

Open-source infrastructure for intelligent trading — built around
[algorithms](https://en.wikipedia.org/wiki/Algorithm) and
[machine learning](https://en.wikipedia.org/wiki/Machine_learning).

We conduct educational activities and publish news on intelligent bots,
automated strategies, and AI agents for portfolio management.

-   **[Cryptomeria](#projects)** — a production-grade market-data pipeline
    written in [Rust](https://www.rust-lang.org/)
-   **[License](#license)** — everything is Apache-2.0

---

## Projects

Cryptomeria is a three-stage pipeline that turns raw exchange feeds into an
analytical time-series store.

| Project | Stage | Description |
| :--- | :--- | :--- |
| [`cryptomeria-ingest`](https://github.com/fibonsai/cryptomeria-ingest) | Ingest | Multi-exchange WebSocket library (OKX, Kraken, Bitstamp, **Bitvavo**) returning normalized `LOB` and `Trade` streams. |
| [`cryptomeria-marketdata`](https://github.com/fibonsai/cryptomeria-marketdata) | Broker | Service that subscribes to `cryptomeria-ingest`, fans out multiple exchanges in parallel, and publishes normalized events over an NNG TCP pub/sub broker. |
| [`cryptomeria-historic`](https://github.com/fibonsai/cryptomeria-historic) | Store | NNG subscriber that deserializes the events and persists normalized LOB/trade data into [QuestDB](https://questdb.io) over the QWP/WebSocket protocol (QuestDB 10+) with embedded schema migrations. |

### Architecture

```mermaid
flowchart TB
    %% Exchange feeds
    OKX["OKX"]
    Kraken["Kraken"]
    Bitstamp["Bitstamp"]
    Bitvavo["Bitvavo"]

    %% Stage 1 - Ingest (library)
    ingest("cryptomeria-ingest - Rust library")

    %% Stage 2 - Marketdata (service)
    marketdata["cryptomeria-marketdata - service bounded channel to NNG"]

    %% NNG pub/sub broker
    nng(["NNG PUB/SUB<br/>topic: {kind}__{instrument}"])

    %% Stage 3 - Historic (subscriber)
    historic["cryptomeria-historic - subscriber and persist"]

    %% Analytical store
    questdb[("QuestDB - analytical time-series store")]

    %% Strategy
    strategy["cryptomeria-strategy (future) - process rules and send signals"]

    %% Trader
    trader["cryptomeria-trader (future) - process signals, manage positions"]

    %% Risk Management
    risk["cryptomeria-risk (future) - risk management"]

    OKX --> ingest
    Kraken --> ingest
    Bitstamp --> ingest
    Bitvavo --> ingest
    ingest --> marketdata
    marketdata --> nng
    nng --> historic
    historic --> questdb
    questdb --> strategy
    nng --> strategy
    strategy --> trader
    risk --> trader
    trader --> OKX
    trader --> Kraken
    trader --> Bitstamp
    trader --> Bitvavo

    classDef exchange
    classDef store
    class OKX,Kraken,Bitstamp,Bitvavo exchange
    class questdb store
```

Each `cryptomeria-*` crate is independently versioned, licensed, and
documented. They share a common `MarketDataItem` data model so that items
flow from ingest to store without translation loss.

### Feature matrix

| Feature | `cryptomeria-ingest` | `cryptomeria-marketdata` | `cryptomeria-historic` |
| :--- | :---: | :---: | :---: |
| Exchanges | OKX, Kraken, Bitstamp, Bitvavo | OKX, Kraken, Bitstamp, Bitvavo | OKX, Kraken, Bitstamp, Bitvavo |
| Normalized `LobItem` + `TradeItem` | ✅ | ✅ | ✅ |
| Bitstamp LOB stream | ⚠ disabled¹ | ⚠ disabled¹ | ⚠ disabled¹ |
| Automatic reconnect (backoff + jitter) | ✅ | ✅ | — |
| NNG pub/sub topic routing | — | ✅ | ✅ (sub) |
| Schema migrations | — | — | ✅ (V1 `trades`, V2 `lob_levels`) |
| Dry-run / test-timeout for CI | ✅ | ✅ | ✅ |

¹ The Bitstamp **LOB** order-book stream is disabled due to a known bug ([issue #65](https://github.com/fibonsai/cryptomeria-ingest/issues/65)). Bitstamp *trades* work normally — only LOB snapshots/updates on Bitstamp are affected. Prefer OKX, Kraken, or Bitvavo when you need LOB data.

---

## Quick start

The fastest path to a full pipeline is to run every stage with `cargo`. The three
crates ship as separate repositories (there is no shared workspace), so build each
in its own directory. `cryptomeria-ingest` is pulled in automatically as a
dependency of `cryptomeria-marketdata`.

```bash
# 1. Run QuestDB (QuestDB 10+), exposed on ws port 9000
docker run -d -p 9000:9000 questdb/questdb

# 2. Clone the stack
git clone https://github.com/fibonsai/cryptomeria-ingest
git clone https://github.com/fibonsai/cryptomeria-marketdata
git clone https://github.com/fibonsai/cryptomeria-historic

# 3. Build marketdata (this also builds cryptomeria-ingest)
(cd cryptomeria-marketdata && cargo build --release)

# 4. Run marketdata: NNG broker + live exchange feeds.
#    The default config subscribes to OKX + Kraken + Bitvavo; Bitvavo needs
#    credentials (BITVAVO_API_KEY / BITVAVO_API_SECRET) — see config.toml.
./cryptomeria-marketdata/target/release/marketdata \
  --config cryptomeria-marketdata/config.toml

# 5. Build and run historic: NNG subscriber -> QuestDB (QWP/WebSocket)
(cd cryptomeria-historic && cargo build --release)
./cryptomeria-historic/target/release/cryptomeria-historic \
  --nng-addr tcp://127.0.0.1:14242 \
  --qdb-conf "ws::addr=localhost:9000"

# 6. Inspect the live NNG feed from any NNG client
nngcat -sub -t "lob__*" tcp://127.0.0.1:14242
```

For CI without a broker, run marketdata in dry-run and let it exit on its own:

```bash
./cryptomeria-marketdata/target/release/marketdata \
  --config cryptomeria-marketdata/config.toml \
  --dry-run --test-timeout-secs 10
```

See each project's own `README.md` for full API reference, configuration, and
CLI options.

---

## Getting involved

We follow [Rust] idioms and a shared set of standards across all crates:

- **Async Rust** on `Tokio` (Tokio-Tungstenite for WebSocket feeds)
- **Apache-2.0** — contributions are licensed under the same
- **Zero task leaks** — streams abort their background tasks on drop
- **Tested + linted in CI** — `cargo test`, `cargo clippy -D warnings`, [`cargo-audit`](https://github.com/RustSec/cargo-audit)

| Resource | Link |
| :--- | :--- |
| Code of conduct | [CODE_OF_CONDUCT.md](https://github.com/fibonsai/cryptomeria-ingest/blob/main/CODE_OF_CONDUCT.md) |
| How to contribute | [CONTRIBUTING.md](https://github.com/fibonsai/cryptomeria-ingest/blob/main/CONTRIBUTING.md) |
| Report a vulnerability | [SECURITY.md](https://github.com/fibonsai/cryptomeria-ingest/blob/main/SECURITY.md) |
| Development guide | [AGENTS.md](https://github.com/fibonsai/cryptomeria-ingest/blob/main/AGENTS.md) |

[Rust]: https://www.rust-lang.org/

---

## License

All projects are licensed under the **Apache License 2.0** — see each crate's
[`LICENSE`](https://github.com/fibonsai/cryptomeria-ingest/blob/main/LICENSE).
