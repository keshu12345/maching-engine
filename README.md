# Matching Engine High-Level Design for Solana

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         UI Layer                            │
│  (React/Next.js + Wallet Adapter + WebSocket Client)        │
└──────────────┬──────────────────────────────┬───────────────┘
               │                               │
               │ REST API                      │ WebSocket
               │                               │
┌──────────────▼───────────────────────────────▼───────────────┐
│                    API Gateway / Backend                     │
│         (Node.js/Rust - handles auth, routing)               │
└──────────────┬───────────────────────────────┬───────────────┘
               │                               │
               │ gRPC/HTTP                     │ Events
               │                               │
┌──────────────▼───────────────────────────────▼───────────────┐
│                  Off-Chain Matching Engine                   │
│  (Rust - high-performance order matching & state management) │
│                                                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐     │
│  │ Order Book  │  │ Match Engine │  │ Settlement Queue │     │
│  │  Manager    │  │   (Core)     │  │                  │     │
│  └─────────────┘  └──────────────┘  └──────────────────┘     │
└──────────────┬────────────────────────────────┬──────────────┘
               │                                │
               │ Settlement Instructions        │ State Sync
               │                                │
┌──────────────▼────────────────────────────────▼──────────────┐
│                  Solana On-Chain Programs                    │
│                                                              │
│  ┌──────────────────┐         ┌────────────────────┐         │
│  │ Orderbook Program│         │ Settlement Program │         │
│  │  (Anchor/Native) │◄────────┤   (Cranking)       │         │
│  └──────────────────┘         └────────────────────┘         │
│                                                              │
│  ┌──────────────────────────────────────────────────┐        │
│  │         Token Accounts & PDAs                    │        │
│  │  (User balances, escrow, market state)           │        │
│  └──────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

## 1. On-Chain Components (Solana Programs)

### 1.1 Core Smart Contracts

**Orderbook Program** (Anchor/Rust)
- **Accounts Structure:**
  - Market Account (stores trading pair config, fees)
  - User Order Accounts (open orders)
  - Escrow Accounts (locked tokens)
  - Event Queue (for off-chain indexing)

- **Instructions:**
  - `initialize_market`: Create new trading pair
  - `deposit_funds`: Lock tokens in escrow
  - `place_order`: Submit order on-chain (for full on-chain mode)
  - `cancel_order`: Cancel pending order
  - `withdraw_funds`: Return tokens to user wallet
  - `settle_match`: Execute matched trades

**Settlement Program**
- Processes batched settlements from off-chain engine
- Validates signatures and order authenticity
- Updates token balances atomically
- Emits settlement events

### 1.2 On-Chain Data Requirements

```rust
// Market Account
pub struct Market {
    pub authority: Pubkey,
    pub base_mint: Pubkey,
    pub quote_mint: Pubkey,
    pub base_vault: Pubkey,
    pub quote_vault: Pubkey,
    pub fee_bps: u16,
    pub min_order_size: u64,
    pub tick_size: u64,
}

// Order Account (for on-chain orders)
pub struct Order {
    pub owner: Pubkey,
    pub market: Pubkey,
    pub order_id: u64,
    pub side: Side, // Buy/Sell
    pub price: u64,
    pub quantity: u64,
    pub filled: u64,
    pub timestamp: i64,
}
```

## 2. Off-Chain Matching Engine

### 2.1 Core Components (Rust Implementation)

**Order Book Manager**
- Maintains sorted buy/sell order books in memory
- Red-Black Tree or B-Tree for price levels
- Linked lists for orders at same price (FIFO)
- Fast O(log n) insertion, deletion, matching

**Matching Engine Core**
- Price-time priority algorithm
- Processes orders sequentially
- Generates fill events
- Supports order types: Market, Limit, IOC, FOK, Post-Only

**Settlement Queue**
- Batches matched trades
- Prepares Solana transaction bundles
- Retry logic for failed settlements
- Transaction optimization (combines multiple fills)

### 2.2 State Management

**In-Memory State**
- Redis or in-process memory for active orders
- PostgreSQL/TimescaleDB for historical data
- Message queue (Kafka/RabbitMQ) for event streaming

**State Synchronization**
- Listen to on-chain events via WebSocket/gRPC
- Reconcile off-chain state with on-chain truth
- Handle reorg scenarios (though rare on Solana)

## 3. Backend API Layer

### 3.1 REST API Endpoints

```
POST   /api/v1/orders              # Submit new order
DELETE /api/v1/orders/:id          # Cancel order
GET    /api/v1/orders              # Get user orders
GET    /api/v1/orderbook/:market   # Get current orderbook
GET    /api/v1/trades              # Get trade history
GET    /api/v1/markets             # Get available markets
```

### 3.2 WebSocket Feeds

```javascript
// Real-time data streams
ws://api.exchange.com/ws

// Channels:
- orderbook.{market}     // L2 orderbook updates
- trades.{market}        // Recent trades
- orders.{user}          // User's order updates
- fills.{user}           // User's fill notifications
```

## 4. UI Integration Setup

### 4.1 Frontend Dependencies

```json
{
  "dependencies": {
    "@solana/web3.js": "^1.95.0",
    "@solana/wallet-adapter-react": "^0.15.35",
    "@solana/wallet-adapter-wallets": "^0.19.32",
    "@project-serum/anchor": "^0.30.0",
    "axios": "^1.6.0",
    "socket.io-client": "^4.7.0",
    "react": "^18.2.0",
    "recharts": "^2.10.0"
  }
}
```

### 4.2 Wallet Connection

```typescript
// Wallet setup
import { WalletAdapterNetwork } from '@solana/wallet-adapter-base';
import { ConnectionProvider, WalletProvider } from '@solana/wallet-adapter-react';
import { PhantomWalletAdapter } from '@solana/wallet-adapter-wallets';

const network = WalletAdapterNetwork.Mainnet;
const endpoint = 'https://api.mainnet-beta.solana.com';
const wallets = [new PhantomWalletAdapter()];
```

### 4.3 Communication Patterns

**Order Submission Flow:**
```
1. UI: User creates order → Signs with wallet
2. UI → Backend: POST /api/v1/orders + signature
3. Backend → Matching Engine: Process order
4. Matching Engine: Match + emit events
5. Backend → Solana: Submit settlement tx
6. Backend → UI: WebSocket order status update
7. Solana → Backend: Confirmation event
8. Backend → UI: Final confirmation
```

**Real-time Updates:**
```typescript
// WebSocket client
const socket = io('wss://api.exchange.com');

socket.on('connect', () => {
  socket.emit('subscribe', { 
    channels: ['orderbook.SOL-USDC', 'orders.user123'] 
  });
});

socket.on('orderbook', (data) => {
  // Update orderbook UI
  updateOrderBook(data);
});

socket.on('fill', (data) => {
  // Show notification
  showFillNotification(data);
});
```

## 5. Infrastructure Requirements

### 5.1 On-Chain Infrastructure

- **RPC Nodes**: 
  - Private RPC endpoints (Triton, Helius, QuickNode)
  - Multiple nodes for redundancy
  - WebSocket support for event listening

- **Keypairs**:
  - Program deployer keypair
  - Market authority keypair
  - Settlement cranker keypair (funded for tx fees)

### 5.2 Off-Chain Infrastructure

**Compute:**
- Matching engine servers (high CPU, low latency)
- API servers (load balanced)
- WebSocket servers (sticky sessions)

**Storage:**
- Redis Cluster (order book cache)
- PostgreSQL (user data, order history)
- TimescaleDB (time-series trade data)
- S3/IPFS (backups, archives)

**Message Queue:**
- Kafka/RabbitMQ (event streaming)
- Partition by market for scaling

### 5.3 Monitoring & DevOps

- Prometheus + Grafana (metrics)
- ELK Stack (logging)
- Sentry (error tracking)
- PagerDuty (alerting)

## 6. Security Considerations

**On-Chain:**
- Multi-sig program authority
- Admin key rotation
- Emergency pause functionality
- Audit by security firms

**Off-Chain:**
- API rate limiting
- DDoS protection
- Order signature verification
- Database encryption
- Secure key management (HSM/KMS)

**User Protection:**
- Client-side order validation
- Nonce/timestamp to prevent replay
- Max order size limits
- Circuit breakers for extreme volatility

## 7. Hybrid Approach (Recommended)

**Fast Trading Path (Off-Chain):**
- User signs order with wallet
- Off-chain engine matches instantly
- Batched settlement to Solana every N seconds
- 10-100ms latency for matches

**Secure Settlement (On-Chain):**
- Periodic on-chain settlement (every 1-5 seconds)
- Final balances always on-chain
- Users can force settlement anytime
- Emergency mode: full on-chain fallback

This design provides high throughput off-chain while maintaining Solana's security guarantees through regular on-chain settlement.