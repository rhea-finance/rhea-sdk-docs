# 1. What Is Rhea Aggregated DEX?

**Rhea Aggregated DEX** is a multi-liquidity-pool aggregated trading system running on the **NEAR blockchain**.  
It works through the collaboration of an **off-chain routing algorithm** and **on-chain aggregation contracts**, providing users with optimal swap paths and execution results.

## Supported Pool Types

- AMM Pools
- Stable Pools
- ALMM Pools
- CLMM Pools
- Delta Orders(Coming Soon)

## User Input

Users only need to specify:

- Which token to swap **from**
- Which token to swap **to**
- The **amount**

## The System Automatically Handles

- Multi-pool routing calculation
- Slippage control
- Sequential execution across multiple contracts
- Safety fallback and balance management

---

# 2. Overall Workflow Overview

A complete swap process consists of **four stages**:

**step1.SmartRouter API calculates the optimal path**  
↓  
**step2.Token storage registration check**  
↓  
**step3.ft_transfer_call** → Aggregated contract executes the swap  
↓  
**step4.Balance query**

---

# 3. Contract Address

**Aggregated DEX Contract: [aggregatedex.near](https://nearblocks.io/zh-cn/address/aggregatedex.near)**

---

# 4. Step 1: Call SmartRouter API to Calculate the Swap Path

The **SmartRouter API** is used to calculate the optimal swap route and generate parameters that can be executed directly on-chain.

## 4.1 API Endpoint

GET https://smartx.rhea.finance/swapMultiDexPath

## 4.2 Query Parameters

| Parameter | Type | Required | Description |
|----------|------|----------|-------------|
| amountIn | string | Yes | Input token amount (smallest unit, integer) |
| tokenIn | string | Yes | Input token contract address / AccountId |
| tokenOut | string | Yes | Output token contract address / AccountId |
| slippage | number | Yes | Slippage tolerance, e.g. 0.005 = 0.5% |
| user | string | Optional / Required | Optional for quote, required before execution |
| receiveUser | string | Optional | Final recipient address |
| skipUnwrapNativeToken | boolean | Optional | Skip wNEAR unwrap |
| appFeeRate | number | Optional | Partner fee rate (per mille) |
| appFeeRecipient | string | Optional | Partner fee recipient |

## 4.3 Key Parameter Notes

### amountIn

Must be specified in the **smallest unit**

Example:

100 NEAR → 100000000000000000000000000

### user

Can be omitted for quotation  
Must be provided before on-chain execution

### appFeeRate & appFeeRecipient

Must be provided together  
Maximum appFeeRate: 1000 (10%)

Fee distribution:

- 20% App Fee → Rhea
- 80% App Fee → appFeeRecipient

For example: 
```
appFeeRate=10, means 0.1% app fee
appFeeRecipient=test.near

user swap $1000, generate $1 appFee
test.near will receive $0.8
rhea will receive $0.2
```

Withdraw App Fee:
[see here](https://github.com/rhea-finance/rhea-sdk-docs/blob/main/SmartRouterV2(Aggregated%20DEX).md#withdraw)

**Note That:**

**The protocol defines six fee whitelist tokens(NEAR, USDC, USDT, USDC.e, USDT.e, RHEA).**

**If a whitelist token appears in the swap route, it will be used preferentially for fee collection.
If no whitelist token is present in the route, the fee will be collected from tokenOut.**

## 4.4 API Response

{
  "amount_in": "...",
  "amount_out": "...",
  "min_amount_out": "...",
  "msg": "...",
  "signature": "...",
  "tokens": ["wrap.near", "usdt.tether-token.near"]
}

All tokens in `tokens` must be registered in advance.

---

# 5. Step 2: Token Storage Registration Check

Before swapping, ensure all tokens are registered.

## 5.1 Register on Aggregate Dex

user and appFeeRecipient account, registration is required on Aggregate Dex.

### Check Registration

near view aggregatedex.near query_user_tokens_registered '{"user":"xx","tokens":["wrap.near","17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1"]}'

### Register Tokens

near call aggregatedex.near tokens_storage_deposit '{"user":"xx","tokens":["wrap.near","17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1"]}'

Each token requires 0.005 NEAR storage fee.

## 5.2 Register on Token

### Check Registration

Ensure token storage registration for both the user, the AggregateDex contract and app fee recipient.
All tokens returned by the API must be registered in the corresponding token contracts.

### Register Tokens

near call 17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1 storage_deposit '{"account_id": "aggregatedexstg.near/user", "registration_only": true}' --deposit=0.005 --accountId=xx

---

# 6. Step 3: Execute Swap via ft_transfer_call

Swaps must be triggered via `ft_transfer_call`.

Caller: tokenIn contract  
Receiver: aggregatedex.near  
Message: msg + signature
Gas required: >= 270 Tgas  
depositYocto: 1

for example:
```
near call wrap.near ft_transfer_call  '{
  "receiver_id": "aggregatedex.near",
  "amount": "10000000000000000000000",
  "msg": "{\"msg\":\"xxx\",\"signature\":\"xxx\"}"
}' --accountId user --depositYocto 1 --gas 300000000000000
```

---

# 7. Step 4: Balance Query

If tokens are not properly registered, user funds may remain in the Aggregate contract.
The user can then query their internal balance and perform a withdraw operation.

## Query Balance

near view aggregatedex.near query_user_exist_balance '{"user": "xx", "from_index": 0, "count": 100}'

## Withdraw

near call aggregatedex.near withdraw '{"token": "xx", "return_near": true}'

---

# 8. Developer Notes

1. Token registration is mandatory
2. Swaps must use ft_transfer_call
3. Always provide balance query and withdraw fallback
4. All amounts use smallest units
5. Revalidate routing and fill user field before execution
