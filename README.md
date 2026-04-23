# Tradovate Copy Trading System

A high-performance Go application for replicating trades from a leader account to multiple follower accounts on Tradovate, designed for prop firm traders.

## Features

- **Session-Based Authentication**: No API subscription required - uses standard Tradovate credentials
- **Real-Time Order Monitoring**: WebSocket-based monitoring for millisecond-level precision
- **Parallel Order Execution**: Uses Go goroutines to place orders on all follower accounts simultaneously
- **Automatic Reconnection**: Handles disconnections and token expiration automatically
- **Ratio Scaling**: Configure different position sizes for follower accounts
- **Contract Symbol Caching**: Efficient contract lookups for faster replication
- **Statistics Tracking**: Monitor success/failure rates in real-time

## How It Works

1. Authenticates with Tradovate using username/password (no API key needed)
2. Opens WebSocket connections to both leader and follower accounts
3. Monitors leader account for order events (fills, working orders)
4. Immediately replicates orders to all follower accounts in parallel
5. Handles token refresh and reconnection automatically

## Installation

### Prerequisites

- Go 1.19 or later
- Tradovate account(s) with live credentials

### Build from Source

```bash
git clone <repository-url>
cd tradovate_copier
go mod download
go build -o tradovate-copier
```

For Windows:
```bash
go build -o tradovate-copier.exe
```

## Configuration

### Generate Example Config

```bash
./tradovate-copier -init
```

This creates a `config.json` file with the following structure:

```json
{
  "leader_account": {
    "username": "leader@example.com",
    "password": "your_password_here",
    "account_id": 123456,
    "name": "Leader Account"
  },
  "follower_accounts": [
    {
      "username": "follower1@example.com",
      "password": "follower1_password",
      "account_id": 234567,
      "name": "Follower Account 1"
    }
  ],
  "tradovate_api": {
    "auth_url": "https://live.tradovateapi.com/v1/auth/accesstokenrequest",
    "websocket_url": "wss://live.tradovateapi.com/v1/websocket",
    "app_id": "TradovateCopier",
    "app_version": "1.0"
  },
  "settings": {
    "ratio_scaling": 1.0,
    "daily_loss_limit": 0,
    "enable_risk_filter": false,
    "reconnect_interval_seconds": 5,
    "heartbeat_interval_seconds": 30
  }
}
```

### Configuration Fields

**Leader Account**:
- `username`: Tradovate login email
- `password`: Tradovate password
- `account_id`: Your Tradovate account ID (find in account settings)
- `name`: Friendly name for logging

**Follower Accounts**:
- Array of accounts that will copy the leader's trades
- Same fields as leader account

**API Settings**:
- `auth_url`: Tradovate authentication endpoint (use demo for testing)
- `websocket_url`: WebSocket endpoint
- `app_id`: Application identifier
- `app_version`: Version string

**Trading Settings**:
- `ratio_scaling`: Multiplier for follower position sizes (1.0 = same size, 2.0 = double, 0.5 = half)
- `daily_loss_limit`: Maximum daily loss before stopping (0 = disabled)
- `enable_risk_filter`: Enable risk management features
- `reconnect_interval_seconds`: Wait time before reconnection attempts
- `heartbeat_interval_seconds`: WebSocket keepalive interval

### Finding Your Account ID

1. Log in to https://trader.tradovate.com/
2. Open browser developer tools (F12)
3. Go to Network tab
4. Look for API calls containing your account information
5. Find the `accountId` field in the response

## Usage

### Run the Application

```bash
./tradovate-copier
```

Or with custom config:
```bash
./tradovate-copier -config my-config.json
```

### Command Line Options

- `-config <file>`: Specify configuration file (default: config.json)
- `-init`: Create example configuration file
- `-version`: Show version information

### What to Expect

When you start the application:

```
╔════════════════════════════════════════╗
║   Tradovate Copy Trading System       ║
║   Version: 1.0.0                      ║
╚════════════════════════════════════════╝

Configuration loaded successfully
Leader Account: Leader Account (ID: 123456)
Follower Accounts: 2

=== Connecting to Follower Accounts ===
Follower account Follower Account 1 connected successfully
Follower account Follower Account 2 connected successfully
All 2 follower account(s) connected successfully

=== Starting Leader Account Monitor ===
WebSocket connected and authorized successfully
Leader account authenticated: leader@example.com
Monitor started successfully. Listening for orders...

✓ System is running. Press Ctrl+C to stop.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

When orders are detected:
```
Detected order event: ID=12345, Status=Filled, Action=Buy, Qty=1, Type=Market
Replicating order: Account=123456, Symbol=ESH4, Action=Buy, Qty=1, Type=Market, Status=Filled
Successfully placed order on follower Follower Account 1
Successfully placed order on follower Follower Account 2

[STATS] Orders: 1 | Success: 2 | Failed: 0 | Last Copy: 14:32:15
```

### Graceful Shutdown

Press `Ctrl+C` to stop:
```
Shutdown signal received. Cleaning up...
Stopping monitor...
Monitor stopped
Closing connection for account 234567
Closing connection for account 345678

=== Final Statistics ===
Total Orders Detected: 15
Successful Copies: 30
Failed Copies: 0

Shutdown complete. Goodbye!
```

## Testing with Demo Environment

For testing, use Tradovate's demo environment:

1. Create demo accounts at https://trader.tradovate.com/
2. Update config to use demo endpoints:
   - `auth_url`: `https://demo.tradovateapi.com/v1/auth/accesstokenrequest`
   - `websocket_url`: `wss://demo.tradovateapi.com/v1/websocket`

## Browser Login Balance Test

This repo includes a headless-browser test that logs into the Tradovate demo web platform (no API key) and then queries the account's cash balance using the captured session token.

- Configure `config.json` with your demo credentials. Optionally set either `leader_account.account_id` or `leader_account.account_spec` to select the target account.
- Run the test:

```
go run test_browser_balance.go browser_auth.go config.go
```

Notes:
- The browser automation uses Chrome via `chromedp`. It launches a visible browser (non-headless) to allow any CAPTCHA or additional challenges to be handled. After successful login, it captures the session token from network traffic.
- The test uses the demo API base `https://demo.tradovateapi.com/v1` to request the cash balance snapshot for the selected account.
- Do not commit real credentials. Consider using environment variables or a local, ignored config file for sensitive data.

## HTTP Login Balance Test (No Browser)

If you prefer not to automate a browser, use a direct HTTP session login and then query balance:

```
go run test_http_login_balance.go config.go auth.go
```

Tips:
- Ensure `config.json` uses demo endpoints:
  - `auth_url`: `https://demo.tradovateapi.com/v1/auth/accesstokenrequest`
  - `websocket_url`: `wss://demo.tradovateapi.com/v1/websocket`
- Set `tradovate_api.app_id` if empty. A generic label like `TraderWeb` is used by default.
- On some accounts, the auth endpoint may return a POW/CAPTCHA challenge (`p-ticket`). If that happens, use the browser-based method or log in once in a real browser to satisfy the challenge, then rerun.

## Balance From Existing Token

If you already logged in with the browser test, a token file is written to `tools/token.json`. You can reuse it (no new login) to query balances or compute accumulated profit per account.

Steps:

1) Capture token once (if not already):

```
node tools/playwright-login.js
```

2) Print balances for specific account IDs (comma-separated) using the same token:

```
$env:TV_ACCOUNT_IDS = "35565919,36595884,36595848"
go run test_balance_with_token_file.go config.go
```

3) Optional debug / baselines:

```
$env:TV_DEBUG = "1"                # show exact endpoints per account
$env:TV_BASELINES = "35565919=100000,36595884=100000,36595848=100000"  # override baseline per account
```

How it works:
- The script loads `tools/token.json` and calls the Tradovate demo API with the correct Authorization header.
- For each account ID: it fetches `cashBalance/list` and selects the most recent row by timestamp.
- Accumulated profit = `latest_amount - baseline`.
  - Baseline sources (in order): `TV_BASELINES` override → minimum amount found in history → default of `100000` for SIM/PA/APEX-style accounts.
- Some tokens return the same rows for any `accountId` filter. In that case the script retries using `accountSpec` derived from the account name (e.g., `PA-APEX-68036-14`, `APEX-68036-238`).
- Set `TV_DEBUG=1` to see per-account requests and short responses.

Example output (profits):

```
Using API base: https://demo.tradovateapi.com/v1
Found 5 account(s):
- APEX6803600000237 (ID: 36595848)
- PAAPEX680360000014 (ID: 35565919)
- APEX6803600000235 (ID: 36420065)
- APEX6803600000236 (ID: 36420117)
- APEX6803600000238 (ID: 36595884)

Balances (accumulated profit):
  PAAPEX680360000014 (ID 35565919): Profit $6530.18 (latest 106530.18 - baseline 100000.00)
  APEX6803600000238 (ID 36595884): Profit $778.90  (latest 100778.90 - baseline 100000.00)
  APEX6803600000237 (ID 36595848): Profit $758.90  (latest 100758.90 - baseline 100000.00)
```

This matches the browser’s “accumulated profit” view by normalizing out the standard initial funding baseline commonly used in demo/prop accounts.

## Architecture

The application consists of several key components:

- **auth.go**: Handles session authentication with Tradovate API
- **websocket.go**: WebSocket client with automatic reconnection and heartbeat
- **monitor.go**: Monitors leader account for order events
- **replicator.go**: Replicates orders to follower accounts in parallel
- **config.go**: Configuration management and validation
- **main.go**: Application entry point and orchestration

## Performance Considerations

- **Latency**: Typical replication latency is 500ms-2s depending on network conditions
- **Concurrent Execution**: Uses goroutines for parallel order placement
- **WebSocket Efficiency**: Maintains persistent connections to minimize overhead
- **Token Management**: Automatically refreshes tokens before expiration

## Limitations & Notes

1. **Grey Area**: This method bypasses Tradovate's $25/month API subscription by using session authentication. This may violate terms of service.

2. **Order Types Supported**: Market, Limit, Stop orders are supported. Complex order types may require additional implementation.

3. **No Historical Sync**: Only monitors new orders placed after the application starts.

4. **Account Sub-accounts**: If your Tradovate account has multiple sub-accounts (common with prop firms), ensure you specify the correct `account_id` for each.

5. **Risk Management**: The built-in risk filter is basic. Consider implementing additional safeguards for prop firm drawdown rules.

## Troubleshooting

**"Failed to authenticate"**
- Verify credentials are correct
- Check if account is active
- Ensure using correct API endpoint (live vs demo)

**"WebSocket connection failed"**
- Check firewall/proxy settings
- Verify WebSocket URL is correct
- Ensure network allows WSS connections

**"Orders not replicating"**
- Verify leader account ID is correct
- Check that orders are actually filling on leader account
- Review logs for error messages

**"Token expired"**
- Application should auto-refresh tokens
- If issues persist, restart the application

## Security Best Practices

1. **Never commit config.json** to version control (contains passwords)
2. Use environment variables for sensitive data in production
3. Restrict file permissions on config.json
4. Use separate passwords for each account if possible
5. Monitor application logs for suspicious activity

## Future Enhancements

Potential improvements:
- Web dashboard for monitoring
- Advanced risk management (trailing drawdowns, max position limits)
- Support for multiple leader accounts
- Order modification/cancellation replication
- Telegram/Discord notifications
- Database logging for audit trail

## License

This software is provided as-is for educational purposes. Use at your own risk.

## Disclaimer

This application is designed for educational and research purposes. Trading involves substantial risk of loss. The authors are not responsible for any financial losses incurred while using this software. Always test thoroughly with demo accounts before using with real money.

Prop firm traders should verify that automated trading and copy trading are permitted under their firm's rules before use.
## One-Command Bot

Run everything (auto-login + token reuse + accumulated profit print) with one command. Credentials can be supplied via flags to avoid editing `config.json`.

Examples:

```
# Auto-login with provided creds, then print profits for listed accounts
go run tv_bot.go config.go -username "BLU_USER_56R8BW_T" -password "ASn3#fV2" -accounts "35565919,36595884,36595848"

# Force a fresh login (ignores existing token.json)
go run tv_bot.go config.go -force-login -username "BLU_USER_56R8BW_T" -password "ASn3#fV2" -accounts "35565919,36595884,36595848"
```

Notes:
- The bot launches a headless Chromium to authenticate exactly like the web app (no API key), captures the session token, and saves it to `tools/token.json` for reuse.
- The bot then queries `cashBalance/list` per account and computes accumulated profit (latest amount minus baseline).
- Baselines are auto-inferred for SIM/PA/APEX accounts (defaults to `100000`). You can override with `TV_BASELINES` if needed.
- Omit `-accounts` to list all active accounts.

This single command is the foundation for adding “place” and “exit” order flows under the same entry point.
