# Base RPC Getting Started

This guide covers connecting to Base Mainnet and making Ethereum-compatible JSON-RPC calls.

## RPC endpoint

```text
https://base.api.onfinality.io/public
```

Base is an OP Stack Layer 2 with chain ID `8453`.

## 1. Verify the network

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Base Mainnet returns `0x2105`.

## 2. Get the latest block

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

## 3. Read ETH balance

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

ETH is the native gas token on Base.

## 4. Inspect recent fee history

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_feeHistory",
    "params":["0x5","latest",[25,50,75]]
  }'
```

`eth_feeHistory` is useful when building fee-aware EIP-1559-style transactions.

## 5. Make a read-only contract call

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_call",
    "params":[
      {"to":"0xCONTRACT_ADDRESS","data":"0xABI_ENCODED_CALLDATA"},
      "latest"
    ]
  }'
```

## 6. Query logs

```bash
curl -s https://base.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"latest",
      "address":"0xCONTRACT_ADDRESS"
    }]
  }'
```

For indexers, use bounded ranges and checkpoint the last processed block.

## 7. JavaScript example

```js
const RPC_URL = 'https://base.api.onfinality.io/public';

async function rpc(method, params = []) {
  const res = await fetch(RPC_URL, {
    method: 'POST',
    headers: {'content-type': 'application/json'},
    body: JSON.stringify({jsonrpc: '2.0', id: 1, method, params}),
  });

  const body = await res.json();
  if (body.error) throw new Error(body.error.message);
  return body.result;
}

const chainId = Number.parseInt(await rpc('eth_chainId'), 16);
const block = Number.parseInt(await rpc('eth_blockNumber'), 16);
console.log({chainId, block});
```

## Troubleshooting

### Contract works on Ethereum but fails on Base

Deployment addresses and state are chain-specific. Verify the contract is actually deployed on Base.

### Large log queries fail

Reduce the block range and retry smaller windows.

### Fee assumptions are wrong

Base is an L2. Do not assume total transaction-cost behavior is identical to Ethereum Mainnet simply because the RPC interface is compatible.

## Mainnet settings

| Setting | Value |
| --- | --- |
| Network | Base Mainnet |
| Chain ID | `8453` |
| Native token | ETH |
| RPC | `https://base.api.onfinality.io/public` |
| Explorer | `https://basescan.org` |

## Resources

- [Base documentation](https://docs.base.org/)
- [OnFinality Base RPC](https://onfinality.io/en/networks/base)
- [OnFinality RPC network directory](https://onfinality.io/en/networks)
