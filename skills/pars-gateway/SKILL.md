---
name: pars-gateway
description: >
  Use this skill for keyless blockchain access via x402 USDC micropayments.
  No API key or account needed. Agents can sign up and pay autonomously
  using an onchain wallet. Supports the same 100+ chains as pars-api.
metadata:
  author: parsnetwork
  version: "1.0"
  bot:
    emoji: "\U0001F916"
    requires:
      env: [PARS_WALLET_PRIVATE_KEY]
---

# Agentic Gateway Skill

## When to use

Use this skill when:
- The agent needs keyless blockchain access without a Pars Cloud account
- The agent should pay per request with USDC (x402 protocol)
- Building autonomous agents that sign up and pay independently
- No human needs to create an API key or manage billing

If `PARS_API_KEY` is set, use the `pars-api` skill instead.

## Hard requirements

1. **Wallet required.** `PARS_WALLET_PRIVATE_KEY` must be set to an Ethereum private key that holds USDC on Base.
2. **USDC on Base.** The wallet must have USDC balance on Base network for x402 payments.
3. **Never expose the private key** in user-visible output, logs, or screenshots.
4. **If `PARS_API_KEY` is set**, prefer the `pars-api` skill (it's cheaper for high volume).

## Preflight checks

Before making any request, silently verify:
1. `PARS_WALLET_PRIVATE_KEY` is set and non-empty
2. If `PARS_API_KEY` is also set, warn that `pars-api` skill may be more cost-effective

## How x402 works

1. Agent sends request to gateway without auth
2. Gateway responds with `402 Payment Required` + payment details
3. Agent signs USDC payment authorization (EIP-712)
4. Agent resends request with `X-Payment` header containing signed authorization
5. Gateway verifies signature, deducts USDC, processes request
6. Response returned as normal

## Quick reference

| Item | Value |
|------|-------|
| Gateway URL | `https://gateway.cloud.pars.network/v1` |
| Payment token | USDC on Base (0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913) |
| Payment protocol | x402 (EIP-712 signed authorization) |
| Auth method | SIWE (Sign-In with Ethereum) |
| Docs | https://cloud.pars.network/docs/agents |
| Status | https://status.cloud.pars.network |

## One-file quickstart

### 1. Generate a wallet (if needed)

```bash
# Using cast (foundry)
cast wallet new

# Save the private key
export PARS_WALLET_PRIVATE_KEY=0x...
```

### 2. Fund with USDC on Base

Transfer USDC to your wallet address on Base network. Even $1 covers thousands of requests.

### 3. Make a request

```bash
# First request gets 402 with payment details
curl -X POST https://gateway.cloud.pars.network/v1/eth/mainnet \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Response: 402 Payment Required with X-Payment-Details header
# Agent SDK handles payment signing automatically
```

### Using the Pars Cloud Agent SDK

```typescript
import { ParsAgent } from "@parsnetwork/agent-sdk"

const agent = new ParsAgent({
  privateKey: process.env.PARS_WALLET_PRIVATE_KEY,
})

// Automatically handles x402 payment flow
const blockNumber = await agent.rpc("eth/mainnet", {
  method: "eth_blockNumber",
  params: [],
})
```

## Pricing

| Request type | Cost (USDC) |
|-------------|-------------|
| Simple query (eth_blockNumber, eth_gasPrice) | $0.000001 |
| Standard query (eth_getBalance, eth_getBlock) | $0.000005 |
| Compute query (eth_call, eth_estimateGas) | $0.00001 |
| Log query (eth_getLogs) | $0.000025 |
| Write (eth_sendRawTransaction) | $0.00005 |
| Debug/Trace | $0.0001 |
| Data API (tokens, NFTs) | $0.00001 |

## Endpoint selector

Same endpoints as `pars-api` skill, but using gateway URL:

| Task | Endpoint |
|------|----------|
| JSON-RPC | `POST https://gateway.cloud.pars.network/v1/{network}` |
| Token data | `GET https://gateway.cloud.pars.network/v1/data/tokens/*` |
| NFT data | `GET https://gateway.cloud.pars.network/v1/data/nfts/*` |
| Portfolio | `GET https://gateway.cloud.pars.network/v1/data/portfolio` |

## Error handling

| Code | Meaning | Action |
|------|---------|--------|
| 402 | Payment required | Sign payment and retry with X-Payment header |
| 401 | Invalid signature | Re-sign with correct wallet |
| 402 + insufficient | Wallet balance too low | Fund wallet with more USDC |
| 429 | Rate limited | Backoff and retry |

## Official links

- Agent docs: https://cloud.pars.network/docs/agents
- x402 protocol: https://x402.org
- Dashboard: https://cloud.pars.network/dashboard
- Powered by [Hanzo AI](https://hanzo.ai)
