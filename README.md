# Base RPC: Canonical State, Pending State, and Fee-Aware Reads

Base exposes an Ethereum-compatible JSON-RPC API, but current Base documentation adds useful L2-specific semantics around block tags and pre-confirmed state. This guide keeps the main path on a standard Base RPC endpoint and calls out where Base's separate Flashblocks interface changes the meaning of `pending`.

For standard Base Mainnet RPC calls:

```bash
export BASE_RPC=https://base.api.onfinality.io/public
```

Base Mainnet chain ID is `8453` (`0x2105`).

## First verify the chain

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Expected result:

```json
{"jsonrpc":"2.0","id":1,"result":"0x2105"}
```

## Read the chain at the confirmation level you actually need

Base's JSON-RPC documentation accepts the familiar EVM block tags such as `latest`, `safe`, and `finalized` on state/block methods.

Latest block:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockByNumber",
    "params":["latest",false]
  }'
```

Safer view:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockByNumber",
    "params":["safe",false]
  }'
```

For an application, this is more expressive than using "latest minus N blocks" everywhere.

## Nonces: `latest` and `pending` answer different questions

The next nonce for an account comes from `eth_getTransactionCount`.

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionCount",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

`latest` is appropriate when you want the nonce derived from confirmed chain state.

Base also operates a separate Flashblocks/pre-confirmation endpoint. Base's documentation notes that querying that interface with the `pending` tag can include pre-confirmed transactions. Do not assume a normal provider endpoint and a Flashblocks endpoint expose identical `pending` semantics.

That distinction matters for high-frequency transaction senders: a stale nonce view can create nonce gaps or replacement conflicts.

## Read Base fee history

Base supports Ethereum-style fee RPC methods. `eth_feeHistory` is useful when building a fee estimator from recent blocks:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_feeHistory",
    "params":["0xA","latest",[10,50,90]]
  }'
```

You can also query a suggested priority fee:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_maxPriorityFeePerGas",
    "params":[]
  }'
```

Fee-related methods help construct the L2 transaction, but an L2's total transaction economics should not be assumed to be identical to Ethereum Mainnet.

## Pull every receipt for one block

Base documents `eth_getBlockReceipts`, which is convenient for block-oriented pipelines:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockReceipts",
    "params":["latest"]
  }'
```

For an indexer that needs every receipt anyway, this can be more natural than fetching each transaction receipt one by one. Provider support and response-size limits can still vary, so production code should handle method/provider capability differences.

## Contract reads and logs are ordinary EVM calls

Read contract state:

```bash
curl -s "$BASE_RPC" \
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

Query logs:

```bash
curl -s "$BASE_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"0xEND_BLOCK",
      "address":"0xCONTRACT_ADDRESS"
    }]
  }'
```

If you are building an indexer, use bounded ranges and store a checkpoint. L2 chains can advance quickly enough that "scan from genesis to latest" is not a sensible recurring query.

## Use Base's chain definition with viem

```bash
npm install viem
```

```js
import {createPublicClient, http} from 'viem';
import {base} from 'viem/chains';

const client = createPublicClient({
  chain: base,
  transport: http('https://base.api.onfinality.io/public'),
});

const [chainId, blockNumber, gasPrice] = await Promise.all([
  client.getChainId(),
  client.getBlockNumber(),
  client.getGasPrice(),
]);

console.log({chainId, blockNumber, gasPrice});
```

Using a chain definition also gives wallet tooling the correct chain ID, native currency metadata, and explorer defaults.

## Standard RPC vs Flashblocks

They solve different problems:

- **Standard RPC** is the general interface for blocks, state, logs, simulation, receipts, and normal application traffic.
- **Flashblocks/pre-confirmation RPC** is for applications that explicitly need Base's faster pre-confirmed view of pending state.

This repository uses the standard path because it is the portable default. If your application depends on sub-block latency, read Base's Flashblocks documentation and design the state machine around the weaker confirmation guarantee instead of silently treating pre-confirmation as finality.

## References

- [Base Ethereum JSON-RPC API](https://docs.base.org/base-chain/api-reference/ethereum-json-rpc-api/eth_chainId)
- [Base documentation](https://docs.base.org/)
- [OnFinality Base RPC](https://onfinality.io/en/networks/base)
- [OnFinality network directory](https://onfinality.io/en/networks)
