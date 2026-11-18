---
name: loyal-spark
description: Onchain loyalty protocol on Base. AI agents create native B20 Asset programs by default (legacy ERC-20 optional), mint/transfer points, manage rewards and vouchers, redeem LOYAL-XXXXXX gift certificates, and trade on P2P escrow via MCP or REST. Returns ready-to-sign calldata for Base Account, CDP MPC, or any user wallet.
version: 0.2.0
homepage: https://loyalspark.online
license: MIT
---

# Loyal Spark Skill

> [!IMPORTANT]
> Run the short onboarding below at the start of every conversation that touches Loyal Spark, including conversations that jump straight to a single tool. Onboarding is two sentences plus one disclaimer.

## Detection

The Loyal Spark MCP exposes its tools to the harness when connected. If no `get_platform_info`, `list_loyalty_programs`, or `create_loyalty_program` tool is callable, the MCP server is not installed — direct the user to [references/install.md](https://loyalspark.online/skills/loyal-spark/references/install.md) and stop.

If Loyal Spark tools are available, load [references/tone.md](https://loyalspark.online/skills/loyal-spark/references/tone.md) and continue to Onboarding. All links are absolute, so they work from `https://loyalspark.online/skill.md` or any copy. Load references lazily only when the conversation needs them.

## Onboarding

1. **Briefly mention what is available** — one sentence. The user can create native B20 loyalty programs on Base (or legacy ERC-20 through the API), mint and transfer points (by `0x` wallet address or customer email — resolving registered emails to Base wallets or crediting virtual points in `store_points_ledger` for unregistered emails that auto-convert to B20 tokens upon signup), manage rewards and vouchers, issue and redeem `LOYAL-XXXXXX` gift certificates, and trade tokens on a P2P escrow marketplace.

2. **Show this disclaimer verbatim before any write:**

   > By using Loyal Spark you agree to the Loyal Spark Terms (https://loyalspark.online/legal/terms) and Privacy Policy (https://loyalspark.online/legal/privacy). Loyal Spark prepares onchain calldata; the connected wallet (Base Account, CDP MPC, or a user wallet) is what actually signs and broadcasts transactions on Base mainnet (chain 8453).

3. **Identity is optional up front.** Only call profile tools when the user asks who they are or when a flow needs the bound wallet address.

## Agent rules: probes, 400 before 402, payment headers, next_steps

- **Probe vs. bad input.** An unpaid request with no input (no query params other than `?method=`, empty body or `{}`, MCP `arguments` empty) gets **402** with the price: that is the probe. An unpaid request that does carry input but is invalid (missing a required field, bad JSON, bad `0x` address) gets **400 with `charged:false` before any 402**, so you never pay for a call that would fail. A paid call that fails the same check is 400 and is not settled.
- **x402 headers.** Send the signed payload in `PAYMENT-SIGNATURE` (x402 v2). `X-PAYMENT` is the legacy v1 name and still works. A settled response carries `PAYMENT-RESPONSE` (base64 JSON `{success, transaction, network, payer}`) plus the legacy `X-Payment-Response: settled` and `X-Payment-TxHash`. Any response with `charged:false` was not settled.
- **MPP.** `https://api.loyalspark.online/mpp-gateway/<route>` answers `WWW-Authenticate: Payment`. Pay on Tempo (chain 4217, pathUSD or USDC) and resend with `Authorization: Payment <credential>` (`npm install -g mppx && mppx account create && mppx <url>`). USDC on Base is x402 only.
- **MPP is REST-only.** `Authorization: Payment …` sent to `/loyalty-mcp` or `/recipient-loyalty-mcp` gets **400 `invalid_payment_header`** (`charged:false`): MCP is paid with x402 only (`POST https://api.loyalspark.online/x402-gateway/mcp-tools/<tool>` or `/x402-gateway/recipient-mcp-tools/<tool>` with `PAYMENT-SIGNATURE`), or call the MCP server with your `lsk_`/`rwk_` key and no payment header. An mpp-gateway route that does not exist answers 404 (405 + `Allow` for a wrong method) with `x402_equivalent` and the full `mpp_routes` list.
- **`bazaar_probe_x402` is keyless.** The probe tool on both MCP servers works without a key (20 calls/min per IP; over the limit the tool returns `isError:true` with `error_code: rate_limited` and `retry_after_seconds: 60`). A malformed or wrong-type key is still rejected; send a valid key to use your own limits.
- **`?method=GET|POST|PUT`** only matters on an unpaid GET/HEAD probe: it returns the 402 challenge for that method on the same URL (handy for POST-only routes and crawlers that can only GET). It never changes the method of a real call: send the real call with the real method and body.
- **`POST /program-status` vs `GET /workflow/program-status`.** `POST /program-status` changes a program's status (`{token_address, status}`, $0.005, needs `lsk_`). `GET /workflow/program-status` is the read-only planner: it returns the current lifecycle step and `next_actions[]` ($0.001, payment alone works).
- **Required inputs on paid reads.** `GET /rewards` needs `token_address` (or `all=true` for every program you own). `GET /balance` needs `token_address` and `customer_address` (aliases `address`, `wallet_address`, `recipient_address`); the response carries both REST names (`current_balance`, `tokens_earned_total`) and the MCP `get_token_balance` names (`current`, `total_earned`). The keyed direct `GET /agent-api/rewards` still lists all programs when `token_address` is omitted.
- **Keyless recipient lookup.** `GET https://api.loyalspark.online/x402-gateway/resolve-recipient?recipient_email=...` ($0.001, x402, no key): the paying wallet is the merchant. Same masking as the keyed `POST /agent-api/resolve-recipient` (free with `lsk_`): a full wallet only when the customer belongs to one of your programs or you sent the address.
- **Free keyed create vs $0.01 create.** With an `lsk_` key, `POST https://api.loyalspark.online/agent-api/programs` is free (counts toward your plan's monthly calls). With no key, the cheapest path is $0.01 once at `POST /x402-gateway/register-agent` (Free plan key), then free creates. `POST /x402-gateway/programs` ($0.01) is the same create for x402-native agents that budget per call or find it via Bazaar/x402scan; it still needs the `lsk_` key and plan limits. Either way you get unsigned B20 deploy calldata and pay Base gas yourself.
- **Per-call x402/MPP payments never use plan quota.** A call paid through `/x402-gateway` or `/mpp-gateway` is billed per call only: it never counts toward, and is never blocked by, your plan's monthly API calls, even when you send your `lsk_`/`rwk_` key with it. The per-minute rate limit still applies. Direct keyed calls use the monthly plan; `rwk_` keys have their own monthly bucket, separate from the same wallet's `lsk_` keys. `GET /agent-api/me` and `GET /recipient-api/me` never use quota, so you can always check which key you hold. A keyed call without payment that gets `429` with `detail: monthly_quota` will not clear in a minute: `next_steps.action` is `upgrade_plan_or_pay_per_call` with `quota_resets_at`; pay per call at the gateway URL, or upgrade.
- **Failed paid calls are not charged.** x402: the gateway verifies the payment, runs the call, and settles only when it succeeded (2xx and not an error payload, including MCP `isError:true`). A failure comes back with `charged:false` and no `PAYMENT-RESPONSE`. Writes that cannot be undone (reward redeem, voucher use, offer accept/cancel, certificate claim/revoke/mark-minted, report delete) first run as a dry run (every check, no write), then settle, then run for real; only a state change in between can fail after settlement, and that answer carries `charged:true` and `payment_tx`. A no-op write is HTTP 409 `no_change` and is not charged.
- **`payment_already_used` (HTTP 409, `charged:false`).** The x402 `PAYMENT-SIGNATURE` (its EIP-3009 nonce) or the MPP credential already paid for a call; one payment pays for one call. Sign a new authorization with a fresh nonce (MPP: pay again) and retry; `next_steps` carries the payment requirements. Nothing was charged the second time.
- **MPP charges first, so it checks first.** Before the charge, MPP runs the input check (400 `charged:false`), a preflight of key scope, program ownership and the per-minute limit, and, for the keyed writes above plus `POST activate-program` and `POST program-status`, the same dry run. A call that still fails after an MPP charge (the state changed in between, or a route that accepts payment alone and so learns the caller only from the charge, such as reads and recipient-api routes) returns its error with `charged:true` and a `charge_note`: contact support for a refund. Use x402 when a charged failure is not acceptable.
- **Next steps stay on your gateway.** Through `/x402-gateway` or `/mpp-gateway`, REST `next_actions[]` point at the same gateway (`path`, `url`); the keyed route stays in `direct_path`. An action that gateway does not sell carries `gateway_note`: call it directly with your key.
- **Transfers take `to` or `to_address`.** `POST /agent-api/transfer`, `POST /recipient-api/prepare-transfer`, MCP `transfer_loyalty_tokens` and `prepare_loyalty_token_transfer` accept either name for the recipient (B20 token transfer calldata, ERC-20 compatible).
- **Balances and customer counts.** `GET /balance` and `get_token_balance`: `current_balance` (alias `current`) is the on-chain `balanceOf`, `offchain_ledger_balance` is the ledger that `POST /mint/confirm` writes, and `balance_basis` says which one `current_balance` is (`offchain_ledger` only when the chain read failed; then `onchain_balance` is null). `total_customers` is the same number on `GET /customers`, `export_customers`, `GET /analytics` and `get_program_analytics`: distinct wallets with a voucher, a ledger row or a completed mint (platform fee wallet excluded). Email-only store customers are `store_customers`; each response carries `count_basis`. `GET /analytics?token_address=0x…` returns one program, and HTTP 404 when you do not own it.
- **Fees and reports.** Mint `fee_amount` is rounded to 15 significant digits (3 tokens at 1.25% is `0.0375`, not `0.037500000000000006`). `send_report` takes `report_type` `seo_audit`, `audit`, `growth_idea`, `data_report`, `anomaly`, `task`, `recommendation` or `weekly_report`; anything else is 400 `invalid_input` with `allowed_report_types`, before any charge.
- **Trade and certificate error codes.** `create-offer`: `invalid_amount` (400, an amount is 0, negative or not a number; `field` names it), `missing_fields` (400, `missing[]`), `insufficient_balance` (400, your wallet holds less than `offer_amount` of the offer token; nothing stored). `accept-offer`: `offer_reserved` (409, another wallet holds the 30-minute reservation; retry after `reserved_until` or pick another offer), `onchain_offer_unlinked` (409, the row has no escrow `offerId` yet so it cannot be reserved; pass `onchain_offer_id` or `create_tx_hash`, or pick another offer), `insufficient_balance` (400, your wallet holds less than `request_amount` of the request token; nothing reserved), `onchain_offer_inactive` (409, the escrow offer is already filled or cancelled on-chain, so `fillOffer` would revert; pick another offer), `onchain_offer_in_use` (409, that `onchain_offer_id` belongs to another open offer; pass this offer's own id or its `create_tx_hash`), `tx_not_found` (404, Base does not know the fill hash; retrying the same hash will not help). `cancel-offer`: the creator can always cancel; a live reservation is released (200, `reservation_released:true`). `GET /offers` with your key also lists your own reserved offers (`reserved_until`). `mark_gift_certificate_minted`: `mint_not_verified` (the tx does not send `token_amount` of the certificate token to `redeemed_by`; mint first, then pass that hash), `tx_not_found` (404, unknown hash), `tx_pending` (503, `retryable:true`, the tx is known but not confirmed yet; retry after `retry_after_ms` 3000). On x402 none of these is charged.
- **`PUT /merchant-profile` is a partial update.** Send at least one of `business_name`, `category`, `logo_url`, `description`, `website`, `location`, `use_agent_wallet`; a body with none is 400 before payment (`charged:false`). `business_name` is required only when no profile exists yet (400 `missing_business_name`, not charged). `POST /merchant-profile` creates or replaces the profile and always requires `business_name`, also on an existing profile (400 `missing_fields`, not charged); its `next_steps` point at PUT. `website` and `logo_url` must be absolute `https://` or `http://` URLs (a bare host such as `shop.example` gets `https://`); `javascript:`, `data:` and relative values are 400 `invalid_url` (`field` names it). Send `null` to clear one.
- **Other error codes.** `register-agent`: when the wallet already has an active agent, a paid call without `agent_name`, or with a new name and no free seat, is 409 `agent_exists` (`charged:false`); keep the existing key, or resend with `rotate:true` to replace it (sending that agent's own `agent_name` always rotates its key). `POST /rewards` / `create_reward`: `cost` 0 or negative is 400 `invalid_amount`. `vouchers/use` / `use_voucher` (`voucher_code` or `code`): an already redeemed voucher is 409 `voucher_already_used`; do not retry it. Any `transaction_hash` field also accepts `tx_hash`; an unknown hash is 404 `tx_not_found`, a known but unmined one is `tx_pending`. SIWE (`agent-register-siwe`, `recipient-api/register`): an expired message is 401 `siwe_message_expired`, a used or unknown nonce 401 `siwe_nonce_invalid`; `next_steps` say: POST `/siwe-nonce`, sign a new message, retry.
- **Every error says what to do next.** Every 4xx/5xx JSON body from the gateways, agent-api, recipient-api and both MCP servers carries `hint` and `next_steps` `{action, summary, request:{method, url, body|query}, docs, openapi, discovery}` plus `how_to_pay` (402 and gateway 400) or `get_key` (401/403). Follow `next_steps.request`. Paid MCP tool errors are a JSON-RPC result with `isError:true` (HTTP 200, `charged:false`); JSON-RPC protocol errors carry `next_steps` in `error.data`. Docs without JavaScript: https://loyalspark.online/for-agents.md and https://loyalspark.online/.well-known/skills/00-getting-started.md

## Tools

Loyal Spark advertises its own tool catalog (43 merchant + 20 recipient tools). Read tool descriptions exposed by the MCP — they are the source of truth; do not preload a copied catalog from this skill.

| Topic | Reference |
| --- | --- |
| Two API personas: `lsk_` vs `rwk_` | [references/auth.md](https://loyalspark.online/skills/loyal-spark/references/auth.md) |
| B20 native spec and roles/policies | [references/b20-native-spec.md](https://loyalspark.online/skills/loyal-spark/references/b20-native-spec.md) |
| Payment choices: x402 exact, MPP, subscriptions | [references/payment-scenarios.md](https://loyalspark.online/skills/loyal-spark/references/payment-scenarios.md) · [references/x402-paid.md](https://loyalspark.online/skills/loyal-spark/references/x402-paid.md) |
| Calldata execution and fee obligations | [references/calldata-flow.md](https://loyalspark.online/skills/loyal-spark/references/calldata-flow.md) |
| Pairing with Base MCP for signing | [references/base-mcp-integration.md](https://loyalspark.online/skills/loyal-spark/references/base-mcp-integration.md) |
| Gift certificates | [references/gift-certificates.md](https://loyalspark.online/skills/loyal-spark/references/gift-certificates.md) |
| Install steps | [references/install.md](https://loyalspark.online/skills/loyal-spark/references/install.md) |
| Tone, attribution and write-safety | [references/tone.md](https://loyalspark.online/skills/loyal-spark/references/tone.md) |

## Pairing with Base MCP

Loyal Spark tools return **ready-to-sign calldata**, not signed transactions. With Base MCP: call the tool, pass `{ to, data, value }` to **`send_calls`** (EIP-5792 batch when multiple calls), let Base Account approve, wait for a receipt, then run any required sync endpoint. For autonomous agents without Base Account, use a Loyal Spark CDP MPC wallet (`agent-wallet` endpoint).

## Installation

Download the bundle from https://loyalspark.online/skills/loyal-spark/ (`SKILL.md`, `references/`, `plugins/`) into your agent's skills folder, e.g. `~/.claude/skills/loyal-spark/`.

For per-surface install (Claude.ai, ChatGPT, Cursor, Claude Code, Codex, Hermes) see [references/install.md](https://loyalspark.online/skills/loyal-spark/references/install.md).

## x402 / MPP paid calls without input

- 51 of the 89 paid resources work with only a payment: a bare call (no query/body/arguments) returns 2xx with demo data for the LSCSC program (`0xb2000000000000000000008ce542828ec5f09bb7`), `default_applied: true`, a `note`, `required_inputs`, a ready `example_request` (pass your own `token_address`) and `next_steps`. It is charged like any successful call. In discovery these defaulted params are optional (`x-bare-call-default`). Invalid input you do supply is rejected before settlement and never charged.
- 38 resources need `x-api-key` as well as payment (writes and merchant/recipient data). Discovery, the Bazaar extension (`info.input.headers`) and OpenAPI (`x-api-key-required`) mark them, and the unpaid 402 says so before you pay. Get a merchant key (`lsk_`) via `POST /x402-gateway/register-agent` ($0.01) or SIWE; a recipient key (`rwk_`) via `siwe-nonce` + `recipient-api/register`.
- Every resource has a working example input in `/.well-known/x402` and OpenAPI (`x-example-input`).

## Discovery endpoints

| Resource | URL |
| --- | --- |
| Agent manifest | https://loyalspark.online/.well-known/agent.json |
| OpenAPI 3.1 | https://loyalspark.online/openapi.json |
| MCP (merchant, 43 tools, `lsk_`) | https://api.loyalspark.online/loyalty-mcp |
| MCP (recipient, 20 tools, `rwk_`) | https://api.loyalspark.online/recipient-loyalty-mcp |
| x402 paid resources | https://api.loyalspark.online/.well-known/x402 |
| x402 static mirror | https://loyalspark.online/.well-known/x402.json |
| MPP manifest | https://loyalspark.online/.well-known/mpp.json |
| Long LLM reference | https://loyalspark.online/llms-full.txt |
| B20 native spec skill | https://loyalspark.online/.well-known/skills/14-b20-native-spec.md |
| Payment scenarios skill | https://loyalspark.online/.well-known/skills/15-payment-scenarios.md |

## License

MIT. See repository LICENSE.
