# @rhea-finance/cross-chain-aggregation-dex

A TypeScript SDK for the RHEA unified multi-chain Swap API. The SDK handles quote normalization, swap building, wallet execution, EVM approvals, order submission, status polling, reporting, and history. Applications do not need to branch on routers, cross-chain status, or transaction types.

This package is a client SDK for the RHEA Cross-Chain Swap API. For endpoint contracts, request/response fields, and protocol details, see the [Cross-Chain Swap API documentation](https://github.com/rhea-finance/rhea-sdk-docs/blob/main/CrossChainDexAPI.md).

Supported chain families: EVM, Solana, Aptos, NEAR, Tron, Bitcoin, Zcash, and Sui. How to load the product token list and chain coverage is described in [Supported chains and tokens](#supported-chains-and-tokens).

## Supported chains and tokens

The product support list matches `multi-chain-lending` Trade: **18 mainnets**. Use the HTTP API chain ID for `fromChain` / `toChain` in Swap calls, and the token-query alias when fetching token metadata.

| Chain | Type | HTTP / SDK chain ID | Token query alias |
| --- | --- | --- | --- |
| Ethereum | EVM | `1` | `eth` |
| BNB Smart Chain | EVM | `56` | `bsc` |
| Arbitrum One | EVM | `42161` | `arb` |
| Base | EVM | `8453` | `base` |
| Optimism | EVM | `10` | `op` |
| Berachain | EVM | `1385` | `bera` |
| Monad | EVM | `143` | `monad` |
| X Layer | EVM | `196` | `xlayer` |
| Polygon PoS | EVM | `137` | `pol` |
| Gnosis Chain | EVM | `100` | `gnosis` |
| Plasma | EVM | `9745` | `plasma` |
| Solana | Non-EVM | `solana` | `sol` |
| Bitcoin | Non-EVM | `btc` | `btc` |
| NEAR | Non-EVM | `near` | `near` |
| Zcash | Non-EVM | `zcash` | `zcash` (also accepts `zec`) |
| Aptos | Non-EVM | `aptos` | `aptos` |
| Tron | Non-EVM | `tron` | `tron` |
| Sui | Non-EVM | `sui` | `sui` |

"Supported" means the product can load tokens for that chain and send them into the unified quote flow. It does **not** guarantee a route for every pair. Live liquidity, routers, amount, and service status still decide whether a quote succeeds.

### Token coverage (Unified Swap)

Across the 18 product chains above, Unified Swap currently covers **4,000+** tokens (same-chain DEX metadata plus cross-chain Intents tokens, deduplicated per direction). Snapshot: 2026-07-24. Counts change as liquidity providers and Intents listings update; always fetch the live token APIs below for the current list.

### How to fetch supported tokens

Token discovery uses HTTP endpoints **outside** `/api/swap/*`. The SDK does not wrap these calls; applications should `fetch` them directly and map results into `AssetRef` when calling `client.quote()`.

#### 1. Product token list (recommended for Trade UI)

```http
GET https://api.rhea.finance/get_multichain_lending_tokens_data?chains=<COMMA_SEPARATED_ALIASES>
```

Query all currently supported product chains:

```bash
curl "https://api.rhea.finance/get_multichain_lending_tokens_data?chains=bsc,eth,arb,base,op,bera,monad,xlayer,pol,gnosis,plasma,sol,btc,near,zcash,zec,aptos,tron,sui"
```

A successful response is a **JSON array** of token objects (not the `/api/swap` `{ code, data, msg }` envelope). Typical fields:

| Field | Description |
| --- | --- |
| `assetId` | Multichain / Intents asset ID |
| `blockchain` | Token query alias (`eth`, `near`, …) |
| `symbol` | Display symbol |
| `decimals` | Token decimals |
| `contractAddress` | On-chain address; may be empty for natives |
| `price` / `priceUpdatedAt` | Display price only; not a swap quote |
| `icon` | Optional icon URL |

Integration tips:

- Group by `blockchain`; do not merge tokens by `symbol` alone.
- For `quote()`, pass the chain-specific token address/ID as `AssetRef.address`, and use the HTTP/SDK chain ID in `fromChain` / `toChain` / `AssetRef.chain` (for example Base `"8453"`, Solana `"solana"`).
- Presence in this list means the token is discoverable. Confirm a route with `client.quote()` (or `POST /api/swap/quote`) before trading.
- Prefer short-lived cache; refresh periodically instead of hard-coding the list.

#### 2. Per-chain token price metadata

```http
GET https://api.rhea.finance/get_chain_prices?chain=<TOKEN_QUERY_ALIAS>
```

Example:

```bash
curl "https://api.rhea.finance/get_chain_prices?chain=eth"
curl "https://api.rhea.finance/get_chain_prices?chain=bsc"
curl "https://api.rhea.finance/get_chain_prices?chain=base"
```

`chain` is required and must be a **token query alias** (`eth`, `bsc`, `base`, …). A successful response uses `{ code, data, msg }`:

```ts
// code === 0 && msg === "success"
type ChainPricesResponse = {
  code: number;
  msg: string;
  data: Record<
    string,
    {
      address: string;
      chainId: number;
      decimals: number;
      symbol: string;
      name?: string;
      price: string;
      logoURI?: string;
      updated_at?: number;
    }
  >;
};
```

Use this endpoint for same-chain token metadata and display prices. It is keyed by token address. It does not replace `get_multichain_lending_tokens_data` for the full multi-chain Trade selector, and some non-EVM aliases may return an empty `data` object.

## 1. What the SDK does

The recommended flow has only two steps:

```ts
const quote = await client.quote(quoteRequest);
const result = await client.swap({ quote });
```

The normalized object returned by `quote()` can be passed directly to `swap()`. Internally, `swap()` continues with `buildSwap()` and `executeSwap()`, then selects the registered executor that matches the execution type returned by the API. Applications do not need to determine:

- whether the swap is same-chain or cross-chain;
- which router is used;
- whether execution requires a transaction, a signed order, or a deposit transfer;
- whether an EVM approval is required;
- whether the flow is an MCA deposit, NEAR withdrawal, or relayer withdrawal.

The application only provides the request fields and the relevant wallet adapters. Do not modify `quote.buildContext` or rebuild a swap request from `quote.raw`.

## 2. Installation

```bash
pnpm add @rhea-finance/cross-chain-aggregation-dex
```

Node.js 16 or later is required. When running on Node.js 16 without a global `fetch`, inject a compatible implementation through `SwapClientConfig.fetch`.

## 3. Recommended: simple quote → swap flow

The example below swaps USDC on Base for USDC on Solana. All token amounts use base-unit decimal strings. For example, USDC has 6 decimals, so `"1000000"` represents 1 USDC.

### 3.1 Minimal EVM adapter

The SDK does not receive private keys and is not coupled to a specific wallet library. Wrap your wallet implementation in an adapter:

```ts
import {
  createEvmExecutor,
  type EvmWalletAdapter,
} from "@rhea-finance/cross-chain-aggregation-dex/executors/evm";

const evmAdapter: EvmWalletAdapter = {
  async sendTransaction(tx) {
    const response = await wallet.sendTransaction({
      to: tx.to,
      data: tx.data,
      value: tx.value,
      gasLimit: tx.gasLimit,
    });
    return { txHash: response.hash, raw: response };
  },

  async signTypedData(request) {
    return wallet.signTypedData(
      request.typedData.domain,
      request.typedData.types,
      request.typedData.message
    );
  },

  async waitForTransaction(txHash) {
    return provider.waitForTransaction(txHash);
  },
};

const evmExecutor = createEvmExecutor(evmAdapter);
```

The EVM executor does not read the currently connected chain. It uses `tx.chainId` from the API build response. The wallet should prompt the user or switch networks when sending the transaction.

### 3.2 Create a SwapClient

```ts
import { SwapClient } from "@rhea-finance/cross-chain-aggregation-dex";

const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  getAccessToken: () => sessionStorage.getItem("access-token") ?? "",
  executors: [evmExecutor],
});
```

### 3.3 Request a quote

```ts
import type {
  AssetRef,
  QuoteRequest,
} from "@rhea-finance/cross-chain-aggregation-dex";

const baseUsdc: AssetRef = {
  chain: "8453",
  address: "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
  symbol: "USDC",
  decimals: 6,
};

const solanaUsdc: AssetRef = {
  chain: "solana",
  address: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  symbol: "USDC",
  decimals: 6,
};

const quoteRequest: QuoteRequest = {
  fromChain: "8453",
  toChain: "solana",
  tokenIn: baseUsdc,
  tokenOut: solanaUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  quoteWaitingTimeMs: 3000,
  sender: "0xYourBaseAddress",
  recipient: "YourSolanaAddress",
};

const quote = await client.quote(quoteRequest);
```

`quote()` calls `POST /api/swap/quote`. `quoteWaitingTimeMs` is a quote API request parameter used by routes such as Near Intents to wait for a quote. It is not an on-chain RPC timeout. The SDK sends `3000` when the field is omitted.

### 3.4 Execute the swap directly

```ts
const result = await client.swap({
  quote,
  waitFor: "submitted",
  beforeSign(preview) {
    console.log("Wallet action requested", preview);
  },
});

console.log(result.status, result.txHash, result.orderId);
```

`swap()` does not request another quote. It uses the `buildContext` stored in the quote to call the swap API, normalizes the build response, and invokes the matching executor.

The default `waitFor` mode is `"submitted"`. The method returns after the wallet successfully signs or submits the transaction. This does not mean the assets have arrived on the destination chain.

To let the SDK continue polling for final delivery:

```ts
const result = await client.swap({
  quote,
  waitFor: "completed",
});

if (result.status === "completed") {
  console.log("Order completed");
}
```

`"completed"` polls the API only when the build response contains a queryable `orderId`. The default polling interval is 5 seconds and the default timeout is 250 seconds. A timeout throws `ORDER_TIMEOUT`, but it does not revert an already submitted on-chain transaction.

### 3.5 Check final delivery status

If the swap first returns with `"submitted"`, use the returned `orderId` to poll manually:

```ts
if (result.orderId) {
  const finalStatus = await client.waitForOrder({
    orderId: result.orderId,
    router: result.router,
    intervalMs: 5000,
    timeoutMs: 250000,
  });

  console.log(finalStatus.status);
}
```

For a single status request:

```ts
const status = await client.getOrderStatus({
  orderId: result.orderId!,
  router: result.router,
});
```

Terminal statuses are `completed`, `failed`, `refunded`, and `expired`.

## 4. Simple-flow field reference

### 4.1 SwapClientConfig

| Field | Type | Required | Description and default |
| --- | --- | --- | --- |
| `baseUrl` | `string` | Yes | API base URL, for example `https://api.rhea.finance`. A trailing `/` is removed. |
| `apiKey` | `string` | No | API credential sent with requests. |
| `getAccessToken` | `() => string \| Promise<string>` | No | Reads an access token before each request. Use this for refreshable sessions. |
| `fetch` | `typeof globalThis.fetch` | No | Custom fetch implementation. The SDK binds its invocation context to avoid browser `Illegal invocation` errors. It is normally required on Node.js 16. |
| `headers` | `Record<string,string>` or function | No | Additional request headers. The function form may return a promise. |
| `timeoutMs` | `number` | No | Timeout for each HTTP request in milliseconds. Default: `15000`. |
| `retry` | `Partial<RetryConfig>` | No | Retry policy for retryable quote/read operations. Defaults: 2 retries, 250ms base delay, 2000ms maximum delay, and jitter enabled. |
| `logger` | `SdkLogger` | No | Receives structured `api.request`, `api.response`, and `api.retry` entries. |
| `executors` | `readonly ChainExecutor[]` | Required for execution | Wallet executors. May be omitted when only calling `quote()` or `buildSwap()`. |
| `maxQuoteAgeMs` | `number \| null` | No | Maximum local quote age in milliseconds. Default: `30000`. Set to `null` to disable the local age check; an API-provided `expiresAt` still applies. |
| `reportMode` | `"auto" \| "manual" \| "disabled"` | No | Reporting policy. Default: `"auto"`. A reporting failure does not turn a submitted swap into a failed swap. |
| `onEvent` | `(event) => void` | No | Receives all lifecycle events. |
| `now` | `() => number` | No | Custom millisecond clock, mainly for testing. Default: `Date.now`. |

### 4.2 AssetRef

| Field | Type | Required | Format and meaning |
| --- | --- | --- | --- |
| `chain` | `ChainRef` | Yes | The SDK uses one chain ID format everywhere. EVM chains use decimal strings, such as Base `"8453"`. Other values are `"solana"`, `"aptos"`, `"near"`, `"tron"`, `"btc"`, `"zcash"`, and `"sui"`. |
| `address` | `string` | Yes | Token contract address, mint, coin type, or the API-defined native-token identifier. |
| `symbol` | `string` | No | Display symbol. It is not used for calculations. |
| `decimals` | `number` | No | Token precision. Supplying it is recommended so applications can format and convert amounts correctly. |
| `isNative` | `boolean` | No | Whether the asset is the chain's native token. |

`tokenIn.chain` should match `fromChain`, and `tokenOut.chain` should match `toChain`.

### 4.3 QuoteRequest

| Field | Type | Required | Format and meaning |
| --- | --- | --- | --- |
| `fromChain` | `ChainRef` | Yes | Source chain ID, for example `"8453"`. |
| `toChain` | `ChainRef` | Yes | Destination chain ID, for example `"solana"`. |
| `tokenIn` | `AssetRef` | Yes | Asset being spent. |
| `tokenOut` | `AssetRef` | Yes | Asset being received. |
| `amountIn` | `string` | Yes | A non-negative base-unit decimal integer string. Do not pass `"1.5"` or scientific notation. |
| `slippageBps` | `number` | Yes | Slippage in basis points. `50` means 0.5%; `100` means 1%. |
| `quoteWaitingTimeMs` | `number` | No | Time the quote API may wait for route quotes, in milliseconds. Must be a non-negative integer. Default: `3000`. |
| `sender` | `string` | Yes | Sender address on the source chain. |
| `recipient` | `string` | No | Recipient address on the destination chain. Cross-chain requests should normally provide it explicitly. |
| `extensions` | `Record<string,unknown>` | No | Additional fields forwarded to the API. Regular applications should not use this to replace standard fields. |

Important normalized quote fields:

| Field | Meaning |
| --- | --- |
| `estimatedOut` | Estimated output amount as a base-unit string. |
| `minAmountOut` | Minimum output after slippage as a base-unit string. |
| `route.router` | Router selected by the SDK. Use it for display or diagnostics, not application-side execution branching. |
| `alternatives` | Normalized summaries of other available routes. |
| `receivedAt` / `expiresAt` | Millisecond timestamps used for quote freshness validation. |
| `buildContext` | Read-only context used by `swap()` and `buildSwap()`. Do not modify it. |
| `raw` | Original quote API `data`, useful for diagnostics or fields that are not normalized. |

### 4.4 SwapInput and WaitMode

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `quote` | `Quote` | Yes | The value returned by `client.quote()`. Pass it directly without modification. |
| `waitFor` | `"submitted" \| "source-confirmed" \| "completed"` | No | Default: `"submitted"`. |
| `signal` | `AbortSignal` | No | Cancels unfinished SDK requests or waits. It cannot withdraw an already broadcast transaction. |
| `onEvent` | `(event) => void` | No | Receives lifecycle events for this swap only. |
| `beforeSign` | `(preview) => void \| Promise<void>` | No | Called before each wallet signature or transaction request. It can be used for application confirmation UI. |
| `idempotencyKey` | `string` | No | Sent with the build request. Duplicate-execution protection is determined by the server. |

Wait modes:

| Mode | Return condition |
| --- | --- |
| `submitted` | Returns after the wallet signs or broadcasts the source action. |
| `source-confirmed` | Waits for source-chain confirmation when the executor provides a confirmation method; otherwise falls back to `submitted`. |
| `completed` | After source execution, polls the server until a delivery terminal state when an order reference exists. Without an order reference, it returns the source execution result. |

### 4.5 SwapExecutionResult

| Field | Type | Description |
| --- | --- | --- |
| `executionId` | `string` | Unique identifier for this SDK execution. |
| `status` | `string` | `submitted`, `source-confirmed`, `processing`, `completed`, `failed`, `refunded`, or `expired`. |
| `router` | `string` | Router used for execution. Pass it unchanged when querying order status. |
| `txHash` | `string?` | Hash of a single source-chain transaction. |
| `txHashes` | `string[]?` | Hashes of multiple source-chain transactions, such as a NEAR transaction batch. |
| `orderId` | `string?` | Server order identifier. When present, it can be passed to `waitForOrder()`. |
| `depositAddress` | `string?` | Cross-chain deposit address. |
| `report` | `object?` | Reporting state: `reported`, `failed`, or `skipped`. A report warning does not invalidate the source submission. |
| `raw` | `unknown` | Executor confirmation response or original swap API build data. |

`submitted` and `source-confirmed` do not mean that the destination assets have arrived. Use `waitFor: "completed"` or `waitForOrder()` to check final delivery.

## 5. Advanced: buildSwap → executeSwap

Use the advanced flow only when you need to inspect the build, pass it between processes, or separate building from wallet execution:

```ts
const quote = await client.quote(quoteRequest);

// Builds and normalizes the execution without opening a wallet.
const build = await client.buildSwap({
  quote,
  idempotencyKey: crypto.randomUUID(),
});

console.log(build.router, build.execution.kind, build.raw);

// Wallet execution can happen later.
const result = await client.executeSwap({
  build,
  waitFor: "submitted",
});
```

The following calls are equivalent:

```ts
await client.swap({ quote });

// Equivalent to:
const build = await client.buildSwap({ quote });
await client.executeSwap({ build });
```

`buildSwap()` does not require an executor. `executeSwap()` requires a registered executor that supports `build.execution.kind`. MCA relayer withdrawals include an SDK-managed message-signing and submission flow, so use `swap()` instead of splitting that flow.

## 6. Executor adapters

Import each chain executor from its dedicated subpath:

```ts
import { createEvmExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/evm";
import { createSolanaExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/solana";
import { createAptosExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/aptos";
import { createNearExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/near";
import { createTronExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/tron";
import { createBitcoinExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/bitcoin";
import { createZcashExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/zcash";
import { createSuiExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/sui";
```

| Chain | Supported execution kinds | Core adapter capabilities |
| --- | --- | --- |
| EVM | `evm-transaction`, `evm-signature` | `sendTransaction`, `signTypedData`; optional `waitForTransaction`. |
| Solana | `solana-transaction` | Submit a serialized transaction; optional confirmation wait. |
| Aptos | `aptos-entry-function` | Submit an entry function; optional confirmation wait. |
| NEAR | `near-transaction-batch` | Submit NEAR transactions sequentially or as a batch; optional confirmation wait. |
| Tron | `tron-transfer` | Submit a native-token or token transfer; optional confirmation wait. |
| Bitcoin | `bitcoin-transfer` | Submit a UTXO transfer. Configure `defaultFeeRate` when the build does not provide a fee rate. |
| Zcash | `zcash-transfer` | Submit a transparent-address transfer and return a real `txHash`; optional confirmation wait. |
| Sui | `sui-transfer` | Submit a coin transfer; optional confirmation wait. |

Zcash follows the same standard as every other chain: a successful wallet submission must return a real transaction hash. The SDK never invents a transaction hash.

For MCA quotes, the SDK may also read these methods from the registered executor:

- `getIdentityKey()` generates `mca.signer.identityKey` and is required for MCA quotes;
- `signMessage()` signs the exact API-provided message when required by an MCA relayer withdrawal.

These are adapter capabilities. Application code calling `client.swap()` does not pass a separate signer.

## 7. EVM approvals

When the swap API build response contains an `approval`, the EVM executor performs these steps in order:

1. Call the optional `isApprovalRequired(approval)` method. If it is not implemented, approval is assumed to be required.
2. Submit the approval transaction.
3. If the adapter implements `waitForTransaction`, wait for the approval transaction to confirm.
4. Submit the main swap transaction or request the EIP-712 signature.

Applications do not need to query allowances or inspect `needsApprove`. To avoid unnecessary approval transactions, implement `isApprovalRequired` in the adapter:

```ts
const evmAdapter: EvmWalletAdapter = {
  // ...sendTransaction, signTypedData, and other methods
  async isApprovalRequired(approval) {
    const allowance = await readAllowance(approval.spender);
    return allowance < requiredAmount;
  },
};
```

The lifecycle emits `approval-requested`, `approval-submitted`, and the subsequent signing or submission events in order.

## 8. MCA deposits and withdrawals

MCA flows use the same `client.quote()` and `client.swap()` methods. The `flow` field distinguishes deposits from withdrawals. Applications do not branch on the returned router.

Common MCA fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `flow` | `"deposit" \| "withdraw"` | Yes | MCA operation. |
| `mcaAccountId` | `string` | Yes | NEAR account ID of the MCA. |
| `signerChain` | `McaSignerChain` | Yes | Selects the registered executor that supplies identity and signing capabilities. |
| `recipientMsgSignatures` | `string[]` | No | Existing recipient-message signatures forwarded in the MCA payload. |
| `depositSignerProofSignatures` | `string[]` | No | Existing deposit-signer-proof signatures forwarded in the MCA payload. |

### Deposit

```ts
const quote = await client.quote({
  flow: "deposit",
  mcaAccountId: "account.near",
  signerChain: "evm",
  fromChain: "42161",
  toChain: "near",
  tokenIn: arbitrumUsdc,
  tokenOut: mcaUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "0xYourAddress",
  recipient: "account.near",
  collateral: {
    useAsCollateral: true,
  },
});

const result = await client.swap({ quote, waitFor: "completed" });
```

`collateral.useAsCollateral` is a required boolean that specifies whether the deposited asset should be used as Burrow collateral.

### Withdraw

```ts
const quote = await client.quote({
  flow: "withdraw",
  mcaAccountId: "account.near",
  signerChain: "evm",
  fromChain: "near",
  toChain: "8453",
  tokenIn: mcaUsdc,
  tokenOut: baseUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "account.near",
  recipient: "0xYourBaseAddress",
  collateral: {
    needDecrease: true,
    decreaseAmountBurrow: "1.0",
    withdrawAll: false,
  },
  executionPreference: "relayer",
});

const result = await client.swap({
  quote,
  waitFor: "completed",
  beforeSign(preview) {
    console.log("MCA message signature preview", preview);
  },
});
```

Withdraw-only fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `collateral.needDecrease` | `boolean` | Yes | Whether Burrow collateral must be decreased. |
| `collateral.decreaseAmountBurrow` | `string` | Yes | Human-readable Burrow decimal amount, such as `"1.0"`. This is not a token base-unit amount. |
| `collateral.withdrawAll` | `boolean` | No | Whether to withdraw the full available amount. |
| `executionPreference` | `"auto" \| "near" \| "relayer"` | No | Default: `"auto"`. Set explicitly to force direct NEAR or relayer execution. |
| `boundNearAccountId` | `string` | Required for automatic NEAR selection | In `auto` mode, direct NEAR execution is selected only when `toChain === "near"` and `recipient` exactly matches this field. Otherwise, the relayer is selected. |

For direct NEAR execution, the NEAR executor submits `nearMcaWithdrawTx`. For relayer execution, the executor selected by `signerChain` uses `signMessage()` to sign the API-provided `messageToSign`, after which the SDK submits the order. Application code only calls `swap()`.

## 9. Lifecycle, errors, and cancellation

Lifecycle events can be observed globally on the client or for an individual swap:

```ts
const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  executors: [evmExecutor],
  onEvent(event) {
    console.log("Swap lifecycle", event);
  },
});
```

Events may cover build, approval, signing, submission, source confirmation, order status, completion, warnings, and failures.

SDK errors use the `SwapSdkError` type:

```ts
import { SwapSdkError } from "@rhea-finance/cross-chain-aggregation-dex";

try {
  await client.swap({ quote });
} catch (error) {
  if (error instanceof SwapSdkError) {
    console.error({
      code: error.code,
      stage: error.stage,
      message: error.message,
      retryable: error.retryable,
      cause: error.cause,
      details: error.details,
    });
  }
}
```

Common `stage` values are `quote`, `build`, `approve`, `sign`, `broadcast`, `submit`, `report`, `status`, and `history`. Common `code` values include `QUOTE_EXPIRED`, `USER_REJECTED`, `APPROVAL_FAILED`, `SIGNING_FAILED`, `BROADCAST_FAILED`, and `ORDER_TIMEOUT`.

Pass an `AbortSignal` to stop an unfinished request or wait:

```ts
const controller = new AbortController();
const promise = client.swap({ quote, signal: controller.signal });
controller.abort();
await promise;
```

Cancellation only stops the SDK's current work. It cannot revert an approval, signature, or transaction that has already been submitted.

## 10. Retries, logging, and credentials

```ts
const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  getAccessToken: async () => authStore.getToken(),
  retry: {
    maxRetries: 2,
    baseDelayMs: 250,
    maxDelayMs: 2000,
    jitter: true,
  },
  logger: {
    log(entry) {
      console.log("SDK API", entry);
    },
  },
});
```

The SDK automatically retries only retryable quote/read operations. It does not automatically retry build, broadcast, or order-submission operations that could execute twice. Network failures preserve the underlying error name and message to help diagnose CORS failures, connection resets, timeouts, or fetch invocation problems.

## 11. Raw API, reporting, and history

Most applications should use normalized methods. Use the Raw API only when the complete original server fields are required:

```ts
await client.quoteRaw(rawQuoteRequest);
await client.buildRaw(rawBuildRequest);
await client.submitOrderRaw(rawSubmitRequest);
await client.getOrderStatusRaw(rawStatusRequest);
await client.reportRaw(rawReportRequest);
await client.getHistoryRaw(rawHistoryRequest);
```

The default reporting mode is `reportMode: "auto"`. If a swap succeeds but reporting fails, `result.report.status` is `"failed"` and a warning is emitted. The SDK does not throw a misleading swap failure. For manual reporting:

```ts
const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  executors: [evmExecutor],
  reportMode: "manual",
});

const result = await client.swap({ quote });
await client.report(result);

// Retry after a reporting failure:
await client.retryReport(result);
```

Query history with:

```ts
const history = await client.getHistory({
  sender: "0xYourAddress",
  page: 1,
  pageSize: 20,
  status: ["processing", "completed"],
});
```

The SDK applies `status` filtering locally. The returned page has `filteredLocally: true`.

## 12. Amount utilities

Avoid JavaScript floating-point arithmetic for token amounts:

```ts
import {
  formatUnits,
  parseUnits,
} from "@rhea-finance/cross-chain-aggregation-dex";

parseUnits("1.25", 6);      // "1250000"
formatUnits("1250000", 6); // "1.25"
```

`parseUnits()` rejects fractional precision beyond the token's decimals. `formatUnits()` accepts only non-negative base-unit decimal integer strings.
