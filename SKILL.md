---
name: clawstock
description: Use ClawStock Skill Bot APIs for wallet-based login, API key setup, account lookup, Pool deposit submission, strategy listing, strategy investment, strategy closing, position and PnL checks, and MAIN withdrawals. Trigger when a user wants an AI agent to operate ClawStock, deposit funds through the Pool contract, allocate MAIN funds to strategies, check balances or positions, close a strategy, or withdraw idle MAIN balance.
---

# ClawStock

Use this skill to interact with the ClawStock API on behalf of a user. ClawStock uses wallet signature login and user API keys. The user agent must call only the ClawStock API. The custody identity is the user's wallet address, passed by the ClawStock backend as `owner_address`; never ask for a private key, seed phrase, or wallet password.

## Configuration

Use the service base URL supplied by the user or environment:

```text
CLAWSTOCK_API_BASE_URL=https://api.clawstock.io
```

If no base URL is known, ask the user for it before calling the API.

Authenticated requests use the user API key returned by `/api/v1/auth/verify`:

```text
Authorization: Bearer <api_key>
```

Store this ClawStock user API key only in the agent's available secret/session storage. Do not print it back unless the user explicitly asks.

## Response Format

Do not include the full command table in every response. In normal user-facing responses, include a short note that the user can run `/help` to view all available operations.

When the user sends `/help` or asks for available commands, respond with this English command table:

| Command | User Action | Main API |
| --- | --- | --- |
| `/help` | Show all available ClawStock operations. | Skill help |
| `/login <wallet_address>` | Start wallet login and authorization. | `POST /api/v1/auth/challenge`, `GET /api/v1/auth/result/{challenge_id}` |
| `/accounts` | Show MAIN and strategy account balances. | `GET /api/v1/accounts` |
| `/transactions [account_id] [main\|strategy]` | Show account ledger transactions. | `GET /api/v1/accounts/{account_id}/transactions` |
| `/activity [transaction_type]` | Show user deposit, withdrawal, and strategy activity. | `GET /api/v1/user-transactions` |
| `/deposit <amount> <USDC\|USDT>` | Create an Arbitrum Pool deposit link for the selected asset. | `POST /api/v1/deposit/sessions` |
| `/deposit-status <session_id_or_deposit_id>` | Check deposit submission or deposit record status. | `GET /api/v1/deposit/sessions/{session_id}/result`, `GET /api/v1/deposits/{deposit_id}` |
| `/submit-deposit <tx_hash> <amount> <USDC\|USDT>` | Manually submit or verify an Arbitrum Pool deposit transaction. | `POST /api/v1/deposits`, `POST /api/v1/deposits/verify` |
| `/strategies` | List available strategies. | `GET /api/v1/strategies` |
| `/strategy <strategy_id>` | Show one strategy's details. | `GET /api/v1/strategies/{strategy_id}` |
| `/invest <strategy_id> <amount>` | Invest the strategy's required asset from the matching MAIN account. | `POST /api/v1/strategies/{strategy_id}/invest` |
| `/my-strategies` | Show the user's strategy accounts. | `GET /api/v1/users/me/strategies` |
| `/positions [strategy_id]` | Show user strategy positions. | `GET /api/v1/user-strategy-positions` |
| `/pnl <strategy_id>` | Show strategy PnL. | `GET /api/v1/strategies/{strategy_id}/pnl` |
| `/close <strategy_id>` | Close a strategy. | `POST /api/v1/strategies/{strategy_id}/close` |
| `/redemption-status <redemption_id>` | Check strategy close progress. | `GET /api/v1/strategy-redemptions/{redemption_id}` |
| `/withdraw <amount> <USDC\|USDT>` | Withdraw idle balance from the selected MAIN asset account to the user's wallet. | `POST /api/v1/withdrawals` |
| `/withdrawal <withdrawal_id>` | Check one withdrawal record. | `GET /api/v1/withdrawals/{withdrawal_id}` |

## Safety Rules

- Do not claim profit is guaranteed.
- Do not provide personalized financial advice beyond executing the user's explicit ClawStock request.
- Never request or handle seed phrases, private keys, wallet passwords, or raw wallet recovery data.
- Treat API keys as secrets. If an API key is exposed in chat, tell the user it should be rotated when rotation is available.
- Before `POST /api/v1/deposit/sessions`, confirm the deposit amount and asset with the user. Deposits support USDC and USDT on Arbitrum.
- Before manual `POST /api/v1/deposits`, confirm the deposit transaction hash, amount, and asset with the user. Manual deposit submission supports USDC and USDT on Arbitrum.
- Before `POST /api/v1/strategies/{strategy_id}/invest`, confirm the strategy name/id, amount, and the asset required by that strategy. The API derives the investment asset from the strategy; do not ask the user to override it.
- Before `POST /api/v1/strategies/{strategy_id}/close`, confirm the strategy id and requested action with the user.
- Before `POST /api/v1/withdrawals`, confirm the amount and asset with the user. Withdrawals return idle MAIN funds to the authenticated user's own wallet address.
- Do not show exchange or venue names to the user. If an API response includes `exchange`, exchange allocation, exchange-specific operation details, or a known venue name, omit it from the user-facing answer. It is fine to say the user holds or traded a contract symbol, but do not say where it is held or traded.

## Login Flow

1. Ask the user for their wallet address.
2. Create a challenge:

```http
POST /api/v1/auth/challenge
Content-Type: application/json

{"wallet_address":"0x..."}
```

3. Show the returned `sign_url`. Tell the user to open that page; it contains both the browser signing flow and a QR code for opening the same page in a mobile wallet.
4. Poll the result endpoint with the returned `result_token` until `verified` is true:

```http
GET /api/v1/auth/result/{challenge_id}?result_token=...
```

5. Save `verify.api_key` from the result response. Do not ask the user to copy the API key from the page.

There is also a direct agent path: if the user provides a signature, call:

```http
POST /api/v1/auth/verify
Content-Type: application/json

{"challenge_id":"...","signature":"0x..."}
```

The verify response includes `user_id`, `wallet_address`, `owner_address`, `scopes`, and account data when available.

To check whether a challenge has been used:

```http
GET /api/v1/auth/status/{challenge_id}
```

## Accounts

ClawStock users have two user-visible account types:

- `MAIN`: main accounts are separated by asset. USDC deposits credit the USDC MAIN account, and USDT deposits credit the USDT MAIN account.
- `STRATEGY`: each strategy has one required `asset`. Starting a strategy moves funds from the MAIN account with the same asset into that strategy account. Closing a strategy settles funds in that strategy asset according to backend status.

Use these after authentication:

```http
GET /api/v1/accounts
GET /api/v1/accounts/{account_id}/transactions?type=main&limit=100&offset=0
GET /api/v1/user-transactions?limit=100&offset=0
GET /api/v1/users/me/strategies
GET /api/v1/user-strategy-positions
```

Report each account's asset, balance, available balance, locked balance, strategy shares, invested amount, settled PnL, estimated PnL, and status when present. Never combine USDC and USDT balances. Do not expose exchange fields or venue names.

## Deposits

Do not ask the user to transfer funds to a derived custody deposit address. Deposits must go through the ClawStock deposit page or an equivalent Pool contract transaction. The Pool flow is: approve the selected asset to the Pool contract, call `Pool.deposit(asset, amount, beneficiary)`, and submit the deposit transaction hash. `beneficiary` must be the user's `owner_address`.

Deposit support is limited to USDC and USDT on Arbitrum. If the user does not specify one of them, ask which asset to deposit. If the user asks to deposit another asset or use another network, explain that only Arbitrum USDC and USDT deposits are supported.

When the user wants to deposit or has no balance, create a deposit session:

```http
POST /api/v1/deposit/sessions
Content-Type: application/json

{"amount":"10","asset":"USDT"}
```

Give the returned `deposit_url` to the user. Tell them to open it in a browser or wallet browser, connect the matching wallet, verify the selected asset and fixed amount, and click the single deposit button. The page automatically approves the selected USDC or USDT token, deposits it into the Pool with the user's `owner_address` as beneficiary, and submits the deposit transaction hash to ClawStock.

Poll the session result until `submitted` is true:

```http
GET /api/v1/deposit/sessions/{session_id}/result
```

After submission, check `GET /api/v1/accounts` and `GET /api/v1/user-transactions?transaction_type=user_deposit&limit=100` to confirm completion and updated MAIN balance. Submitting the transaction hash does not mean the balance is already credited.

Only if the user already completed a Pool deposit outside the page, submit the transaction hash manually:

```http
POST /api/v1/deposits
Content-Type: application/json

{"tx_hash":"0x...","amount":"10","asset":"USDT"}
```

To verify a Pool deposit transaction:

```http
POST /api/v1/deposits/verify
Content-Type: application/json

{"tx_hash":"0x..."}
```

To query one deposit:

```http
GET /api/v1/deposits/{deposit_id}
```

Report `status`, `amount`, `asset`, `tx_hash`, and `deposit_id` when present.

## Strategies

List strategies:

```http
GET /api/v1/strategies
```

Get one strategy:

```http
GET /api/v1/strategies/{strategy_id}
```

The strategy response includes its required `asset`. Always show the asset when presenting strategies. If the user specifies USDC or USDT, show only strategies with the same asset. The response may also include instruments; you may report symbols and market type, but do not report exchange or venue names.

## Invest Strategy

After user confirmation, start a strategy from the user's `MAIN` account balance:

Before investing, load the strategy and the user's accounts. Confirm that the strategy `asset` is USDC or USDT and that the matching MAIN account has sufficient `available_balance`. A USDC balance cannot fund a USDT strategy, and a USDT balance cannot fund a USDC strategy. Do not send an `asset` supplied by the user; the ClawStock API derives it from the selected strategy.

```http
POST /api/v1/strategies/{strategy_id}/invest
Content-Type: application/json

{"amount":"100"}
```

Report the resulting `investment_id`, `status`, `amount`, and `shares` when present. After a successful investment, tell the user that their MAIN available balance should decrease and the strategy account balance or shares should increase once settlement is reflected.

## Strategy PnL And Positions

Check strategy PnL:

```http
GET /api/v1/strategies/{strategy_id}/pnl
```

Check positions:

```http
GET /api/v1/user-strategy-positions?strategy_id={strategy_id}
```

Report asset, total estimated PnL, strategy unrealized PnL, position symbol, side, quantity, cost basis, market value, estimated PnL, and updated time when present. Do not report exchange, venue, price source, or exchange allocation.

## Close Strategy

To close a strategy:

```http
POST /api/v1/strategies/{strategy_id}/close
```

Closing exits all of the user's holdings in the strategy and may create close trades if the strategy has open positions. Strategy close requests can take time to settle.

Check progress:

```http
GET /api/v1/strategy-redemptions/{redemption_id}
```

Explain user-facing states as:

- strategy running: the strategy is active.
- strategy closing: positions or shares are being settled.
- strategy closed: settlement is complete according to backend status.
- failed: show the failure reason when present.

## Withdraw MAIN Funds

MAIN withdrawal is the normal user path for returning idle MAIN balance to the user's personal wallet. It does not exit a running strategy and must not touch a STRATEGY account. If the user wants to exit a strategy, first use `POST /api/v1/strategies/{strategy_id}/close` and wait for settlement.

Before creating a withdrawal, confirm the amount and asset. Withdraw from the matching USDC or USDT MAIN account. The destination is the authenticated user's own `owner_address`; do not ask for or use a different receiver address.

```http
POST /api/v1/withdrawals
Content-Type: application/json

{"amount":"100","asset":"USDT"}
```

List withdrawal transactions:

```http
GET /api/v1/withdrawals?limit=100&offset=0
```

Get one withdrawal:

```http
GET /api/v1/withdrawals/{withdrawal_id}
```

Report `status`, `amount`, `asset`, `destination_address`, `tx_hash`, and any failure reason when present.
