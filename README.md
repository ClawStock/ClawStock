# ClawStock
An AI skill that unleashes autonomous crypto portfolio power

## Skill

This is the Skill file repository for ClawStock, designed for AI Agents. The core open-source content is [`skill/SKILL.md`](skill/SKILL.md).

This Skill is used to guide Agents that support `SKILL.md` in calling the ClawStock Skill Bot API to complete operations such as wallet login, API Key retrieval, account queries, Pool deposits, product listing, strategy subscriptions, strategy closing, and MAIN account redemption.

## Repository Contents

```text
skill/
  SKILL.md   # Main ClawStock Agent Skill file
README.md   # Project documentation
```

## Use Cases

This Skill is suitable for scenarios where an AI Agent needs to operate the ClawStock service on behalf of a user, including:

- Completing ClawStock login through wallet signing.
- Saving and using the ClawStock user API Key.
- Querying MAIN account balances, strategy accounts, and ledger records.
- Creating Pool deposit sessions and tracking deposit transactions.
- Viewing available ClawStock products and Market Service strategies.
- After user confirmation, investing MAIN account funds into a specified strategy.
- After user confirmation, closing a user strategy and waiting for funds to return to MAIN.
- Redeeming idle MAIN balances back to the user’s own wallet address.

## Usage

1. Place the `skill/` directory into an Agent environment that supports custom Skills.
2. Ensure that the Agent can read `skill/SKILL.md`.
3. Configure the ClawStock API base URL for the Agent:

```text
CLAWSTOCK_API_BASE_URL=https://your-clawstock-api.example.com
```

4. Start a ClawStock-related request in the Agent session, for example:

```text
Help me log in to ClawStock and check my MAIN balance.
```

## Login Flow Overview

The Skill requires the Agent to use wallet-signature login instead of asking for sensitive credentials:

1. The user provides a wallet address.
2. The Agent calls `/api/v1/auth/challenge` to create a login challenge.
3. The Agent gives the `sign_url` returned by the backend to the user to open.
4. The user completes signing on the web page or in a mobile wallet.
5. The Agent polls `/api/v1/auth/result/{challenge_id}`.
6. After login verification succeeds, the Agent saves the returned `api_key` for subsequent API requests.

Authenticated API requests use:

```text
Authorization: Bearer <api_key>
```

## Safety Rules

`SKILL.md` explicitly requires the Agent to observe the following boundaries:

- Do not request or process the user’s private key, mnemonic phrase, wallet password, or recovery phrase.
- Do not promise returns, and do not describe products as principal-protected or guaranteed-profit.
- Do not provide personalized financial advice beyond the scope of the user’s explicit request.
- Before investing, depositing, submitting a deposit transaction hash, closing a strategy, or redeeming funds, the Agent must confirm key parameters with the user.
- MAIN redemptions may only be returned to the authenticated user’s own `owner_address`.
- To exit a strategy, the strategy should be closed first; after funds return to MAIN, a MAIN redemption may then be initiated.
- If an API Key is exposed in chat, the Agent should tell the user to rotate the key when rotation becomes available.

## Account Model

ClawStock mainly exposes two types of accounts on the user side:

- `MAIN`: The user’s main account. Pool deposits enter this account, and new strategies transfer funds out of this account.
- `STRATEGY`: An isolated strategy account corresponding to each user strategy instance. When a strategy starts, funds are transferred from `MAIN` into this account; after the strategy is closed and settled, funds return to `MAIN`.

The Skill requires the Agent to follow this model when explaining balance changes, strategy starts, and strategy closures.

## Disclaimer

This project only provides documentation for AI Agents to operate the ClawStock API. It does not constitute investment advice, a promise of returns, or a description of custody services. Users should independently confirm the risks and outcomes of transactions, deposits, investments, strategy closures, and redemptions.
