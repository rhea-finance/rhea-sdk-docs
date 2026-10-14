# @rhea-finance/cross-chain-aggregation-dex

A TypeScript SDK for the RHEA unified multi-chain Swap API. The SDK handles quote normalization, swap building, wallet execution, EVM approvals, order submission, status polling, reporting, and history. Applications do not need to branch on routers, cross-chain status, or transaction types.

This package is a client SDK for the RHEA Cross-Chain Swap API. For endpoint contracts, request/response fields, and protocol details, see the [Cross-Chain Swap API documentation](https://github.com/rhea-finance/rhea-sdk-docs/blob/main/CrossChainDexAPI.md).

The SDK uses `POST /api/v2/swap/quote` for quotes and `POST /api/v2/swap/swap` for swap building. Order submission, reporting, order status, history, history authorization, and destination token-list requests use `/api/swap/*`. Configure `baseUrl` as the API origin, for example `https://api.rhea.finance`; the SDK appends each endpoint path.

Supported chain families: EVM, Solana, Aptos, NEAR, Tron, Bitcoin, Zcash, and Sui. How to load the product token list and chain coverage is described in [Supported chains and tokens](#supported-chains-and-tokens).

## Supported chains and tokens

The product support list matches `multi-chain-lending` Trade: **27 mainnets**. Use the HTTP / SDK chain ID for `fromChain`, `toChain`, and `AssetRef.chain` in Swap calls. Token-list endpoints use the separate numeric ID shown below; for EVM chains the two IDs are the same.

| Chain | Type | HTTP / SDK chain ID | Token-list chain ID | `blockchain` alias |
| --- | --- | --- | --- | --- |
| Ethereum | EVM | `1` | `1` | `eth` |
| Optimism | EVM | `10` | `10` | `op` |
| Avalanche | EVM | `43114` | `43114` | `avax` |
| Robinhood Chain | EVM | `4663` | `4663` | `robinhood` |
| Katana | EVM | `747474` | `747474` | `katana` |
| Sonic | EVM | `146` | `146` | `sonic` |
| Unichain | EVM | `130` | `130` | `unichain` |
| Pharos | EVM | `1672` | `1672` | `pharos` |
| Tempo | EVM | `4217` | `4217` | `tempo` |
| Arc | EVM | `5042` | `5042` | `arc` |
| BNB Smart Chain | EVM | `56` | `56` | `bsc` |
| Gnosis Chain | EVM | `100` | `100` | `gnosis` |
| Polygon PoS | EVM | `137` | `137` | `pol` |
| Monad | EVM | `143` | `143` | `monad` |
| X Layer | EVM | `196` | `196` | `xlayer` |
| Base | EVM | `8453` | `8453` | `base` |
| Plasma | EVM | `9745` | `9745` | `plasma` |
| HyperEVM | EVM | `999` | `999` | `hypercore` |
| Arbitrum One | EVM | `42161` | `42161` | `arb` |
| Berachain | EVM | `80094` | `80094` | `bera` |
| Tron | Non-EVM | `tron` | `195` | `tron` |
| Solana | Non-EVM | `solana` | `501` | `sol` |
| Sui | Non-EVM | `sui` | `784` | `sui` |
| NEAR | Non-EVM | `near` | `900001` | `near` |
| Bitcoin | Non-EVM | `btc` | `900002` | `btc` |
| Zcash | Non-EVM | `zcash` | `900010` | `zec` |
| Aptos | Non-EVM | `aptos` | `900012` | `aptos` |

For Arc, pass `"5042"` as `fromChain` / `toChain` and use `{ chainId: 5042 }` when loading tokens. For HyperEVM, pass `"999"` as `fromChain` / `toChain` and use `{ chainId: 999 }` when loading tokens; its token metadata uses the `blockchain` alias `hypercore`. The same chain string belongs in the selected asset's `chain` field.

"Supported" means the product can load tokens for that chain and send them into the unified quote flow. It does **not** guarantee a route for every pair. Live liquidity, routers, amount, and service status still decide whether a quote succeeds.

### Token coverage (Unified Swap)

Token coverage is dynamic and direction-specific. The from-token list combines same-chain routing metadata with cross-chain-capable assets, while the cross-chain to-token list is maintained independently. Do not hard-code a token count or assume every from token can also be selected as a cross-chain destination; fetch both live endpoints described below.

### How to load supported tokens

Use the SDK methods for application selectors. Both methods return the same normalized `SwapTokenListItem[]` shape, even though the two HTTP endpoints have different raw response formats:

```ts
const fromTokens = await client.getFromTokens({ chainId: 8453 });
const crossChainToTokens = await client.getCrossChainToTokens({ chainId: 501 });

const tokenIn = fromTokens.find((token) => token.symbol === "USDC")!;
const tokenOut = crossChainToTokens.find((token) => token.symbol === "USDC")!;

const quote = await client.quote({
  fromChain: tokenIn.chain,
  toChain: tokenOut.chain,
  tokenIn,
  tokenOut,
  amountIn: "1000000",
  slippageBps: 50,
  sender: "0x...",
  recipient: "...",
});
```

`chainId` is the numeric **token-list chain ID** from the table above. For same-chain tokenOut, reuse `getFromTokens`; call `getCrossChainToTokens` only for a destination on another chain.

The normalized item extends `AssetRef`:

```ts
interface SwapTokenListItem extends AssetRef {
  chain: ChainRef;
  address: string;
  symbol: string;
  decimals: number;
  isNative: boolean;
  tokenListChainId: number;
  blockchain: string;
  assetId: string;
  contractAddress: string | null;
  coinType: string | null;
  name: string | null;
  logoURI: string | null;
  price: string | number | null;
  priceUpdatedAt: number | null;
  sources: string[];
  raw: Record<string, unknown>;
}
```

`address` and `assetId` contain the identifier accepted by the unified quote API. `contractAddress` separately preserves the on-chain contract/mint when available and is `null` for native assets. Unknown backend fields remain available in `raw`.

The methods cache successful lists for 10 minutes per direction and chain, and merge concurrent identical loads. Failures are never cached. Configure `tokenListCacheTtlMs`, or set it to `0` to disable caching. Requests with an `AbortSignal` are not shared with another caller.

The corresponding HTTP endpoints are documented below for server integrations that do not use the SDK.

#### 1. From-token list

```http
GET https://api.rhea.finance/get_chain_prices?chain=<NUMERIC_CHAIN_ID>
```

Example:

```bash
curl "https://api.rhea.finance/get_chain_prices?chain=1"
curl "https://api.rhea.finance/get_chain_prices?chain=56"
curl "https://api.rhea.finance/get_chain_prices?chain=8453"
```

`chain` is required and uses the token-list registry's numeric ID. EVM chains use their normal chain ID. Common non-EVM IDs are Solana `501`, Tron `195`, Sui `784`, NEAR `900001`, Bitcoin `900002`, Zcash `900010`, and Aptos `900012`.

A successful response uses `{ code, data, msg }`:

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

Use every returned row as the chain's from-token list. The same list is also used for tokenOut when the source and destination chains are the same.

#### 2. Cross-chain to-token list

```http
GET https://api.rhea.finance/api/swap/supported_to_tokens?chain=<NUMERIC_CHAIN_ID>
```

Example:

```bash
curl "https://api.rhea.finance/api/swap/supported_to_tokens?chain=8453"
```

The response uses the standard envelope and returns the final destination list in `data.tokens`:

```ts
type SupportedToTokensResponse = {
  code: number;
  msg: string;
  data: {
    tokens: Array<{
      address?: string;
      assetId?: string;
      symbol: string;
      decimals: number;
      logoURI?: string;
      price?: string | number;
      isNative?: boolean;
      coinType?: string;
      sources?: string[];
    }>;
  };
};
```

Use this endpoint only when tokenOut is on a different chain from tokenIn. The returned rows are already the final supported tokenOut list; do not filter them again with legacy `crossChainTo` or `sameChain` flags.

Integration rules:

- Never build both selectors from one combined token list.
- Do not merge tokens by symbol alone; use chain plus address, asset ID, or coin type.
- Pass the selected normalized token to `client.quote()` and still confirm route availability with the quote API.
- Invalidate a selected cross-chain tokenOut if a refreshed destination list no longer contains its chain and `address` pair.

## DEX and Related Contract Addresses

The following registry lists the chain-specific contracts for integrated DEXes, aggregators, bridges, and RHEA execution infrastructure. Contracts are grouped by integration; when several chains share an address, they appear in the same row.

Integration contract coverage is separate from the unified HTTP API's [supported chain list](#supported-chains-and-tokens). For swap execution, pass the normalized quote to `client.swap({ quote })`. The SDK builds the swap and executes the approval and transaction fields returned for that quote. In particular, an approval spender and a settlement/verifying contract serve different roles.

The Aptos network ID below is `637`; the HTTP API still uses `"aptos"` for `fromChain` / `toChain` and `900012` for token-list requests. NEAR contracts use account IDs instead of hexadecimal addresses.

### OKX (`okx`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), Avalanche (`43114`) | `0x40aA958dd87FC8305b97f2BA922CDdCa374bcD7f` |
| Optimism (`10`), Conflux (`1030`) | `0x68D6B739D2020067D1e2F713b999dA97E4d54812` |
| Cronos (`25`) | `0x70cbb871e8f30fc8ce23609e9e0ea87b6b222f58` |
| BNB Chain (`56`) | `0x2c34A2Fb1d0b4f55de51E1d0bDEfaDDce6b7cDD6` |
| Unichain (`130`) | `0x2e28281Cf3D58f475cebE27bec4B8a23dFC7782c` |
| Polygon (`137`) | `0x3B86917369B83a6892f553609F3c2F439C184e31` |
| Monad (`143`) | `0xf534A8a1CAD0543Cd6438f7534CA3486c01998d4` |
| Sonic (`146`) | `0xd321ab5589d3e8fa5df985ccfef625022e2dd910` |
| Manta (`169`), Metis (`1088`), Mantle (`5000`), Base (`8453`), Linea (`59144`), Scroll (`534352`) | `0x57df6092665eb6058DE53939612413ff4B09114E` |
| X Layer (`196`), Merlin (`4200`) | `0x8b773D83bc66Be128c60e07E17C8901f7a64F000` |
| Fantom (`250`), Arbitrum (`42161`) | `0x70cBb871E8f30Fc8Ce23609E9E0Ea87B6b222F58` |
| zkSync (`324`) | `0xc67879F4065d3B9fe1C09EE990B891Aa8E3a4c2f` |
| HyperEVM (`999`) | `0x56e6983D59bF472Ced0E63966A14d94A3A291589` |
| Pharos (`1672`) | `0x78466A1488f1883d71cFddd1c621351572dE0a1C` |
| Robinhood Chain (`4663`) | `0x42170295F1173c9e5874ea9d00c6d137E1a4f53d` |
| ZetaChain (`7000`) | `0x03B5ACdA01207824cc7Bc21783Ee5aa2B8d1D2fE` |
| Plasma (`9745`) | `0x9FD43F5E4c24543b2eBC807321E58e6D350d6a5A` |
| Blast (`81457`) | `0x5fD2Dc91FF1dE7FF4AEB1CACeF8E9911bAAECa68` |
| Polygon zkEVM (`1101`) | The current API did not return a valid address. |

### Kyber (`kyber`)

| Chains | Contract address |
| --- | --- |
| Optimism (`10`), BNB Chain (`56`), Polygon (`137`), Monad (`143`), Arbitrum (`42161`), Avalanche (`43114`) | `0x6131B5fae19EA4f9D964eAc0408E4408b66337b5` |

### 1inch (`1inch`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), Optimism (`10`), Cronos (`25`), BNB Chain (`56`), Gnosis (`100`), Unichain (`130`), Polygon (`137`), Monad (`143`), Sonic (`146`), Base (`8453`), Arbitrum (`42161`), Avalanche (`43114`), Linea (`59144`) | `0x111111125421ca6dc452d289314280a0f8842a65` |
| zkSync (`324`) | `0x6fd4383cb451173d5f9304f041c7bcbf27d561ff` |
| HyperEVM (`999`) | `0x5281602adc446a94eb48d055f514a6d8d5bee176` |
| Robinhood Chain (`4663`) | `0x5a705de8982235a7fa45bb83dcacf03a211389c7` |

### ParaSwap (`paraswap`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), Optimism (`10`), BNB Chain (`56`), Gnosis (`100`), Unichain (`130`), Polygon (`137`), Robinhood Chain (`4663`), Base (`8453`), Arbitrum (`42161`), Avalanche (`43114`) | `0x6a000f20005980200259b80c5102003040001068` |

### CoW (`cow`)

Both contracts below apply to Ethereum (`1`), BNB Chain (`56`), Gnosis (`100`), Polygon (`137`), Base (`8453`), Plasma (`9745`), Arbitrum (`42161`), Avalanche (`43114`), and Linea (`59144`).

| Contract role | Address |
| --- | --- |
| Approval spender | `0xC92E8bdf79f0507f65a392b0ab4667716BFE0110` |
| Settlement / EIP-712 verifying contract | `0x9008d19f58aabd9ed0d60971565aa8510560ab41` |

### Sushi (`sushi`, currently disabled)

This integration is disabled in the current configuration.

| Chains | Contract address |
| --- | --- |
| Katana (`747474`) | `0xAC4c6e212A361c968F1725b4d055b47E63F80b75` |

### Rango (`rango`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), BNB Chain (`56`), Gnosis (`100`), Unichain (`130`), Polygon (`137`), Monad (`143`), Sonic (`146`), X Layer (`196`), Robinhood Chain (`4663`), Base (`8453`), Arbitrum (`42161`), Avalanche (`43114`), Berachain (`80094`), Katana (`747474`) | `0x69460570c93f9DE5E2edbC3052bf10125f0Ca22d` |

### LI.FI / Jumper (`lifi` / `jumper`)

Both integrations use the following chain-specific addresses.

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), BNB Chain (`56`), Gnosis (`100`), Polygon (`137`), X Layer (`196`), Base (`8453`), Optimism (`10`), Sonic (`146`), Arbitrum (`42161`), Avalanche (`43114`) | `0x1231DEB6f5749EF6cE6943a275A1D3E7486F4EaE` |
| Unichain (`130`) | `0x864b314D4C5a0399368609581d3E8933a63b9232` |
| Monad (`143`) | `0x026F252016A7C47CDEf1F05a3Fc9E20C92a49C37` |
| Pharos (`1672`) | `0xFf70F4A1d11995621854F3692acF286d8aCd04b2` |
| Tempo (`4217`) | `0x2cAcAE8e22418E65dcf7651c67aEbe6288EB8243` |
| Robinhood Chain (`4663`) | `0xB477751B76CF82d00a686A1232f5fCD772414Af3` |
| Arc (`5042`) | `0xA4072583658Fae592A3506A42431cb6316a8d40b` |
| Berachain (`80094`) | `0xf909c4Ae16622898b885B89d7F839E0244851c66` |
| Plume (`98866`) | `0x6f5C8Bb0C5Fe4ECeAC40EE1C238EaB6bbb29761c` |
| Katana (`747474`) | `0xC59fe32C9549e3E8B5dCcdAbC45BD287Bd5bA2bc` |

### Bungee (`bungee`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`), BNB Chain (`56`), Gnosis (`100`), Unichain (`130`), Polygon (`137`), Monad (`143`), Sonic (`146`), Base (`8453`), Arbitrum (`42161`), Avalanche (`43114`), Berachain (`80094`), Plume (`98866`) | `0x5e0f8e7337c8955d2124b8e85ca74af884b3e124` |

### OmniBridge (`omnibridge`)

| Chains | Contract address |
| --- | --- |
| Ethereum (`1`) | `0xe00c629aFaCCb0510995A2B95560E446A24c85B9` |
| BNB Chain (`56`) | `0x073C8a225c8Cf9d3f9157F5C1a1DbE02407f5720` |
| Polygon (`137`), Base (`8453`), Arbitrum (`42161`) | `0xd025b38762B4A4E36F0Cde483b86CB13ea00D989` |
| HyperEVM (`999`) | `0xf353b40fC144d1c6c5BCdda712fa6De833016aF9` |
| Abstract (`2741`) | `0xd2490A00bDB97C1EDE4fdf207CFE2664AFB9C20D` |

### Hyperion (`hyperion`)

| Chains | Contract address |
| --- | --- |
| Aptos (`637`) | `0x8b4a2c4bb53857c718a04c020b98f8c2e1f99a68b0f57389a8bf5434cd22e05c` |

### NEAR Smart (`near-smart`)

| Chains | Contract account ID |
| --- | --- |
| NEAR | `v2.ref-finance.near` |

### NEAR SmartX (`near-smartx`)

| Chains | Contract account ID |
| --- | --- |
| NEAR | `aggregatedex.near` |

### RHEA (`rhea`) and RHEA Executor (`rhea-executor`)

The non-Tempo deployment group comprises Ethereum (`1`), BNB Chain (`56`), Gnosis (`100`), Unichain (`130`), Polygon (`137`), Monad (`143`), Sonic (`146`), X Layer (`196`), HyperEVM (`999`), Pharos (`1672`), Abstract (`2741`), Robinhood Chain (`4663`), Arc (`5042`), Base (`8453`), Arbitrum (`42161`), Avalanche (`43114`), Berachain (`80094`), Plume (`98866`), and Katana (`747474`). Tempo (`4217`) uses separate deployments.

| Contract | Chains | Address |
| --- | --- | --- |
| `rhea` | Non-Tempo deployment group above | `0xF206996f989b0Ef9aC7e3c469Bd123081376bE5F` |
| `rhea` | Tempo (`4217`) | `0x2e00b148E3Cc54fD923ee3D1a29a4F26E6deD09F` |
| `rhea-executor` | Non-Tempo deployment group above | `0xF8B87cbB6f13DCDEc57a1196c8982741fFa20Db8` |
| `rhea-executor` | Tempo (`4217`) | `0xd0F6eCF6372F36BE3C40c00CAa4133e0C10dCCcB` |

### RHEA Aptos (`rhea-aptos`, currently disabled)

This integration is disabled in the current configuration.

| Chains | Contract address |
| --- | --- |
| Aptos (`637`) | `0xfc1e275a813dfefd49d4995818ac8ea7bbd0089ff28d5e0e1762f372e8cb0d5f` |

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

To request API access and an API key, sign in to the [RHEA Boss portal](https://boss.rhea.finance/#/login). Provide the issued credential through `apiKey` for a static credential or `getAccessToken` when your application refreshes access tokens.

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
  sameChainTimeoutMs: 500,
  crossChainTimeoutMs: 3000,
  excludeDexs: ["lifi", "rango"],
  sender: "0xYourBaseAddress",
  recipient: "YourSolanaAddress",
};

const quote = await client.quote(quoteRequest);
```

`quote()` calls `POST /api/v2/swap/quote`. The frontend can configure the Near Intents wait plus the same-chain and cross-chain route timeouts on every `QuoteRequest`. The SDK sends the defaults shown above when fields are omitted.

Set `confidentiality: "basic"` to use the confidential 1Click route. The SDK preserves it through quote and build, and includes it in automatic or manual report payloads. Omit the field for public swaps:

```ts
const confidentialQuote = await client.quote({
  ...quoteRequest,
  confidentiality: "basic",
});
```

#### 3.3.1 Configure quote timing

`quoteWaitingTimeMs`, `sameChainTimeoutMs`, and `crossChainTimeoutMs` are first-class SDK parameters. Pass them directly to `client.quote()`; do not put them in `extensions` or executor configuration.

```ts
// Frontend: allow Near Intents up to 5 seconds to return a route quote.
const quote = await client.quote({
  ...quoteRequest,
  quoteWaitingTimeMs: 5000,
  sameChainTimeoutMs: 750,
  crossChainTimeoutMs: 6000,
});
```

| Field | SDK default | Scope |
| --- | --- | --- |
| `quoteWaitingTimeMs` | `3000` | Wait window for Near Intents and similar intent-based route quotes, including MCA previews produced by the quote call. |
| `sameChainTimeoutMs` | `500` | Timeout budget supplied to same-chain quote routing. |
| `crossChainTimeoutMs` | `3000` | Timeout budget supplied to cross-chain quote routing. |

All three values use milliseconds and must be non-negative integers. They affect only `POST /api/v2/swap/quote`; the SDK removes them from the subsequent build request. They do **not** control the SDK HTTP timeout, wallet signing, source-chain confirmation, bridge settlement, or order polling. Keep the client's `timeoutMs` above the configured quote budget plus network overhead.

#### 3.3.2 Exclude routing providers

`excludeDexs` is an optional first-class `QuoteRequest` field. Pass a string array directly to `client.quote()` to exclude routing providers from quote selection. Omit it when no provider exclusion is needed. Use the exact lowercase route identifiers below.

| Route identifier |
| --- |
| `bitget` |
| `okx` |
| `binance` |
| `1inch` |
| `paraswap` |
| `kodiak` |
| `openocean` |
| `cow` |
| `zerox` |
| `jupiter` |
| `titan` |
| `dflow` |
| `hyperion` |
| `near-smart` |
| `near-smartx` |
| `nearintents` |
| `preswap-nearintents` |
| `rango` |
| `lifi` |
| `jumper` |
| `bungee` |
| `swapkit` |
| `omnibridge` |
| `stableflow` |
| `kyber` |

For example, exclude Bitget, OKX, LI.FI, Rango, and Kyber:

```ts
const quote = await client.quote({
  ...quoteRequest,
  excludeDexs: ["bitget", "okx", "lifi", "rango", "kyber"],
});

const result = await client.swap({ quote });
```

The SDK forwards the array as the top-level `excludeDexs` field in the quote request body. Use an array such as `["lifi", "rango"]`; pass this field directly on `QuoteRequest` rather than in `extensions` or executor configuration. Include the same array when requesting a refreshed quote to keep the same exclusions.

Route exclusions can be configured independently of `quoteWaitingTimeMs`, `sameChainTimeoutMs`, and `crossChainTimeoutMs`. Excluding providers may leave no applicable route for a given pair. Route availability depends on the source/destination chains, tokens, amount, and liquidity; the supported list does not guarantee every route is available for every quote request.

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

`"completed"` polls the order-status API whenever the swap has a queryable order reference. Confidential swaps also require order-status polling when they are same-chain. For confidential Near Intents builds, the SDK uses `deposit.orderId` when present and otherwise uses `deposit.depositAddress` as the status key. If a cross-chain or confidential swap does not provide a usable status key, the SDK throws `INVALID_API_RESPONSE` at the `status` stage instead of treating source-chain confirmation as completion.

The default polling interval is 5 seconds. There is no default polling timeout; an explicit timeout throws `ORDER_TIMEOUT`, but it does not revert an already submitted on-chain transaction.

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
| `apiKey` | `string` | No | Static API credential sent with requests. Request one through the [RHEA Boss portal](https://boss.rhea.finance/#/login). |
| `getAccessToken` | `() => string \| Promise<string>` | No | Reads an access token before each request. Use this for refreshable sessions. |
| `fetch` | `typeof globalThis.fetch` | No | Custom fetch implementation. The SDK binds its invocation context to avoid browser `Illegal invocation` errors. It is normally required on Node.js 16. |
| `headers` | `Record<string,string>` or function | No | Additional request headers. The function form may return a promise. |
| `timeoutMs` | `number` | No | Timeout for each HTTP request in milliseconds. Default: `15000`. |
| `retry` | `Partial<RetryConfig>` | No | Retry policy for retryable quote/read operations. Defaults: 2 retries, 250ms base delay, 2000ms maximum delay, and jitter enabled. |
| `logger` | `SdkLogger` | No | Receives structured `api.request`, `api.response`, and `api.retry` entries. |
| `executors` | `readonly ChainExecutor[]` | Required for execution | Wallet executors. May be omitted when only calling `quote()` or `buildSwap()`. |
| `maxQuoteAgeMs` | `number \| null` | No | Maximum local quote age in milliseconds. Default: `30000`. Set to `null` to disable the local age check; an API-provided `expiresAt` still applies. |
| `tokenListCacheTtlMs` | `number` | No | Successful token-list cache lifetime in milliseconds. Default: `600000` (10 minutes). Set to `0` to disable. |
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
| `quoteWaitingTimeMs` | `number` | No | Near Intents quote wait in milliseconds. Must be a non-negative integer. Default: `3000`. See [Configure quote timing](#331-configure-quote-timing). |
| `sameChainTimeoutMs` | `number` | No | Same-chain route quote timeout in milliseconds. Must be a non-negative integer. Default: `500`. |
| `crossChainTimeoutMs` | `number` | No | Cross-chain route quote timeout in milliseconds. Must be a non-negative integer. Default: `3000`. |
| `excludeDexs` | `string[]` | No | Routing providers to exclude from quote selection, for example `["lifi", "rango", "kyber"]`. |
| `confidentiality` | `"basic"` | No | Enables the confidential 1Click route and marks the resulting report. Omit for public swaps. |
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
| `completed` | After source execution, polls the server until a delivery terminal state. Same-chain confidential swaps are included. A cross-chain or confidential swap without a usable status key fails at the `status` stage instead of being reported as completed. |

### 4.5 SwapExecutionResult

| Field | Type | Description |
| --- | --- | --- |
| `executionId` | `string` | Unique identifier for this SDK execution. |
| `status` | `string` | `submitted`, `source-confirmed`, `processing`, `completed`, `failed`, `refunded`, or `expired`. |
| `router` | `string` | Router used for execution. Pass it unchanged when querying order status. |
| `txHash` | `string?` | Hash of a single source-chain transaction. |
| `txHashes` | `string[]?` | Hashes of multiple source-chain transactions, such as a NEAR transaction batch. |
| `orderId` | `string?` | Status-query key. Usually a server order identifier; confidential Near Intents may use the deposit address. When present, it can be passed to `waitForOrder()`. |
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
import { resolveMcaWithdrawPolicy } from "@rhea-finance/cross-chain-aggregation-dex";

const collateral = resolveMcaWithdrawPolicy({
  amountBurrow,      // requested withdraw in Burrow internal decimals
  suppliedBalance,   // current token supplied balance, same decimals
  availableBalance,  // human/display precision, matching amountInHuman
  amountIn: amountInHuman,
  isMax,
});

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
  collateral,
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
| `collateral.needDecrease` | `boolean` | No | Compatibility hint only. The SDK derives the API value from `decreaseAmountBurrow` and corrects contradictory input. |
| `collateral.decreaseAmountBurrow` | `string` | Yes | Required collateral decrease in Burrow decimals: `max(amountBurrow - suppliedBalance, 0)`. The SDK canonicalizes the value and sends `"0"` when supplied balance covers the withdrawal. |
| `collateral.withdrawAll` | `boolean` | No | Whether to withdraw the full available amount. |
| `executionPreference` | `"auto" \| "near" \| "relayer"` | No | Default: `"auto"`. Set explicitly to force direct NEAR or relayer execution. |
| `boundNearAccountId` | `string` | Required for automatic NEAR selection | In `auto` mode, direct NEAR execution is selected only when `toChain === "near"` and `recipient` exactly matches this field. Otherwise, the relayer is selected. |

For direct NEAR execution, the NEAR executor submits `nearMcaWithdrawTx`. For relayer execution, the executor selected by `signerChain` uses `signMessage()` to sign the API-provided `messageToSign`, after which the SDK submits the order. Application code only calls `swap()`.

Derive the collateral policy from the same balances displayed by the application, as shown above. This mirrors Lending Withdraw and multi-chain Trade's 2026-08-20 withdraw fix: `decreaseCollateralAmountBurrow = max(amountBurrow - suppliedBalance, 0)`, and `needDecreaseCollateral` is true exactly when that result is positive. When relayer gas is reserved, use the supplied balance from the gas-adjusted portfolio snapshot, matching the amount used for the quote. `withdrawAll` remains an independent Max/available-balance decision.

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
await client.createHistoryAuthChallenge(rawChallengeRequest);
await client.verifyHistoryAuthChallenge(rawVerifyRequest);
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

For confidential history, authorize the connected wallet, then query with the returned short-lived token:

```ts
const authorization = await client.authorizeConfidentialHistory(
  {
    chainFamily: "evm",
    chainId: "1",
    walletAddress: connectedAddress,
    // Include mcaAccountId when authorizing an MCA principal.
  },
  async (challenge) => {
    // Check the wallet/network has not changed, then sign exactly this message.
    const signature = await signer.signMessage(
      challenge.signingInput.message
    );
    return { signature };
  }
);

const confidentialHistory = await client.getHistory({
  sender: authorization.queryAddress,
  mode: "confidential",
  walletToken: authorization.token,
  page: 1,
  pageSize: 20,
});
```

The callback returns the wallet-specific proof required by `challenge.signingMethod`; NEP-413 and other chain families may require additional proof fields. `authorizeConfidentialHistory()` validates that the challenge and verified token refer to the requested principal before returning them. The SDK sends the normal API credential in `Authorization` and the wallet token separately in `Authentication`. Public history omits `mode` and `walletToken`.

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
