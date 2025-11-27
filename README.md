# BitBreakout Senior Backend Developer Coding Challenge (Node.js Edition)

**Time Limit:** 2 hours
**Tech Stack:** NestJS (TypeScript) + CCXT + Redis + WebSockets + Zod

### Goal:
Build a minimal backend that simulates a real-time crypto order book for BTC/USDT, fetching live data from Binance, storing it in Redis, and exposing it via API/WebSocket. Add a market order endpoint that "executes" against the book with slippage calculation.
This challenge tests your ability to handle async data fetching, Redis sorted sets for order books, real-time WebSocket deltas, and robust error handling in a Node.js environment. No real trading—just simulation!

---

### Why This Stack?
**NestJS:** Structured, scalable framework (like FastAPI but for Node/TS).
**CCXT:** Universal crypto exchange library (same as Python version).
**Redis:** For fast, sorted order book storage and pub/sub updates.
**WebSockets:** Via NestJS gateways for real-time pushes.
**Zod:** Schema validation for inputs (like Pydantic).

### Setup Instructions

#### 1. Clone and Install:
```bash
git clone <your-starter-repo-url> coding-challenge-node
cd coding-challenge-node
npm install
```

#### 2. Environment Setup:
```text
Create .env file in root:textREDIS_URL=redis://localhost:6379
BINANCE_API_KEY=your_binance_testnet_key  # Optional; CCXT public endpoints don't need it for order books
BINANCE_API_SECRET=your_binance_testnet_secret
```

Use Binance Testnet for any auth (though not needed here).

#### 3. Run Redis and App:
```bash
# Terminal 1: Start Redis (via Docker)
docker run -d -p 6379:6379 --name redis redis:alpine

# Terminal 2: Start the app
npm run start:dev  # Uses NestJS dev mode
```
App runs on `http://localhost:3000`. Redis persists bids/asks as sorted sets.

#### 4. Dependencies (already in package.json):
```text
"dependencies": {
  "@nestjs/core": "^10.0.0",
  "@nestjs/websockets": "^10.0.0",
  "@nestjs/platform-socket.io": "^10.0.0",
  "ccxt": "^4.0.0",
  "ioredis": "^5.0.0",
  "zod": "^3.22.0",
  "ws": "^8.0.0"  // Fallback for WebSockets
}
```

---

## Tasks
Implement the core features in the provided NestJS skeleton (src/app.module.ts, src/main.ts, etc.). Focus on clean, typed code with error handling.
### 1. Background Order Book Updater (Start on App Bootstrap)

- On app startup (e.g., in a NestJS service with @OnModuleInit()), fetch the BTC/USDT order book (depth: 20) from Binance every 2 seconds using CCXT.
- Store in Redis sorted sets:
  - **Bids** (buy orders): `ZADD bids -price quantity` (score = -price for descending sort, e.g., `ZADD bids -103200.0 1.5` for price 103200 with 1.5 BTC).
  - **Asks** (sell orders): `ZADD asks price quantity` (score = price for ascending sort).

- After each update, publish a JSON snapshot to Redis channel `depth:update` (e.g., `{ bids: [[price1, qty1]], asks: [[price1, qty1]] }` – top 10 levels only).
- Handle errors (e.g., network failures) with retries (up to 3) and logging.
- Hint: Use `setInterval` in an async service. CCXT example: `const exchange = new ccxt.binance(); const book = await exchange.fetchOrderBook('BTC/USDT', 20);`.


### 2. WebSocket Endpoint for Real-Time Depth (`/ws/depth`)

- Create a NestJS `@WebSocketGateway({ path: '/ws/depth' })` that uses Socket.IO.
- **On Connect:** Send current top 10 bids/asks as JSON (subscribe to Redis `depth:update` channel via `ioredis` pub/sub).
- **On Update* (Redis Pub/Sub): Push delta only for changed levels (e.g., new/updated price levels). Format:
```json{
  "bids": [[103200, 1.5]],  // Array of [price, quantity] for updates
  "asks": [[103210, 0.8]]
}
```

- Handle disconnects gracefully. Limit to 100 concurrent connections.
- Validation: Use Zod to parse incoming messages (if any, e.g., for subscriptions).


### 3. Market Order Execution Endpoint (`POST /market`)

- Validate input with Zod schema:
```typescript
const schema = z.object({
  side: z.enum(['buy', 'sell']),
  amount: z.number().positive(),  // e.g., 0.5 BTC
});
```

- For a **buy** market order: Walk the asks sorted set (ZRANGEBYSCORE ascending) to "fill" the amount.
  - Consume quantities level-by-level until filled.
  - Calculate: total filled amount, average fill price (weighted), slippage % (vs. best ask price).

- For **sell**: Walk bids (ZRANGEBYSCORE descending via negative scores).
- **Do not modify Redis**—simulate only (return hypothetical fills).
- Response (200 OK):
```json
{
  "filled": 6.5,
  "avg_price": 103225.3,
  "slippage_pct": 0.24,
  "status": "filled"
}
```

- Edge cases: Partial fills, insufficient depth (status: "partial"), validation errors (400).

---

## Testing Your Implementation

1. Start the App:`npm run start:dev` (Redis already running).
2. Test WebSocket (Real-Time Updates):
```bash
npm install -g wscat  # If needed
wscat -c ws://localhost:3000/ws/depth
```
- Expect initial snapshot, then deltas every ~2s. Send `{"subscribe": "BTCUSDT"}` if you add optional pings.

3. Test Market Order:
```bash
curl -X POST http://localhost:3000/market \
  -H "Content-Type: application/json" \
  -d '{"side": "buy", "amount": 6.5}'
```
- Expect JSON response with fills/slippage. Test "sell" too.

4. Verify Redis (Optional):
```bash
redis-cli
> ZRANGE bids 0 9 WITHSCORES  # Top 10 bids (reverse sort in code)
> ZRANGE asks 0 9 WITHSCORES  # Top 10 asks
```

---

## Submission Guidelines

- Push your code to a public GitHub repo (fork this starter or create new).
- Provide:
  - Repo link.
  - Live demo URL (e.g., via ngrok: ngrok http 3000 for WebSocket/curl tests).
  - Sample curl output for /market.
  - Screenshot of WebSocket connection receiving deltas.

- Bonus (if time): Add rate limiting (NestJS Throttler) or unit tests (Jest) for the market order logic.
- Clean up: Remove console.logs, handle async/await properly, use TypeScript everywhere.

**What We're Looking For:** Efficient Redis usage, real-time reliability, clean NestJS structure, and error-resilient code. Questions? Reply to the interview invite.
Good luck!
