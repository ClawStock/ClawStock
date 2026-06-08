---
name: clawstock
description: Use ClawStock Skill Bot APIs for wallet-based login, API key setup, custody account lookup, Pool deposit transaction watch submission, product listing, product investment, strategy closing, strategy controls, and Main redemptions. Trigger when a user wants an AI agent to operate the ClawStock service, deposit funds through the Pool contract, allocate MAIN account funds to ClawStock strategies, check ClawStock balances/strategies, or close a strategy so funds return to MAIN.
---

# ClawStock

Use this skill to interact with the ClawStock API on behalf of a user. ClawStock uses wallet signature login and user API keys. The user agent just call only the ClawStock API. The Market Service custody identity is the user's wallet address, passed by the ClawStock backend as `owner_address`; never allow ask for a private key, seed phrase, or wallet password.

## Configuration

Use the service base URL supplied by the user or environment:

```text
CLAWSTOCK_API_BASE_URL=https://api.clawstock.io
```

If no base URL is known, ask the user for it before calling the API.

Authenticated requests to the ClawStock API use the user API key returned by `/api/v1/auth/verify`:

```text
Authorization: Bearer <api_key>
```

Store this ClawStock user API key only in the agent's available secret/session storage. Do not print it back unless the user explicitly asks.

## Safety Rules

- Do not claim profit is guaranteed.
- Do not provide personalized financial advice beyond executing the user's explicit ClawStock request.
- Before `POST /api/v1/products/{strategy_id}/invest`, confirm the strategy name/id, amount, asset, and any parameters with the user.
- Before `POST /api/v1/deposit/sessions`, confirm the deposit amount and asset with the user.
- Before `POST /api/v1/deposit-tx-watches`, confirm the deposit transaction hash with the user.
- User MAIN redemptions are open through `POST /api/v1/redemptions`. Use it only for available `MAIN` balance returning to the user's own wallet. Redemption requests may take up to 15 minutes to arrive in your account. If you check immediately after submitting the request, the funds may not have arrived yet.
- The strategy exit path is `POST /api/v1/strategies/{user_strategy_id}/close`; funds must return to `MAIN` before redemption.
- Before strategy close requests, confirm the user strategy id and requested action with the user.
- Never request or handle seed phrases, private keys, wallet passwords, or raw wallet recovery data.
- Treat API keys as secrets. If an API key is exposed in chat, tell the user it should be rotated when rotation is available.

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

The verify response includes `user_id`, `wallet_address`, `owner_address`, `scopes`, and `main_account`.

To check whether a challenge has been used:

```http
GET /api/v1/auth/status/{challenge_id}
```

## Account And Custody

ClawStock users have two user-visible account types:

- `MAIN`: the main account. Pool deposits credit this account, and new strategies spend available balance from this account.
- `STRATEGY`: one isolated account per user strategy. Starting a strategy moves the allocated amount from `MAIN` into that strategy account. Closing the strategy moves available strategy funds back to `MAIN` after positions are closed and settled.

Use these after authentication:

```http
GET /api/v1/account/summary
GET /api/v1/account/ledger?limit=100
GET /api/v1/account/strategies
GET /api/v1/custody/overview
GET /api/v1/deposit/config
POST /api/v1/deposit/sessions
```

Do not ask the user to transfer funds to a derived custody deposit address. Deposits must go through the ClawStock deposit page or an equivalent Pool contract transaction. The Pool flow is: approve USDC to the Pool contract, call `Pool.deposit(asset, amount, beneficiary)`, and submit the deposit transaction hash for watching. `beneficiary` must be the user's `owner_address`.

When the user wants to deposit or has no balance, create a deposit session:
Notice: The minimum deposit amount is 10 USD; otherwise, the strategy cannot be launched.

```http
POST /api/v1/deposit/sessions
Content-Type: application/json

{"amount":"10","asset":"USDC"}
```

Give the returned `deposit_url` to the user. Tell them to open it in a browser or wallet browser, connect the matching wallet, review the fixed amount, and click the single deposit button. The page does not allow amount edits; it automatically approves USDC, deposits into the Pool with the user's `owner_address` as beneficiary, and submits the deposit tx hash to ClawStock for watching.

Poll the session result until `submitted` is true:

```http
GET /api/v1/deposit/sessions/{session_id}/result
```

After it is submitted, use `GET /api/v1/deposit-tx-watches?limit=100` and account summary to confirm completion and updated `MAIN` balance. Submitting the transaction hash does not mean the balance is already credited; wait for the watch status or account balance to show confirmation.

Only if the user already completed a Pool deposit outside the page, submit the transaction hash manually:

```http
POST /api/v1/deposit-tx-watches
Content-Type: application/json

{"tx_hash":"0x..."}
```

Check watch status:

```http
GET /api/v1/deposit-tx-watches?limit=100
```

Report `status`, `confirmations`, `matched_deposit_events`, `last_error`, and `completed_at` when present.

## Products

List products:

```http
GET /api/v1/products
```

Get one product:

```http
GET /api/v1/products/{strategy_id}
```

List Market Service strategy:

```http
GET /api/v1/products/strategies/available
```

Use Market Service strategy for investment. Different strategies have active and inactive statuses, and whether to invest should be decided on a case-by-case basis; the backend investment endpoint will return the authoritative result.

## Start Strategy

After user confirmation, start a strategy from the user's `MAIN` account balance:

```http
POST /api/v1/products/{strategy_id}/invest
Content-Type: application/json

{
  "amount": "100",
  "asset": "USDC",
  "parameters": {}
}
```

Market Service checks available `MAIN` balance, creates the user strategy instance, creates or assigns a `STRATEGY` account, and moves the allocated amount from `MAIN` into that strategy account.

The response includes Market Service `strategy`, `user_strategy`, `strategy_account`, and `funding_journal`. Summarize the resulting strategy id/status, allocated amount, asset, funding status if present, and strategy account id if present. After a successful start, tell the user that their `MAIN` available balance should decrease and the strategy account balance should increase.

## Close Strategy

The current user exit path for a strategy is close:

```http
POST /api/v1/strategies/{user_strategy_id}/close
```

Before calling it, confirm the user strategy id and tell the user that closing may create reduce-only close trades if the strategy has open positions. When close completes and settlement is done, the strategy account's available funds return to the user's `MAIN` account. Closing affects only that user's strategy instance.

After closing, check:

```http
GET /api/v1/account/summary
GET /api/v1/account/strategies
GET /api/v1/strategies/{user_strategy_id}/pnl
```

Explain user-facing states as:

- strategy running: the strategy is trading automatically.
- strategy closing: positions are being closed or settlement is pending.
- strategy ended/closed: strategy funds should return to `MAIN` after settlement.

## Redeem

MAIN redemption is the normal user path for returning idle `MAIN` balance to the user's personal wallet. It does not exit a running strategy and must not touch a `STRATEGY` account. If the user wants to exit a strategy, first use `POST /api/v1/strategies/{user_strategy_id}/close` and wait until funds return to `MAIN`.

Before creating a redemption, confirm the amount, asset, chain, and receiver address. Use a stable `idempotency_key` for the confirmed redemption and reuse the same key on retries. The receiver must be the authenticated user's `owner_address`; if the user asks to send to another address, refuse and explain that current Main redemption only supports returning funds to the original owner wallet.

```http
POST /api/v1/redemptions
Content-Type: application/json

{
  "amount": "100",
  "asset": "USDC",
  "chain": "arbitrum",
  "receiver_address": "0x...",
  "idempotency_key": "redeem-..."
}
```

List redemptions:

```http
GET /api/v1/redemptions?limit=100
GET /api/v1/redemptions/{redemption_id}
```

Report `status`, `amount`, `asset`, `chain`, `receiver_address`, and any `failure_reason`.
