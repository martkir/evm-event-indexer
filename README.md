# evm-event-indexer

# WebSocket Service

Real-time EVM event streaming service for subscribing to blockchain events with optional decoding.

## Overview

The WebSocket service enables clients to receive real-time notifications for specific EVM events. You can subscribe to events by specifying contract addresses, event signatures (topic_0), and whether you want the events decoded or returned in raw format.

## Connection

Connect to the WebSocket endpoint:

```
wss://stream.yourdomain.com/v1/events
```

### Using wscat

Install wscat if you haven't already:

```bash
npm install -g wscat
```

Connect to the service:

```bash
wscat -c wss://stream.yourdomain.com/v1/events
```

## Subscription Message

After connecting, send a subscription message to start receiving events.

### Request Format

```json
{
  "type": "subscribe",
  "chain": "ethereum",
  "decode": true,
  "filters": [
    {
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "topic_0": "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
    }
  ]
}
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | Yes | Must be `"subscribe"` to initiate a subscription |
| `chain` | string | Yes | The blockchain network (e.g., `"ethereum"`, `"polygon"`, `"arbitrum"`) |
| `decode` | boolean | Yes | Whether to decode events (`true`) or return raw logs (`false`) |
| `filters` | array | Yes | Array of filter objects specifying which events to listen for |
| `filters[].address` | string | Yes | Contract address to monitor (checksummed or lowercase) |
| `filters[].topic_0` | string | Yes | Event signature hash (keccak256 of event signature) |

## Response Format

The service streams events matching your filters in real-time. Each message includes the block number and an array of events from that block.

### Decoded Events Response

When `decode: true`, events are returned with parsed parameters:

```json
{
  "block_number": 18500123,
  "data": [
    {
      "event_name": "PoolCreated",
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "transaction_hash": "0xabc123...",
      "log_index": 42,
      "parameters": {
        "token0": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "token1": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "fee": 3000,
        "tickSpacing": 60,
        "pool": "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8"
      }
    }
  ]
}
```

### Raw Events Response

When `decode: false`, events are returned as raw log objects:

```json
{
  "block_number": 18500123,
  "data": [
    {
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "topics": [
        "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118",
        "0x000000000000000000000000a0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "0x000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2"
      ],
      "data": "0x0000000000000000000000000000000000000000000000000000000000000bb8000000000000000000000000000000000000000000000000000000000000003c0000000000000000000000008ad599c3a0ff1de082011efddc58f1908eb6e6d8",
      "block_hash": "0xdef456...",
      "block_number": 18500123,
      "transaction_hash": "0xabc123...",
      "transaction_index": 15,
      "log_index": 42,
      "removed": false
    }
  ]
}
```

### Response Fields

#### Common Fields

| Field | Type | Description |
|-------|------|-------------|
| `block_number` | integer | The block number where the event(s) occurred |
| `data` | array | Array of event objects (decoded or raw based on subscription) |

#### Decoded Event Fields

| Field | Type | Description |
|-------|------|-------------|
| `event_name` | string | Human-readable name of the event |
| `address` | string | Contract address that emitted the event |
| `transaction_hash` | string | Transaction hash containing the event |
| `log_index` | integer | Index of the log within the block |
| `parameters` | object | Decoded event parameters as key-value pairs |

#### Raw Event Fields

| Field | Type | Description |
|-------|------|-------------|
| `address` | string | Contract address that emitted the event |
| `topics` | array | Array of indexed event parameters (topic_0 is event signature) |
| `data` | string | Non-indexed event parameters as hex-encoded data |
| `block_hash` | string | Hash of the block containing this log |
| `block_number` | integer | Block number containing this log |
| `transaction_hash` | string | Transaction hash containing the event |
| `transaction_index` | integer | Transaction's index position in the block |
| `log_index` | integer | Log's index position in the block |
| `removed` | boolean | Whether the log was removed due to chain reorganization |

## Example Usage

### Listening to Uniswap V3 Factory Pool Creation (Decoded)

```bash
# Connect
wscat -c wss://stream.yourdomain.com/v1/events

# Send subscription
> {
  "type": "subscribe",
  "chain": "ethereum",
  "decode": true,
  "filters": [
    {
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "topic_0": "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
    }
  ]
}

# Receive events
< {
  "block_number": 18500123,
  "data": [
    {
      "event_name": "PoolCreated",
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "transaction_hash": "0xabc123...",
      "log_index": 42,
      "parameters": {
        "token0": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "token1": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
        "fee": 3000,
        "tickSpacing": 60,
        "pool": "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8"
      }
    }
  ]
}
```

### Multiple Event Subscriptions

You can subscribe to multiple events in a single connection:

```json
{
  "type": "subscribe",
  "chain": "ethereum",
  "decode": true,
  "filters": [
    {
      "address": "0x1F98431c8aD98523631AE4a59f267346ea31F984",
      "topic_0": "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
    },
    {
      "address": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",
      "topic_0": "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"
    }
  ]
}
```

## Notes

- The WebSocket connection will stream events in real-time as they are confirmed on the blockchain
- Events from the same block are batched together in the response
- Ensure your event signature (topic_0) matches the keccak256 hash of the event signature (e.g., `PoolCreated(address,address,uint24,int24,address)`)
- For decoded events, the service must have the ABI for the contract to properly decode parameters
