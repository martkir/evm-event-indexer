# evm-event-indexer

## Table of Contents

- [WebSocket Service](#websocket-service)
  - [Overview](#overview)
  - [Connection](#connection)
    - [Using wscat](#using-wscat)
  - [Subscription Message](#subscription-message)
    - [Request Format](#request-format)
    - [Request Parameters](#request-parameters)
  - [Response Format](#response-format)
    - [Decoded Events Response](#decoded-events-response)
    - [Raw Events Response](#raw-events-response)
    - [Response Fields](#response-fields)
  - [Example Usage](#example-usage)
    - [Listening to Uniswap V3 Factory Pool Creation (Decoded)](#listening-to-uniswap-v3-factory-pool-creation-decoded)
    - [Multiple Event Subscriptions](#multiple-event-subscriptions)
    - [Listen to All Contracts](#listen-to-all-contracts)
  - [Notes](#notes)
- [WebSocket Architecture Design](#websocket-architecture-design)
  - [Method: Shared Cache with Pub/Sub Broadcast](#method-shared-cache-with-pubsub-broadcast)
- [Webhook Service](#webhook-service)
  - [Overview](#overview-1)
  - [Registering a Webhook](#registering-a-webhook)
    - [Endpoint](#endpoint)
    - [Request](#request)
    - [Request Parameters](#request-parameters-1)
    - [Response](#response)
  - [Deregistering a Webhook](#deregistering-a-webhook)
  - [Webhook Payload](#webhook-payload)
    - [Decoded Events Payload](#decoded-events-payload)
    - [Raw Events Payload](#raw-events-payload)
  - [Response Requirements](#response-requirements)
  - [Example: Uniswap V3 Pool Creation](#example-uniswap-v3-pool-creation)
- [ABI Repository Service](#abi-repository-service)
  - [How to Efficiently Decode Events](#how-to-efficiently-decode-events)
    - [Case 1: Event Signature Matches a Single ABI](#case-1-event-signature-matches-a-single-abi)
    - [Case 2: Event Signature Matches Multiple ABIs](#case-2-event-signature-matches-multiple-abis)
  - [Data Store](#data-store)
    - [Table 1: topic_0 to ABI](#table-1-topic_0-to-abi)
    - [Table 2: Address to Event ABI](#table-2-address-to-event-abi)
    - [Table 3: Event ABI Lookup (Extension)](#table-3-event-abi-lookup-extension)
    - [Query 1: Fetch All ABIs for a topic_0](#query-1-fetch-all-abis-for-a-topic_0)
    - [Query 2: Fetch ABIs for a Set of Addresses](#query-2-fetch-abis-for-a-set-of-addresses)
    - [Why Postgres?](#why-postgres)
  - [Updating the ABI Repository](#updating-the-abi-repository)
    - [Factory Contract Listener](#factory-contract-listener)
    - [Fetching ABIs](#fetching-abis)
    - [Checkpointing](#checkpointing)

---

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
  "topics": [
    "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118",
    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"
  ],
  "addresses": [
    "0x1F98431c8aD98523631AE4a59f267346ea31F984",
    "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"
  ]
}
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `type` | string | Yes | Must be `"subscribe"` to initiate a subscription |
| `chain` | string | Yes | The blockchain network (e.g., `"ethereum"`, `"polygon"`, `"arbitrum"`) |
| `decode` | boolean | Yes | Whether to decode events (`true`) or return raw logs (`false`) |
| `topics` | array | Yes | Array of event signature hashes (topic_0) to monitor |
| `addresses` | array | No | Array of contract addresses to monitor. If omitted, listens to all contracts emitting the specified topics |

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

# Send subscription (specific contract)
> {
  "type": "subscribe",
  "chain": "ethereum",
  "decode": true,
  "topics": [
    "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
  ],
  "addresses": [
    "0x1F98431c8aD98523631AE4a59f267346ea31F984"
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

You can subscribe to multiple events and contracts in a single connection:

```json
{
  "type": "subscribe",
  "chain": "ethereum",
  "decode": true,
  "topics": [
    "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118",
    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"
  ],
  "addresses": [
    "0x1F98431c8aD98523631AE4a59f267346ea31F984",
    "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"
  ]
}
```

### Listen to All Contracts

Omit the `addresses` field to monitor events across all contracts:

```json
{
  "type": "subscribe",
  "chain": "ethereum",
  "decode": false,
  "topics": [
    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"
  ]
}
```

This will stream all ERC-20 Transfer events from any contract on Ethereum.

## Notes

- The WebSocket connection will stream events in real-time as they are confirmed on the blockchain
- Events from the same block are batched together in the response
- Ensure your event signature (topic_0) matches the keccak256 hash of the event signature (e.g., `PoolCreated(address,address,uint24,int24,address)`)
- For decoded events, the service must have the ABI for the contract to properly decode parameters
- When `addresses` is omitted, the service monitors all contracts for the specified topics (use with caution on high-volume events)
- The service matches events where topic_0 is in your `topics` list AND (if specified) the contract address is in your `addresses` list

---

# WebSocket Architecture Design

This section outlines the architectural approaches for the WebSocket event streaming service.

## Method: Shared Cache with Pub/Sub Broadcast

```mermaid
flowchart TB
    subgraph Clients["WebSocket Clients"]
        Client1[Client A<br/>Subscribe: Transfer events]
        Client2[Client B<br/>Subscribe: PoolCreated events]
        Client3[Client C<br/>Subscribe: Swap events]
    end

    subgraph WS["WebSocket Service"]
        Broadcast[Broadcast Loop<br/>For each client: Decode → Send]
        Listener[PubSub Listener]
    end

    subgraph BlockService["Block Listener Service"]
        PubSub{{Redis PubSub}}
        Redis[(Redis Cache<br/>Raw Block Data)]
        Poller[Block Poller]
        Proxy[RPC Proxy<br/>Multiplexing, Caching]
        RPC[(EVM RPC Node<br/>Ethereum / Polygon / Arbitrum)]
    end

    Client1 --- Broadcast
    Client2 --- Broadcast
    Client3 --- Broadcast

    Broadcast --- Listener
    Listener --- PubSub
    Listener --- Redis
    
    Redis --- Poller
    Poller --- Proxy
    Proxy --- RPC
```

### How It Works

- **Block Listener Service** continuously polls the RPC node for new blocks and writes raw log data to Redis cache
- **Redis Pub/Sub** emits a notification whenever new block data is cached
- **WebSocket Service** listens for these notifications, reads raw data from cache, then decodes and broadcasts to connected clients
- Each client receives only the events matching their subscription filters

### Why Shared Cache?

- **Client Reconnection**: If a client disconnects and reconnects, it can resume from its last processed block by reading older entries from the cache — no data gaps
- **Service Restart**: If the WebSocket service restarts, it continues from its last processed block by traversing the cache — no missed events
- **Decoupled Architecture**: Block polling and client delivery are independent; one can fail without affecting the other

---

# Webhook Service

Receive EVM events via HTTP POST requests to your server with optional decoding.

## Overview

The Webhook service sends blockchain events directly to your server via HTTP callbacks. Configure your endpoint once, and we'll automatically send matching events as they occur on-chain.

## Registering a Webhook

### Endpoint

```
POST https://api.yourdomain.com/v1/webhooks
```

### Request

```bash
curl -X POST https://api.yourdomain.com/v1/webhooks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "url": "https://your-server.com/webhook/events",
    "chain": "ethereum",
    "decode": true,
    "topics": [
      "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
    ],
    "addresses": [
      "0x1F98431c8aD98523631AE4a59f267346ea31F984"
    ]
  }'
```

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | Your server endpoint where events will be sent (must be HTTPS) |
| `chain` | string | Yes | The blockchain network (e.g., `"ethereum"`, `"polygon"`, `"arbitrum"`) |
| `decode` | boolean | Yes | Whether to decode events (`true`) or send raw logs (`false`) |
| `topics` | array | Yes | Array of event signature hashes (topic_0) to monitor |
| `addresses` | array | No | Array of contract addresses to monitor. If omitted, listens to all contracts emitting the specified topics |

### Response

```json
{
  "webhook_id": "wh_1a2b3c4d5e6f",
  "url": "https://your-server.com/webhook/events",
  "chain": "ethereum",
  "decode": true,
  "status": "active",
  "created_at": "2024-12-06T10:30:00Z",
  "topics": [
    "0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"
  ],
  "addresses": [
    "0x1F98431c8aD98523631AE4a59f267346ea31F984"
  ]
}
```

## Deregistering a Webhook

### Endpoint

```
DELETE https://api.yourdomain.com/v1/webhooks/{webhook_id}
```

### Request

```bash
curl -X DELETE https://api.yourdomain.com/v1/webhooks/wh_1a2b3c4d5e6f \
  -H "Authorization: Bearer YOUR_API_KEY"
```

### Response

```json
{
  "webhook_id": "wh_1a2b3c4d5e6f",
  "status": "deleted",
  "deleted_at": "2024-12-06T11:00:00Z"
}
```

## Webhook Payload

When events matching your filters occur, we'll send an HTTP POST request to your configured URL.

### Decoded Events Payload

When `decode: true`, events are sent with parsed parameters:

```json
{
  "webhook_id": "wh_1a2b3c4d5e6f",
  "block_number": 18500123,
  "chain": "ethereum",
  "timestamp": "2024-12-06T10:35:42Z",
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

### Raw Events Payload

When `decode: false`, events are sent as raw log objects:

```json
{
  "webhook_id": "wh_1a2b3c4d5e6f",
  "block_number": 18500123,
  "chain": "ethereum",
  "timestamp": "2024-12-06T10:35:42Z",
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

## Response Requirements

Your webhook endpoint must:
- Respond with a `2xx` status code within 5 seconds
- Accept `Content-Type: application/json`

## Example: Uniswap V3 Pool Creation

```bash
# Register webhook for decoded PoolCreated events
curl -X POST https://api.yourdomain.com/v1/webhooks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "url": "https://your-server.com/webhook/pools",
    "chain": "ethereum",
    "decode": true,
    "topics": ["0x783cca1c0412dd0d695e784568c96da2e9c22ff989357a2e8b1d9b2b4e6b7118"],
    "addresses": ["0x1F98431c8aD98523631AE4a59f267346ea31F984"]
  }'

# Your server receives:
{
  "webhook_id": "wh_1a2b3c4d5e6f",
  "block_number": 18500123,
  "chain": "ethereum",
  "timestamp": "2024-12-06T10:35:42Z",
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

---

# ABI Repository Service

Service that stores and retrieves event ABIs for decoding blockchain events.

## How to Efficiently Decode Events

For each chain, for each block, we receive raw logs from the RPC. We need to:

- **Filter relevant logs** — select only logs where `topic_0` matches event signatures provided by the client
- **Decode quickly** — expect 100-200 relevant events per block (typical case is < 5)

### Case 1: Event Signature Matches a Single ABI

Most common scenario:

- `topic_0` maps to exactly one event ABI
- Fetch the ABI directly from the repository
- Use it to decode all matching events across any contract

**Example:**
```
topic_0: 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
→ Transfer(address,address,uint256)
```

### Case 2: Event Signature Matches Multiple ABIs

When `topic_0` maps to multiple event ABIs:

- Cannot use `topic_0` alone to determine correct ABI
- Must disambiguate using the log's `address` field
- Each contract address maps to exactly one event ABI

**Why this happens:**
- Different contracts can emit events with same signature but different parameter names
- Indexed vs non-indexed parameters can differ

**Solution:**
- Look up the ABI by contract `address` instead of `topic_0`
- Must be fast — potentially 200 lookups per block

## Data Store

Postgres with 2 tables. Uses indices for fast lookups.

**Extension:** Add a 3rd table to integer-encode event ABIs for better storage efficiency (see [Table 3](#table-3-event-abi-lookup-extension)).

### Table 1: topic_0 to ABI

Maps event signatures to their ABIs.

```sql
CREATE TABLE topic_0_to_abi (
    topic_0 BYTEA NOT NULL,
    event_abi JSONB NOT NULL,
    PRIMARY KEY (topic_0, event_abi)
);

CREATE INDEX idx_topic_0 ON topic_0_to_abi (topic_0);
```

### Table 2: Address to Event ABI

Maps contract addresses to their event ABIs.

```sql
CREATE TABLE address_to_event_abi (
    address BYTEA NOT NULL,
    event_abi JSONB NOT NULL,
    PRIMARY KEY (address, event_abi)
);

CREATE INDEX idx_address ON address_to_event_abi (address);
```

### Table 3: Event ABI Lookup (Extension)

Optional table that stores ABIs once with an integer ID. Reduces storage when same ABI is referenced by many addresses.

```sql
CREATE TABLE event_abi_lookup (
    abi_id SERIAL PRIMARY KEY,
    event_abi JSONB NOT NULL UNIQUE
);
```

With this extension, Table 1 and Table 2 store `abi_id` (INTEGER) instead of `event_abi` (JSONB):

- **Before:** Each row duplicates the full JSONB ABI
- **After:** Each row stores a 4-byte integer reference

### Query 1: Fetch All ABIs for a topic_0

Used in Case 1 to check if signature maps to single or multiple ABIs.

```sql
SELECT event_abi 
FROM topic_0_to_abi 
WHERE topic_0 = $1;
```

### Query 2: Fetch ABIs for a Set of Addresses

Used in Case 2 when disambiguation is needed.

```sql
SELECT address, event_abi 
FROM address_to_event_abi 
WHERE address = ANY($1);
```

### Why Postgres?

- **Persistent** — not practical to keep everything in memory
- **Client-server** — access from anywhere + good tooling and UI support
- **Fast enough** — indices make lookups quick for our use case

## Updating the ABI Repository

### Factory Contract Listener

- Service listens to contract creation transactions from configured factory contracts
- API endpoint allows adding/removing factory contracts to monitor
- When a new contract is created by a factory, update the ABI repository

### Fetching ABIs

- If new contract differs from previously seen contracts, fetch ABI from Etherscan API
- Store the ABI mapping in both tables

### Checkpointing

- Checkpoint each processed block number
- WebSocket Service waits for ABI repository to catch up to a block before processing it
- Ensures ABIs are available before decoding events
