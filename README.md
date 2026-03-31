# MIVA System Overview

## 1. Overview
MIVA is a real-time, highly scalable matchmaking and social connection platform. It uses a decoupled, event-driven microservices architecture to ingest users, find compatible matches based on specific criteria (age, gender, country, mood, etc.), and establish mutually agreed real-time sessions. The core user flow involves authenticating via HTTP, opening a persistent WebSocket connection, specifying matching criteria, and entering a matchmaking pool. Behind the scenes, the system batches, processes, pairs, and delivers results with strict latency and scalability considerations.

## 2. System Architecture
The application is structured as a monorepo containing multiple separate deployable apps and shared packages. It heavily leverages **Redis** (Hashes, Streams, Pub/Sub) and **BullMQ** as the backbone for inter-service communication and transient state management.

### Key Components:
- **`http` Application**: A stateless Express server built on top of MongoDB. It manages REST APIs, user authentication (`/auth`), and user profile updates (`/me`, `/user`).
- **`socket` Application**: A stateful Socket.io server. It manages the real-time lifecycle of users, handles incoming client events (`join_pool`, `skip_match`), buffers users into the matchmaking pipeline, and emits final match results back to clients. It also hosts the `resultsConsumer` daemon to listen to system outputs.
- **`batcher` Application**: A lightweight stream consumer that continuously pulls users from the ingest stream (`stream:buffer`). Its sole purpose is to aggregate individual users into efficiently sized batches and push them into the background job queue for processing.
- **`workers` Application**: A fleet of BullMQ worker instances. They pull large batches of users from the match queue, execute the heavy compute matching algorithms, and emit the results back into a dedicated Redis Stream (`stream:results`).

## 3. Core Flow
### 1. Ingestion
1. A user authenticates via REST API and establishes a WebSocket connection with the `socket` app.
2. The client emits a `join_pool` event containing their `MatchFilter` (preferences for age, gender, country, and mood).
3. The `socket` server fetches the user's permanent profile from MongoDB (just once), constructs an enriched view, and caches it in a Redis Hash (`pool:{userId}`).
4. The user's ID is appended to a Redis Stream (`stream:buffer`) with a retry count `R=0`.

### 2. Batching
1. The `batcher` app endlessly polls `stream:buffer` using a Redis Consumer Group.
2. Once a sufficient number of users are buffered (or a time-based flush interval elapses), the `batcher` creates a batch job and pushes it into a BullMQ queue (`match`).

### 3. Processing (Workers)
1. A worker from the `workers` app picks up the job.
2. It hydrates user objects by pipelining `HGETALL` against the cached `pool:{userId}` Redis Hashes.
3. The worker runs the `optimizedMega` algorithm to find all valid, mutual pairs within the batch.
4. Matched pairs are pushed to the `stream:results` Redis stream. Unmatched users have their retry count (`R`) incremented and are either re-queued to `stream:buffer` or aggressively pushed to `stream:results` as `noMatch` if they exceed the max retry threshold.

### 4. Result Delivery
1. The `resultsConsumer` loop (running on `socket` nodes) processes `stream:results`. It hydrates the full profile of the matched pair to drop dependencies on further profile loads.
2. It publishes the localized match payload to a **Redis Pub/Sub** channel (`match` or `noMatch`). 
3. The `resultsEmitter` (also on `socket` nodes) subscribes to these Pub/Sub channels. Upon receiving a match event, it initializes a transient session key (`session:{matchId}`) in Redis, marking both users as unaccepted. 
4. The `socket` node uses `io.to(socketId).emit()` to alert the individual clients.

## 4. Matching / Processing Engine
The matching engine is designed to execute heavy combinatorial logic without blowing up runtime complexity—operating internally at nearly **O(N)** complexity rather than O(N*M).

- **Indexing Strategy**: Instead of array iteration, `optimizedMega` restructures the flat batch of users into dense, nested dictionaries: `queues[gender][country][age]`.
- **Lookups**: To find a partner, a user strictly looks up the specific allowed paths in the index (e.g., specific target country, specific target age range).
- **Fast Skipping**: It maintains a `used` Set. As soon as a user forms a pair, their IDs are added to `used` and are skipped in O(1) time for all subsequent checks.
- **Mutual Verification**: The worker ensures both users explicitly align on gender, age, and country constraints before declaring a pairing successful.

## 5. Real-Time Layer
- Websockets are secured using an authentication middleware to block untrusted connections before upgrade.
- **Disconnection Handling**: On disconnect `user_leave`, the `socket` server reliably deletes the state in the `pool:{userId}` Redis Hash to passively prune them out of system resources, aborting their queued matching operations upstream via the worker hydration step gracefully failing.
- Clients interface purely via simple event names: `join_pool`, `skip_match`, `retry_match_again`.

## 6. Scaling Strategy
The architecture is inherently horizontally scalable:
- **Socket Sharding**: The system mitigates "parallel universe" fragmentation on ingestion by allowing future deterministic sharding on the `stream:buffer` so multiple batchers can read partitioned data streams (currently round-robining shards).
- **Stateless HTTP**: The backend REST APIs can scale infinitely.
- **Worker Scaling**: The heavy lifting lives fundamentally on BullMQ. Node instances or Lambdas running the matcher logic can autoscale based on active jobs in the queue. 
- **Decoupled Delivery**: Real-time events use Pub/Sub dynamically routed to whatever WebSocket server holds a particular socket. If a `socket` node dies, users seamlessly reconnect and end up on a new node without losing state.

## 7. Data & State Management
- **MongoDB**: The source of truth. Stores long-term user data, profiles, constraints, and audit trails.
- **Redis Hashes**: Utilized as a high-speed proxy/cache representing the transient state of connected users (`pool:*`) & session negotiation (`session:*`). Keys automatically expire to avoid garbage pile-up.
- **Redis Streams**: Acts as an append-only transaction log buffering ingestion and processing outputs (`stream:buffer`, `stream:results`).
- **Redis Pub/Sub**: Strictly low-latency horizontal routing mechanism for node-to-node signal passing.

## 8. Reliability & Fault Tolerance
- **Consumer Group Tracking**: The `batcher` and `resultsConsumer` both leverage Redis Consumer Groups. This ensures messages are never lost if a node crashes mid-batch.
- **Self-Healing Recovery**: Built-in periodic garbage collection (`recoverPending` polling using `XAUTOCLAIM`) scoops up messages left stranded by abruptly terminated processes.
- **Idempotency**: Pipelined acknowledgments (`XACK` + `XDEL`) happen cleanly at the very end of stream processes. 
- **Retry Mechanisms**: Users who don’t find an immediate pair are transparently rolled over to subsequent matchmaking runs automatically (`R+1`) until giving up gracefully.

## 9. Key Design Decisions
1. **Separation of Streams and Pub/Sub**: The decision to separate worker output arrays into `stream:results` while subsequently pushing mapped items to Pub/Sub is brilliant. It ensures robust, at-least-once message processing initially, followed by fan-out delivery routing that handles scaling across countless WebSocket edge nodes.
2. **Flattened Indexing**: The `optimizedMega` dictionary indexing sacrifices slight initial memory chunking to achieve incredibly fast lookups across filtered dimensions, trading memory explicitly for low latency.
3. **Database Caching on Join**: Hitting MongoDB on WebSocket connect rather than within the matcher process protects the DB from the volatile throughput of the matcher loop.



