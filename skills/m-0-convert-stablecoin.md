---
name: m-0-convert-stablecoin
description: Quote, execute, and track a cross-chain stablecoin conversion on the M0 Orchestration API.
generated: '2026-07-20'
method: generated
source: openapi/m-0-orchestration-openapi.json
api: M0 Orchestration API
base_url: https://gateway.m0.xyz/v1/orchestration
auth: apiKey header x-api-key
operations:
- quote_getSupportedAssets
- quote_quote
- order_getOrders
- order_getOrder
- order_cancelOrder
---

# Convert a stablecoin with the M0 Orchestration API

Move or convert between M0 extensions and other stablecoins (USDC, USDT) across
EVM chains and Solana. Every request is authenticated with the `x-api-key` header.
The API returns unsigned transaction payloads that YOU sign and broadcast; it never
holds funds.

## Steps

1. **Discover assets** — `GET /supported-assets` (`quote_getSupportedAssets`).
   Pick the source and destination `Asset` by `chain` + `address`. `m0Extension:
   true` marks an M0-powered token.

2. **Get a quote** — `POST /quote` (`quote_quote`) with a body of
   `{ route: { source, destination }, amountIn, sender }`. `route.source` and
   `route.destination` are `{ chain, address }` tuples. Raise `maxNumQuotes` to see
   more than the single best quote. The response is a ranked array of `Quote`, each
   with `amountOut`, `estFillTime`, and one or more `payloads`.

3. **Execute** — for each `Payload` in the chosen quote, read `data` (a
   `TransactionPayload`, discriminated `evm` | `svm`). Always broadcast to
   `payload.chain` (and `payload.chainId` for EVM) — do NOT assume it matches the
   source chain, because cross-chain legs target a different chain.

4. **Track the order** — poll `GET /orders/{originChain}/{orderId}`
   (`order_getOrder`) for full state, or `GET /orders` (`order_getOrders`) to list
   by `sender`/`status` with `limit`+`offset`. `OrderStatus` is
   `CREATED` -> `COMPLETED` | `CANCELLED`.

5. **Cancel if needed** — `POST /orders/{originChain}/{orderId}/cancel`
   (`order_cancelOrder`). For cross-chain orders the cancel is initiated on the
   order's DESTINATION chain; read `payload.chain` on the returned transaction to
   know where to broadcast.

## Conventions & error handling

- **Errors**: every failure returns `{ code, message, requestId }`. Branch on
  `code`, not the HTTP status (see `errors/m-0-problem-types.yml`). Retry `QuoteError`
  and `CancelOrderError` (500) with backoff; do not retry `BadQuoteRequest` (400),
  `NoQuotesAvailable` (404), or `OrderNotCancellable` (409).
- **Idempotency**: there is no HTTP Idempotency-Key; replay safety is enforced
  on-chain via each order's nonce.
- **Gasless option**: for EVM routes use `POST /permit/quote` +
  `POST /permit/build` to collapse approve + action into one transaction via an
  EIP-2612 permit signature.
- Include `requestId` from any error body when contacting M0 support.
