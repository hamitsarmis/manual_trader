# Manual Trader

A [Freqtrade](https://www.freqtrade.io/) setup for **manually-triggered** trades on Binance USDT-M Futures, with an automated trailing-style profit-saving exit.

The strategy (`ManualStrategy`) intentionally generates **no entry or exit signals** — positions are opened via `force_entry` (REST API / UI / Telegram) and only the `custom_exit` logic decides when to close them.

## How it works

- **Entries:** never automatic. Use Freqtrade's force-entry RPC to open longs/shorts on any whitelisted pair.
- **Exits:** `custom_exit` tracks the peak profit per trade and exits with reason `half_peak_profit_exit` once the trade has been at least **+3%** in profit and has since given back **25%** of that peak (i.e. drops below `0.75 × max_profit`).
- **Leverage:** fixed at **20x**, isolated margin.
- **Order types:** all market orders (entry, exit, stoploss, force-exit, emergency-exit).
- **Timeframe:** 1m, with `process_only_new_candles = False` so the exit logic runs every tick.

## Layout

```
config.json                                          # main bot config (dry-run, API server, DB, etc.)
pairlist-backtest-static-binance-futures-usdt.json   # exchange + static pair whitelist
user_data/
  strategies/ManualStrategy.py                       # the strategy
  hyperopts/                                         # hyperopt loss functions (sample only)
  data/                                              # downloaded OHLCV
  backtest_results/, hyperopt_results/, logs/, plot/, notebooks/
```

## Requirements

- Python 3.11+
- `freqtrade` (tested with 2026.4) on the system PATH

## Setup

Install the FreqUI web interface (one-time, required if you want the browser UI on `http://127.0.0.1:8080`):

```bash
freqtrade install-ui
```

## Secrets

No credentials are committed to this repo. Provide them in **one** of two ways before running the bot:

**Option A — `secrets.json` (recommended for local dev)**

A starter file already exists at `secrets.json` (gitignored). Open it and fill in real values:

```json
{
  "exchange":   { "key": "...", "secret": "..." },
  "api_server": { "username": "freqtrader", "password": "...", "jwt_secret_key": "..." }
}
```

It is auto-loaded via `add_config_files` in `config.json` and overrides any matching keys.

Generate a strong JWT secret:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

**Option B — environment variables**

Copy `.env.example` to `.env`, fill in the values, and `source .env` (or use any tool that loads it). Freqtrade picks up any `FREQTRADE__<SECTION>__<KEY>` variable and merges it into the config. Env vars win over both `config.json` and `secrets.json`.

## Running

The bot ships configured for **dry-run**:

```bash
freqtrade trade --config config.json
```

The REST API server starts on `http://127.0.0.1:8080` (localhost only). OpenAPI docs are enabled at `/docs`. To expose it on your LAN, change `api_server.listen_ip_address` in `config.json` to `0.0.0.0` — but only do this with a strong password and firewall.

## Docker

A `docker-compose.yml` is included so you can run the bot with the official Freqtrade image — no local Python install or `freqtrade install-ui` step needed (the upstream image bundles FreqUI).

1. Set up secrets — either fill in `secrets.json`, or copy `.env.example` → `.env` and fill it in.
2. Start in the background:

   ```bash
   docker compose up -d
   ```

3. Tail logs:

   ```bash
   docker compose logs -f
   ```

4. Stop:

   ```bash
   docker compose down
   ```

The web UI / REST API is reachable at `http://127.0.0.1:8080` (mapped to host loopback only). The trade DB persists at `user_data/manual_tradingview.db`. To pin to a specific Freqtrade version, change `image: freqtradeorg/freqtrade:stable` in `docker-compose.yml` to a specific tag (e.g. `:2026.4`).

### Switching to live trading

1. Make sure `exchange.key` / `exchange.secret` are set (via `secrets.json` or env vars).
2. In `config.json`, set `"dry_run": false`.
3. Make sure the API server `password` and `jwt_secret_key` are strong values, not the `CHANGE_ME` placeholders.

> Trading futures with real money carries the risk of total loss. Validate any change with a backtest first.

## Triggering trades manually

With the bot running, use the API to force a long or short:

```bash
# Force a long entry on BTC/USDT:USDT
# password = FREQTRADE__API_SERVER__PASSWORD from your .env; curl prompts for it
curl -u freqtrader \
  -X POST http://127.0.0.1:8080/api/v1/forceenter \
  -H 'Content-Type: application/json' \
  -d '{"pair": "BTC/USDT:USDT", "side": "long"}'
```

The position will then be managed by `custom_exit` until the profit-saving rule triggers, or you force-exit it.

## Notes

- The static whitelist is large (~300 USDT-M perpetual pairs); trim it in the pairlist file if you want a smaller universe.
- `minimal_roi` is set to `1000` and `stoploss` to `-0.99` deliberately — those exits should never fire; profit-taking is handled entirely in `custom_exit`.
- SQLite trade DB lives at `manual_tradingview.db` (gitignored).
