---
name: clawstock
description: Use the ClawStock v1 backend API for public strategy discovery and wallet-authenticated account, investment, deposit, close, withdrawal, and performance operations. Distinguish public catalog data from the user's own strategy accounts.
---

# ClawStock

Use this skill for ClawStock requests. The service uses wallet signature login and a user API key. Never request a private key, seed phrase, wallet password, or recovery data. A wallet address alone does not grant access to a user's accounts.

## Endpoints and response handling

- Agent HTTP calls go directly to the backend base URL `https://api.clawstock.io`. Append the relative `/api/v1/...` path once. For example, list strategies with `GET https://api.clawstock.io/api/v1/strategies`.
- `https://clawstock.io` is the human-facing website. Its `/api/clawstock` path is a browser same-origin proxy, **not** a path to append to the backend URL. Use the website only for the human wallet-signing page or other user-facing pages.
- Agent API calls omit the optional language query parameter. Use the response content as returned by the backend.
- Normal responses are `{ "code": 0, "msg": "ok", "data": ... }`. Some deployments may use `code=200` or an unwrapped response. Check both HTTP status and business `code`; do not treat an HTTP 2xx response with another business code as success. Prefer `detail`, then `msg` for errors.
- Money, units, NAV, and fees are generally decimal strings. Preserve their precision; do not treat them as token base-unit integers.
- Send `Authorization: Bearer <api_key>` only for endpoints requiring it. Do not rely on `X-API-Key`; the website proxy does not forward that header. Never print or persist the API key in a public file, chat message, log, or URL. Use available secret/session storage only.

## Public requests: no login needed

| Purpose | Request | Notes |
| --- | --- | --- |
| Strategy catalog | `GET /api/v1/strategies` | Parse `data` as an array or `strategies`/`items`; show `strategy_id`, name, description, `status`, `asset`, and minimum investment when present. A listed strategy is not the user's holding. Only `status=active` is treated as subscribable. |
| Strategy detail and backtest | `GET /api/v1/strategies/{strategy_id}` | Detail may contain `backtest.files[].tables[]`; it can be large. Retrieve only when needed and summarize instead of dumping all rows. |
| Chains and contracts | `GET /api/v1/chains` | Read enabled chains, Pool address, token addresses, decimals, and chain IDs from the response. Do not hard-code a single chain or contract. |
| Fees and minimums | `GET /api/v1/fees` | Match an enabled fee entry to the selected `chain` and `asset`; use its current minimums and fees. |

For a request such as “what strategies are available?”, use the public catalog immediately. For “my strategies,” “my balance,” or “my PnL,” use the authenticated user endpoints below. Never present the public catalog as the user's portfolio.

## Wallet login

1. Obtain the user's public EVM wallet address, or reuse one they supplied. `POST /api/v1/auth/challenge` with `{"wallet_address":"0x..."}`; no API key is needed. Save the returned `challenge_id` and `result_token` in session state.
2. The backend's `sign_url` may point to an internal or loopback host. For the user-facing signing step, construct the **website** page from the documented route: `https://clawstock.io/{locale}/wallet/sign?challenge_id={url_encoded_challenge_id}&result_token={url_encoded_result_token}`, where `locale` is `zh` or `en`. If a working, trusted public `sign_url` is returned, it may also be used.
3. Tell the user to check that the connected wallet address matches the challenge address and sign in their wallet. The page obtains the exact challenge `message` using `GET /api/v1/auth/challenge/{challenge_id}` and submits `POST /api/v1/auth/verify` with `{"challenge_id":"...","signature":"0x..."}`. Never reconstruct or alter the message, and never ask for wallet secrets.
4. Query `GET /api/v1/auth/result/{challenge_id}?result_token={url_encoded_result_token}` without an API key until `data.verified=true`, with a reasonable time limit and without excessive requests. On success, keep `data.verify.api_key`, `owner_address`, and scopes in secret/session state. If the challenge expires, start a new challenge. A direct signature supplied by the user can instead be submitted to `/api/v1/auth/verify`.
5. The website's local sign-out does not prove that the backend key has been revoked. Do not claim server-side revocation without an API response confirming it.

## Authenticated read requests

Use Bearer authentication and the `owner_address` returned by login for owner-scoped paths. Do not derive another user's owner address from a public wallet address.

| Purpose | Request | Main response/use |
| --- | --- | --- |
| Main and strategy accounts | `GET /api/v1/accounts` | `main_accounts`, `strategy_accounts`; amounts may be flat or nested in `balance`. Keep each asset separate. |
| User activity | `GET /api/v1/user-transactions?limit={n}&offset={n}&transaction_type={type}` | Optional filters. Common types: `user_deposit`, `user_withdrawal`, `strategy_trade`. |
| My strategy accounts | `GET /api/v1/users/{owner_address}/strategies` | `data` array or `strategies`/`strategy_accounts`/`items`; report names, IDs, asset, status, balance, shares, invested amount, and PnL when present. |
| My strategy performance | `GET /api/v1/users/{owner_address}/strategies/{strategy_id}/performance` | Report available units, current value, realized/unrealized/total PnL, `as_of`, and `stale` when present. Do not guess the unit of `total_return_rate`. |
| My investments | `GET /api/v1/users/{owner_address}/strategy-investments?limit={n}&offset={n}` | Investment history and progress. |
| One investment | `GET /api/v1/users/{owner_address}/strategy-investments/{investment_id}` | `investment`, `stage`, `operations`. Treat active only when `stage` or investment status is `active` **and** shares are positive. |
| One deposit | `GET /api/v1/deposits/{deposit_id}` | Status and credited amount. |
| One close/redemption | `GET /api/v1/strategy-redemptions/{redemption_id}` | `redemption`, operations, possibly a withdrawal. A completed close alone does not prove a wallet withdrawal occurred. |
| One withdrawal | `GET /api/v1/withdrawals/{withdrawal_id}` | Status, amount, fee, destination, chain, transaction hash when present. A hash alone does not prove final on-chain confirmation. |

Additional documented but not currently used by the website: `GET /api/v1/strategies/{strategy_id}/pnl`, `GET /api/v1/user-strategy-positions?strategy_id={strategy_id}`, `GET /api/v1/users/{owner_address}/strategy-ledger?...`, and `GET /api/v1/withdrawals?limit={n}&offset={n}`. Use only when the user requests their specific data, and handle absent or changed fields gracefully. Old `GET /api/v1/accounts/{account_id}/transactions`, `GET /api/v1/auth/status/{challenge_id}`, `GET /api/v1/auth/me`, and `POST /api/v1/auth/logout` are historical references, not confirmed current flows.

## Fund-moving operations

Only act on the user's explicit request with concrete parameters. Before a fund-moving POST, show the current strategy/asset/chain/amount or share scope and get confirmation if those details were not already explicitly authorized. Never claim a request was settled merely because its POST succeeded. Report identifiers and follow the relevant status endpoint. Never promise profit.

### Deposit

When the user asks to deposit or recharge, **start the deposit workflow**. Do not merely send them to the ClawStock homepage. First call the public `GET /api/v1/chains` and `GET /api/v1/fees`; intersect chains with `deposits_enabled=true`, an asset token address and decimals, and an enabled fee entry for that chain/asset. Show every eligible chain with its current minimum and fee. If more than one chain is eligible and the user has not selected one, ask them to choose; do not silently pick the first response item or infer a default from examples. Ask for any other missing amount or asset after this lookup. The current website uses USDT, but supported assets must still come from the live responses.

There is no deposit-creation HTTP request in the current website flow. The wallet must make an on-chain Pool transaction before the backend can register it. Once amount, asset, and chain are known, check the amount against the current minimum and give the user a **specific deposit page link**, not a generic website link:

```text
https://clawstock.io/{locale}/wallet/deposit?amount={url_encoded_amount}&asset={url_encoded_asset}&chain={url_encoded_chain}
```

`locale` is a website page locale (`zh` or `en`), unrelated to Agent API query parameters. Tell the user to connect the same wallet used for ClawStock login, check the chain/asset/amount and fee, and confirm the wallet transactions. **After the Pool transaction succeeds, its transaction hash must be submitted to the ClawStock backend; the deposit is not complete merely because the wallet shows a successful transaction.** The current website deposit page obtains that hash and calls the registration and verification APIs automatically. Tell the user to wait for its submission result. The beneficiary must equal the authenticated `owner_address`. Do not ask the user to send tokens to a derived custody address.

If the deposit was made outside that page, or the page did not successfully submit its hash, obtain the Pool **deposit transaction hash** from the wallet receipt or from the user. Do not use the separate token-approval transaction hash. Check the selected chain, asset, and amount against the receipt and the user's request. With an authenticated session, submit the hash using the API below. If an authorized wallet tool can execute the chain transaction directly, use the public chain configuration rather than a hard-coded contract, capture its Pool deposit transaction hash, and then submit it to the backend.

Register an unsubmitted successful Pool transaction with authenticated `POST /api/v1/deposits`:

```json
{"tx_hash":"0x...","amount":"<confirmed_decimal_amount>","asset":"<selected_asset>","chain":"<selected_chain>"}
```

Then call `POST /api/v1/deposits/verify` with `{"tx_hash":"0x...","chain":"<selected_chain>"}` and poll `GET /api/v1/deposits/{deposit_id}`. The `chain` field is required in the current client; use the same user-selected chain in both requests. If the website already submitted the hash, do not submit a duplicate registration; use its `deposit_id` or authenticated activity/account reads to check status. If the website leaves submission pending or failed, ask the user for the Pool deposit hash if it is not available to the Agent, then register or verify it. With a `deposit_id`, check the record and refresh `GET /api/v1/accounts` to confirm MAIN credit. Without a transaction hash or deposit ID, use authenticated account/activity reads to check whether credit appears; do not claim verification from the page alone. A submitted hash or registration response is not evidence that MAIN has been credited. The old `POST /api/v1/deposit/sessions` flow is not part of the current website integration; do not use it by default.

### Invest

Fetch the public strategy detail and the authenticated accounts. Confirm `status=active`, the strategy's required `asset`, its minimum when present, and sufficient available MAIN balance in that same asset. `POST /api/v1/strategies/{strategy_id}/invest` with a decimal-string `amount`, the matching `asset`, and `"mark_active":false` (the current website request form). Do not substitute an asset chosen independently of the strategy. Record `investment_id`, then use the owner-scoped investment progress endpoint. An accepted investment is not yet an active holding.

### Close

`POST /api/v1/strategies/{strategy_id}/close` closes the user's full strategy holding. Send `{"asset":"USDT"}` when the strategy asset is known, or `{}` otherwise; do not invent a partial amount or `auto_withdraw`. Record `redemption_id` and check `GET /api/v1/strategy-redemptions/{redemption_id}`. The website treats `redemption.status=redeemed` together with `close_status=completed` as close completion. Check any returned withdrawal separately before saying funds arrived in the wallet. Do not assume automatic withdrawal or that funds always return to MAIN.

### Withdraw

Withdraw only available MAIN balance in the same asset, to the authenticated user's own wallet. Check an enabled chain/asset fee entry and minimum first. `POST /api/v1/withdrawals` with decimal-string `amount`, `asset`, and `chain`; do not submit a different `destination_address`. Record `withdrawal_id` and follow `GET /api/v1/withdrawals/{withdrawal_id}`. Distinguish accepted, processing, transaction broadcast, and confirmed states from actual response data.

## Reporting and compatibility

- Show asset alongside every balance and strategy amount. Do not add USDT and USDC totals together.
- Do not expose exchange or venue names, exchange allocations, price sources, or exchange-specific operating details in user-facing answers. Symbols and market types may be shown. Structured backtest rows may contain venue fields; omit those too.
- Response fields may be optional. Do not make up unavailable values or state transitions. For progress, use the operation-specific completion rules above rather than a generic success-word substring.
- In normal replies, a brief note may say `/help` lists available operations. If the user asks for `/help`, list supported public catalog, login, account, activity, my strategies, investment status, deposit, invest, close, and withdrawal operations. Do not advertise historical endpoints as supported commands.
