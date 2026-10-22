# Advanced TypeScript Client with Middleware

A chainable middleware API for Soroban transaction signing, validation, price
feeds and plugins. It is generic over the transaction type and ships a Soroban submitter that implements the simulate → assemble → sign → send → poll
lifecycle against a narrow RPC interface.

## Install

```sh
npm install @soroban-amm/ts-advanced-client
```

## Quickstart

```ts
import {
  AdvancedClient,
  priceFeedMiddleware,
  signerMiddleware,
  validationMiddleware,
  nonZeroAmountRule,
  deadlineRule,
  slippageRule,
} from '@soroban-amm/ts-advanced-client'

type Transaction = { kind: string; amount: bigint; signed?: boolean }
type Result = { hash: string }

const client = new AdvancedClient<Transaction, Result>({ submitter })

client.middleware()
  .use(priceFeedMiddleware(async () => ({ numerator: 12345n, denominator: 1000n })))
  .use(validationMiddleware([nonZeroAmountRule, deadlineRule(() => Date.now()), slippageRule]))
  .use(signerMiddleware({ sign: async (tx) => ({ ...tx, signed: true }) }))

const result = await client.sendTransaction({ kind: 'swap', amount: 1000n })
```

## Aborting the chain

A middleware that returns without calling `next()` aborts the chain. The
transaction is *not* submitted, and `sendTransaction` rejects with a
`MiddlewareAborted` error. This is the natural way to write a validation
middleware ("if invalid, return"). Throwing is the explicit rejection path
and propagates the error to the caller.

```ts
import { MiddlewareAborted } from '@soroban-amm/ts-advanced-client'

try {
  await client.sendTransaction(tx)
} catch (err) {
  if (err instanceof MiddlewareAborted) {
    // the chain aborted; nothing was submitted
  }
}
```

## Price quotes

Prices are `bigint`-based `PriceQuote { numerator, denominator }` values.
No floating point is involved, so i128 amounts remain exact and downstream
slippage checks do not lose precision.

## Middlewares

- `signerMiddleware(signer)` — signs the transaction.
- `priceFeedMiddleware(getPrice)` — stores a `PriceQuote` in `metadata.price`.
- `validationMiddleware(rules)` — runs built-in rules:
  - `nonZeroAmountRule` — rejects zero/negative amounts (`INVALID_AMOUNT`),
  - `deadlineRule(clock)` — rejects past deadlines (`DEADLINE_PASSED`),
  - `slippageRule` — rejects quotes below `minRatio` (`SLIPPAGE_EXCEEDED`).
- `retryMiddleware({ attempts, backoffMs, retryOn })` — retries transient RPC
  failures with exponential backoff and gives up after attempts with the last error.
- `loggingMiddleware(sink)` — logs begin/end/error to a sink.
- `plugin({ name, middleware, hooks })` — registers a middleware plus
  `beforeSend` / `afterSend` lifecycle hooks.

## Soroban submitter

@SorobanSubmitter` implements `TransactionSubmitter` on top of a narrow
@SorobanRpc` interface (the `@soroban-amm/sdk` write path implements it). It
simulates, assembles with the simulated resource fee, signs, sends, resubmits
on `TRY_AGAIN_LATER`, and polls until `SUCCESS`/`FAILED` or a timeout.

```ts
import { SorobanSubmitter } from '@soroban-amm/ts-advanced-client'

const submitter = new SorobanSubmitter({
  rpc,
  signer,
  timeoutMs: 30_000,
  pollIntervalMs: 1_000,
  maxResubmits: 3,
})
```
