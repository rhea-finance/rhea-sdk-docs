# @rhea-finance/cross-chain-aggregation-dex

TypeScript SDK for the unified multi-chain Swap API. It provides raw and normalized quote, build, execution, order status, report, and history interfaces without coupling the core package to a wallet or UI framework.

Supported chain families: EVM, Solana, Aptos, NEAR, Tron, Bitcoin, Zcash, and Sui.

## Install

```bash
pnpm add @rhea-finance/cross-chain-aggregation-dex
```

## Quick start

All token amounts are base-unit decimal strings. Slippage uses basis points (`50` means 0.5%).

```ts
import { SwapClient, type QuoteRequest } from "@rhea-finance/cross-chain-aggregation-dex";

const getAccessToken = async () => sessionStorage.getItem("access-token") ?? "";

const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  getAccessToken,
});

const request: QuoteRequest = {
  fromChain: "btc",
  toChain: "near",
  tokenIn: {
    chain: "btc",
    address: "btc",
    symbol: "BTC",
    decimals: 8,
    isNative: true,
  },
  tokenOut: {
    chain: "near",
    address: "wrap.near",
    symbol: "wNEAR",
    decimals: 24,
  },
  amountIn: "100000",
  slippageBps: 50,
  sender: "bc1...",
  recipient: "alice.near",
};

const quote = await client.quote(request);
const build = await client.buildSwap({ quote });

// build is safe to inspect or send to another process.
// Register a chain executor before calling executeSwap.
```

`buildSwap()` never opens a wallet. To execute a build, inject one or more `ChainExecutor` implementations when creating the client:

```ts
const clientWithExecutor = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  getAccessToken,
  executors: [bitcoinExecutor],
});

const result = await clientWithExecutor.executeSwap({
  build,
  waitFor: "submitted",
});
```



### Executor adapters

Each executor is imported from its own subpath and accepts a wallet-neutral adapter. The application decides whether that adapter wraps a browser wallet, server signer, RPC service, or HSM.

```ts
import { SwapClient } from "@rhea-finance/cross-chain-aggregation-dex";
import {
  createEvmExecutor,
  type EvmWalletAdapter,
} from "@rhea-finance/cross-chain-aggregation-dex/executors/evm";

const evmWallet: EvmWalletAdapter = {
  getIdentityKey: () => wallet.address,
  signMessage: (message) => wallet.signMessage(message),
  sendTransaction: async (tx) => {
    const response = await wallet.sendTransaction({
      to: tx.to,
      data: tx.data,
      value: tx.value,
      gasLimit: tx.gasLimit,
    });
    return { txHash: response.hash, raw: response };
  },
  signTypedData: async (request) =>
    wallet.signTypedData(
      request.typedData.domain,
      request.typedData.types,
      request.typedData.message
    ),
  waitForTransaction: async (txHash) => provider.waitForTransaction(txHash),
};

const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  getAccessToken,
  executors: [createEvmExecutor(evmWallet)],
});
```

Other executor subpaths follow the same pattern:

```ts
import { createSolanaExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/solana";
import { createAptosExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/aptos";
import { createNearExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/near";
import { createTronExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/tron";
import { createBitcoinExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/bitcoin";
import { createZcashExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/zcash";
import { createSuiExecutor } from "@rhea-finance/cross-chain-aggregation-dex/executors/sui";
```

Bitcoin requires `feeRate` in the build or a configured fallback:

```ts
const bitcoinExecutor = createBitcoinExecutor(bitcoinWallet, {
  defaultFeeRate: 4,
});
```

Zcash adapters may return `{ requiresUserAction: true }` for legacy wallets that complete transfer confirmation in an external interface. The SDK returns `requires-user-action` without creating a fake transaction hash.

`swap({ quote })` is the convenience form of `buildSwap({ quote })` followed by `executeSwap({ build })`. It does not request another quote.

## MCA swaps

MCA deposit and withdrawal are execution modes of the unified root API. Use the same `client.quote()`, `client.buildSwap()`, `client.swap()`, `client.report()`, and `client.getHistory()` methods as a regular swap. The SDK does not create MCA accounts or query lending positions.

### Deposit into an MCA

The destination asset address is the Burrow token id expected by the Swap API. Execution reuses the registered source-chain executor.

```ts
const quote = await client.quote({
  flow: "deposit",
  mcaAccountId: "account.near",
  fromChain: "1",
  toChain: "near",
  tokenIn: ethereumUsdc,
  tokenOut: mcaUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "0x...",
  recipient: "account.near",
  signerChain: "evm",
  collateral: {
    useAsCollateral: true,
  },
});

const result = await client.swap({ quote, waitFor: "completed" });
```

The report includes `multi_addr`. Its `tx_type` remains `same-chain` or `cross-chain`, matching the unified Swap API.

### Withdraw directly to NEAR

Register a NEAR executor and request the NEAR path. The SDK parses `nearMcaWithdrawTx`, builds the MCA `exec` function call, and asks the injected NEAR wallet to send it. This path does not create an additional off-chain MCA message signature.

```ts
const quote = await client.quote({
  flow: "withdraw",
  mcaAccountId: "account.near",
  fromChain: "near",
  toChain: "near",
  tokenIn: mcaUsdc,
  tokenOut: nearUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "account.near",
  recipient: "alice.near",
  signerChain: "near",
  collateral: {
    needDecrease: false,
    decreaseAmountBurrow: "0",
  },
  executionPreference: "near",
  boundNearAccountId: "alice.near",
});

await client.swap({ quote, waitFor: "completed" });
```

With `executionPreference: "auto"`, NEAR direct execution is selected only when the destination chain is NEAR and `recipient` exactly matches `boundNearAccountId`.

### Withdraw through the multichain relayer

For other destination chains, expose `getIdentityKey()` and `signMessage()` on the connected wallet adapter used by its registered executor. The SDK signs the exact `messageToSign` returned by the API, then submits `mcaRelayer` through `POST /api/swap/swap`. No source-chain executor broadcasts a transaction for this path.

```ts
const quote = await client.quote({
  flow: "withdraw",
  mcaAccountId: "account.near",
  fromChain: "near",
  toChain: "1",
  tokenIn: mcaUsdc,
  tokenOut: ethereumUsdc,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "account.near",
  recipient: wallet.address,
  signerChain: "evm",
  collateral: {
    needDecrease: true,
    decreaseAmountBurrow: "1000000",
    withdrawAll: true,
  },
  executionPreference: "relayer",
});

await client.swap({
  quote,
  waitFor: "completed",
  beforeSign(preview) {
    showMcaSignatureConfirmation(preview);
  },
});
```

Supported MCA signer identity formats are EVM, Solana, Bitcoin, NEAR, Aptos, Sui, Zcash, and Tron. Wallet implementations remain application-owned:

```ts
import {
  formatMcaWallet,
  selectMcaSigner,
} from "@rhea-finance/cross-chain-aggregation-dex";

formatMcaWallet("evm", "0xAbC"); // { EVM: "AbC" }

const signer = selectMcaSigner(boundMcaWallets, connectedSignerIdentities);
```

The executor registered for the selected chain must expose `signMessage` when the relayer preview requests a message signature. The SDK never receives a private key or recovery phrase.

### Collateral policy

The SDK does not fetch a lending portfolio. Pass collateral decisions explicitly, or calculate the API fields from data already held by the application:

```ts
import { resolveMcaWithdrawPolicy } from "@rhea-finance/cross-chain-aggregation-dex";

const collateral = resolveMcaWithdrawPolicy({
  collateralBalance: "12.5",
  availableBalance: "1000000",
  amountIn: "999999",
  isMax: false,
});
```

`withdrawAll` becomes true for max selection, exact available balance, or a ratio of at least `0.999999`. The calculation uses decimal strings and `BigInt`, not floating-point arithmetic.

MCA history uses the MCA account id as the server-side history search key. The backend matches this value against `sender`, `recipient`, and `multi_addr`:

```ts
const history = await client.getHistory({
  sender: "account.near",
});
```

The SDK preserves the server page instead of filtering `record.multi_addr` again. Address fields may be presentation-normalized by the API, so callers should not require exact equality with the MCA account id.

## API surfaces

Normalized methods:

- `quote()`
- `buildSwap()`
- `executeSwap()` and `swap()`
- `getOrderStatus()` and `waitForOrder()`
- `report()` and `retryReport()`
- `getHistory()`

Raw methods preserve the unified API `data` shape:

- `quoteRaw()`
- `buildRaw()`
- `submitOrderRaw()`
- `getOrderStatusRaw()`
- `reportRaw()`
- `getHistoryRaw()`



## Execution kinds

Build responses are validated and represented as a discriminated union:

- `evm-transaction`
- `evm-signature`
- `solana-transaction`
- `aptos-entry-function`
- `near-transaction-batch`
- `tron-transfer`
- `bitcoin-transfer`
- `zcash-transfer`
- `sui-transfer`

The SDK core defines the executor contract and registry. Wallet-specific implementations are injected by the application, so importing the package does not access browser wallet globals.

## Runtime and credentials

- Browsers and Node.js 18+ use the global Fetch API.
- Node.js 16 requires a compatible `fetch` implementation through `new SwapClient({ fetch })`.
- Supply credentials with `apiKey` or `getAccessToken`; when both are present, `getAccessToken` takes precedence.
- The package contains no fixed API credential and never manages private keys or wallet recovery phrases.
- Use `AbortSignal` on network, build, execute, and polling calls when cancellation is required.



## Retry and logging

Quote, history, and order-status requests retry network errors, timeouts, HTTP 429, and retryable 5xx responses twice by default. Build, report, order submission, and wallet execution are never retried automatically.

```ts
const client = new SwapClient({
  baseUrl: "https://api.rhea.finance",
  retry: {
    maxRetries: 2,
    baseDelayMs: 250,
    maxDelayMs: 2_000,
    jitter: true,
  },
  logger: {
    log(entry) {
      telemetry.emit(entry.event, entry);
    },
  },
});
```

Log entries contain only request stage, endpoint path, attempt, response status, timing, and SDK error code. They do not include credentials, query strings, request bodies, signatures, or serialized transactions.

## Amount conversion

`parseUnits` and `formatUnits` use string arithmetic and never pass token values through floating-point numbers:

```ts
import {
  formatUnits,
  parseUnits,
} from "@rhea-finance/cross-chain-aggregation-dex";

parseUnits("1.25", 6); // "1250000"
formatUnits("1250000", 6); // "1.25"
```

Both functions reject negative values, scientific notation, malformed input, and unsupported precision. Token decimals must be an integer between 0 and 255.

## History filtering

The service handles sender and pagination. `getHistory({ status })` filters the current returned page locally and sets `filteredLocally: true`; server totals remain unchanged.

## License

MIT