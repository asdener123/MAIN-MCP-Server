# MAIN MCP Server

MCP (Model Context Protocol) server for the MAIN DEX on Base. Provides AI agents (Claude, Cursor, etc.) with tools to interact with the protocol: swap tokens, manage liquidity, enter/exit ALM strategies, and more.

## Connecting to Claude

Claude Code:

```bash
claude mcp add main-mcp --transport http https://api.main.exchange/mcp
```

## MCP Protocol

The server implements [MCP spec 2025-03-26](https://modelcontextprotocol.io) over Streamable HTTP transport.

```bash
# Initialize
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}'

# List tools
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' \
  | jq '.result.tools[] | {name, description}'

# Health
curl https://api.main.exchange/mcp/health
```

## x402 Payments

Some tools require USDC payment via the [x402 protocol](https://x402.org). When a paid tool is called without payment, the server responds with a JSON-RPC error containing payment requirements (amount, recipient, network). x402-compatible clients handle this automatically.

**Paid tools**: `createPool`, `launchToken`, `getPoolsTerminal`, `getPoolTerminal`, `getTokenTerminal`, `getTokenHolders`, `getTokenSentiment`, `getForecastSummary`

## Tools

### Tokens

**getTokens** — Get all tokens in the protocol sorted by trading volume.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getTokens","arguments":{}}}' \
  | jq '.result.content[0].text | fromjson'
```

**approveTokens** — Prepare an ERC20 approve transaction.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"approveTokens","arguments":{"tokenAddress":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913","spenderAddress":"0x002E67c3F7BF96eB3aA4066073923e415581d385"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getTokenTerminal** *(paid via x402)* — Get detailed token info from the api-terminal: price, market cap, supply, liquidity, volume, transactions, price changes, and social links.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getTokenTerminal","arguments":{"tokenAddress":"0x4200000000000000000000000000000000000006"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getTokenHolders** *(paid via x402)* — Get top 100 holders of a token from the api-terminal: address, label, amount, percentage, and USD value.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getTokenHolders","arguments":{"tokenAddress":"0x4200000000000000000000000000000000000006"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getTokenSentiment** *(paid via x402)* — Get token sentiment analysis from the api-terminal: narrative (summary, catalysts, risks) and community mentions (mood, sentiment breakdown, posts from X/Twitter).

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getTokenSentiment","arguments":{"symbol":"WETH"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getForecastSummary** *(paid via x402)* — Get price forecast summary for an asset (BTC, ETH, SOL) from the api-terminal: outlook, predicted price, support/resistance levels, and probability distribution.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getForecastSummary","arguments":{"asset":"BTC","horizon":"24h"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**launchToken** *(paid via x402)* — Launch a new ERC-20 token on MAIN. Returns calldata for TokenFactory.createToken. Logo and banner must be uploaded to IPFS beforehand (see IPFS upload endpoint in tool description). All token data is immutable after creation.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"launchToken","arguments":{"name":"My Token","symbol":"MTK","totalSupply":"10000000","logo":"bafkreibgxsqxrkrha3oqoatypp4gpuemft47ablrgxkfucqn2rvhsdwaka","description":"A cool token","website":"https://example.com"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Trading

**getPrice** — Get a swap quote: how much tokenOut for a given amount of tokenIn.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPrice","arguments":{"amount":"1000000000000000000","tokenIn":"0x4200000000000000000000000000000000000006","tokenOut":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**quoteSwap** — Get a full swap quote with calldata for execution via SwapRouter.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"quoteSwap","arguments":{"amountIn":"1000000","tokenIn":"0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913","tokenOut":"0x4200000000000000000000000000000000000006","recipient":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Pools

**getPoolsTerminal** *(paid via x402)* — Get the list of pools on Base from the api-terminal with market data: TVL, 24h volume, token prices, and price changes.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPoolsTerminal","arguments":{}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getPoolTerminal** *(paid via x402)* — Get detailed information about a specific pool from the api-terminal, including token metadata (symbol, name, image).

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPoolTerminal","arguments":{"poolAddress":"0x70acdf2ad0bf2402c957154f944c19ef4e1cbae1"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getPools** — Get liquidity pools with TVL, volume, and fees.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPools","arguments":{}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getPoolAPR** — Get pool APR (all pools or a specific one).

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPoolAPR","arguments":{}}}' \
  | jq '.result.content[0].text | fromjson'
```

**addLiquidityToPool** — Prepare a transaction to add liquidity to an Algebra pool.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"addLiquidityToPool","arguments":{"poolAddress":"0xPOOL","amount0":"1000000000000000000","amount1":"1000000","userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**removeLiquidityFromPool** — Prepare a transaction to remove liquidity.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"removeLiquidityFromPool","arguments":{"tokenId":"123","percentage":100,"userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**createPool** *(paid via x402)* — Create and initialize a new CLMM pool on MAIN. Accepts tokens in any order and a human-readable price. The tool fetches token decimals on-chain, sorts tokens, calculates `sqrtPriceX96`, and checks that the pool doesn't already exist.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"createPool","arguments":{"tokenA":"0xTOKEN_A","tokenB":"0xTOKEN_B","price":"0.1"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Positions

**getPositions** — Get all open liquidity positions for an address.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getPositions","arguments":{"userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Smart Wallet

ALM strategies are executed through a smart wallet — a deterministic proxy contract deployed per user. The smart wallet must exist on-chain before entering any strategy.

**deploySmartWallet** — Deploy a smart wallet for the user. Must be called once before using any strategy tools. The transaction must be sent directly from the user's EOA (the factory uses `msg.sender` as the deployment salt).

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"deploySmartWallet","arguments":{"userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Strategies

**getStrategies** — Get available ALM strategies with metrics.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getStrategies","arguments":{}}}' \
  | jq '.result.content[0].text | fromjson'
```

**getStrategyPositions** — Get user positions in ALM strategies.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"getStrategyPositions","arguments":{"userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**enterToStrategy** — Enter an ALM strategy. Returns calldata for smart wallet. Requires a deployed smart wallet (use `deploySmartWallet` first).

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"enterToStrategy","arguments":{"strategyId":"alm-v1","amount0":"500000","amount1":"0","userAddress":"0xYOUR_ADDRESS","duration":86400,"stopLoss":10,"takeProfit":25}}}' \
  | jq '.result.content[0].text | fromjson'
```

**exitFromStrategy** — Exit an ALM strategy. Requires a deployed smart wallet.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"exitFromStrategy","arguments":{"strategyId":"alm-v1","positionId":"5","percentage":10000,"userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

**increaseStrategy** — Increase a position in an ALM strategy. Requires a deployed smart wallet.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"increaseStrategy","arguments":{"strategyId":"alm-v1","positionId":"5","amount0":"500000","amount1":"0","userAddress":"0xYOUR_ADDRESS"}}}' \
  | jq '.result.content[0].text | fromjson'
```

### Broadcast

**broadcastTx** — Broadcast a signed transaction to the blockchain.

```bash
curl -s -X POST https://api.main.exchange/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"broadcastTx","arguments":{"signedTxHex":"0xSIGNED_TX"}}}' \
  | jq '.result.content[0].text | fromjson'
```
