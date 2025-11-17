# ds-g1-sms

---

## Architecture Overview

## Core Architecture: Hybrid P2P with Distributed Hash Table (DHT)

### System Components

1. **Node Types** (all nodes are equal peers)
   - Each node runs the full chat service stack
   - Each node maintains a subset of the global state
   - Minimum 3 nodes for consensus/fault tolerance

2. **State Management Layer**
   - **Distributed Hash Table (DHT)**: For data partitioning and lookup (e.g., Kademlia-based)
   - **Replicated State Machines**: Each partition replicated across multiple nodes
   - **Consistent Hashing**: For even data distribution and minimal reshuffling

3. **Consensus Layer**
   - **Raft Consensus Protocol**: For strongly consistent operations
     - Leader election per data partition
     - Log replication for state changes
     - Handles: room creation, membership changes, role assignments
   
4. **Eventual Consistency Layer**
   - **Conflict-free Replicated Data Types (CRDTs)**: For messages and presence
     - Messages use timestamp-based ordering
     - Presence status uses Last-Write-Wins (LWW)
     - Allows partition tolerance with eventual convergence

## Data Distribution Strategy

### Partitioning Scheme

```
Data Partitioning by Entity:
├── Users: Hash(user_id) → Node
├── Chat Rooms: Hash(room_id) → Node  
├── Memberships: Hash(room_id) → Node (co-located with room)
└── Messages: Hash(room_id) → Node (co-located with room)
```

**Replication Factor**: 3 (each data partition stored on 3 nodes)
- Primary replica (leader): Handles writes via Raft
- Secondary replicas (followers): Handle reads, participate in consensus

## Addressing Your Requirements

### a) Stateful with Global State

**Solution**: Distributed database with multi-master replication

- **CockroachDB**: Distributed SQL with sharding
- Each node maintains portion of global state
- Global state reconstructed via DHT queries across nodes
- Metadata registry: Each node maintains routing table of data locations

**State Components**:
- User registry (distributed)
- Room registry (distributed)
- Membership mappings (co-located with rooms)
- Message logs (partitioned by room)

### b) Data Consistency and Synchronization

**Multi-level consistency model**:

**Strong Consistency** (using Raft):
- User account creation/deletion
- Room creation/deletion
- Membership additions/removals
- Role assignments (admin/member)

**Eventual Consistency** (using CRDTs):
- Message delivery
- User presence/status updates
- Typing indicators
- Last read timestamps

**Synchronization Protocol**:
```
1. Write Operation:
   - Client → Any Node (coordinator)
   - Coordinator determines partition via consistent hashing
   - Coordinator forwards to partition leader
   - Leader proposes via Raft consensus
   - Quorum acknowledgment (2 of 3 replicas)
   - Async replication to remaining nodes

2. Read Operation:
   - Read from any replica (with potential stale reads)
   - OR read from leader (strongly consistent)
   - Configurable per operation
```

**Conflict Resolution**:
- Vector clocks for causality tracking
- Application-level merge for messages (append-only log)
- Last-write-wins for user metadata with timestamps

### c) Consensus (Shared Decision)

**Raft Consensus for Critical Operations**:

**Per-Partition Raft Groups**:
- Each data partition has its own Raft group (3 nodes)
- One leader per partition handles writes
- Followers replicate log entries
- Leader election on failure (typically <1 second)

**Consensus Required For**:
1. Room creation → All replicas agree on room_We can go ahead with direct node connections. We don't need to implement fault tolerance and scalability right now. We'll do CockroachDB for the DB and Go for the system programming and etcd for consensus.id and creator
2. Adding/removing members → Prevents duplicate adds
3. Role changes → Ensures only one admin assignment at a time
4. Room deletion → Coordinated cleanup across nodes

**Consensus Algorithm**:
```
1. Client request → Coordinator node
2. Coordinator → Raft leader for partition
3. Leader appends to log, sends to followers
4. Followers acknowledge
5. Leader commits when quorum reached (2/3)
6. Leader responds to coordinator
7. Coordinator responds to client
```

**Optimization**: Leader leases (reduce leader checks)

### d) Fault Tolerance

**Node Failure Handling**:

**Single Node Failure** (1 of 3):
- Raft continues with 2/3 majority
- Reads/writes proceed normally
- System operates at reduced redundancy

**Two Node Failure** (2 of 3):
- Affected partitions enter read-only mode
- No consensus possible for writes
- Existing data remains readable
- System degraded but operational

**Network Partition** (Split Brain):
- Raft prevents split-brain via majority quorum
- Minority partition becomes read-only
- Majority partition continues operating
- Automatic reconciliation on partition heal

**Fault Tolerance Strategies**:
1. **Replication**: 3x replication factor
2. **Health Checks**: Heartbeat every 50-150ms
3. **Failure Detection**: Timeout-based (typically 150-300ms)
4. **Automatic Failover**: New leader election <1s
5. **Data Recovery**: Failed nodes catch up via log replay
6. **Checksums**: Detect data corruption

**Backup Strategy**:
- Periodic snapshots (every N log entries)
- Write-ahead logging (WAL)
- Incremental backups to external storage

### e) Scalability

**Horizontal Scaling**:

**Adding New Nodes**:
```
1. New node joins DHT network
2. Announces capacity to existing nodes
3. Consistent hashing recalculates partitions
4. Data migration begins (typically 1/N of data where N = nodes)
5. Replication factor maintained
6. Gradual traffic shift to new node
```

**Scaling Dimensions**:

**Storage Scalability**:
- More nodes = more total storage
- Consistent hashing ensures O(log N) lookup
- Each node stores ~1/N of total data

**Throughput Scalability**:
- Read scaling: Queries distributed across replicas
- Write scaling: Each partition independent
- Message delivery: Parallel processing per room

**User Scalability**:
- DHT lookup: O(log N) complexity
- Room discovery: Distributed indices
- Connection handling: Each node handles subset of users

**Bottleneck Mitigation**:
1. **Hot Partitions**: Automatically split popular rooms
2. **Read Heavy**: Add read replicas (>3 replicas)
3. **Write Heavy**: Shard rooms into sub-rooms
4. **Large Rooms**: Message pagination, lazy loading

**Performance Targets** (example):
- 10,000+ concurrent users per 3-node cluster
- 100,000+ messages/second aggregate
- <100ms p99 message delivery latency
- Linear scaling to 10+ nodes

## Communication Protocols

### Inter-Node Communication
- **gRPC** or **WebSockets**: Low-latency RPC
- **Protocol Buffers**: Efficient serialization
- **TLS**: Encrypted node-to-node communication

### Client-Node Communication
- **WebSockets**: Real-time bidirectional messaging
- **HTTP/2**: REST API for control operations
- **Load balancing**: Client connects to any node (DNS round-robin or client-side)

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         Client Layer                        │
│  (Mobile Apps, Web Clients) - Connect to any node           │
└─────────────────┬───────────────────────────────────────────┘
                  │ WebSocket/HTTP
┌─────────────────┴───────────────────────────────────────────┐
│                    Node 1 (Peer)                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ API Gateway (WebSocket + REST)                       │   │
│  └──────────────┬───────────────────────────────────────┘   │
│  ┌──────────────┴───────────────────────────────────────┐   │
│  │ Business Logic Layer                                 │   │
│  │  - Room Management  - User Management                │   │
│  │  - Message Handling - Authorization                  │   │
│  └──────────────┬───────────────────────────────────────┘   │
│  ┌──────────────┴───────────────────────────────────────┐   │
│  │ Consistency Layer                                    │   │
│  │  - Raft Consensus Module (Leader/Follower)           │   │
│  │  - CRDT Merge Engine                                 │   │
│  └──────────────┬───────────────────────────────────────┘   │
│  ┌──────────────┴───────────────────────────────────────┐   │
│  │ Distributed State Layer                              │   │
│  │  - DHT Client (Kademlia)                             │   │
│  │  - Partition Manager                                 │   │
│  │  - Vector Clock Tracker                              │   │
│  └──────────────┬───────────────────────────────────────┘   │
│  ┌──────────────┴───────────────────────────────────────┐   │
│  │ Storage Layer                                        │   │
│  │  - PostgreSQL/CockroachDB (partitioned data)         │   │
│  │  - Write-Ahead Log (WAL)                             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────┬───────────────────────────────────────────┘
                  │ gRPC/Gossip Protocol
         ┌────────┴────────┬────────────────┐
         │                 │                │
    ┌────┴─────┐      ┌────┴─────┐     ┌────┴─────┐
    │  Node 2  │←────→│  Node 3  │←───→│  Node 1  │
    │  (Peer)  │      │  (Peer)  │     │  (Peer)  │
    └──────────┘      └──────────┘     └──────────┘
    (Same internal structure as Node 1)
```

## Deployment Recommendations

### Minimum Viable Deployment
- **3 nodes**: Minimum for fault tolerance and consensus
- **Each node**: 4 CPU cores, 8GB RAM, 100GB SSD
- **Network**: Low-latency (<10ms between nodes)
- **Geographic**: Single datacenter initially

### Production Deployment
- **5-7 nodes**: Better fault tolerance
- **Multi-region**: Nodes across 2-3 regions
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK stack or similar
- **Alerting**: PagerDuty for consensus failures

## Trade-offs and Considerations

**Advantages**:
- ✅ No single point of failure
- ✅ Scales horizontally
- ✅ Strong consistency where needed
- ✅ High availability (survives 1 node failure)

**Challenges**:
- ⚠️ Complex implementation (Raft + DHT + CRDTs)
- ⚠️ Network partitions require careful handling
- ⚠️ Eventual consistency for messages (potential reordering)
- ⚠️ Higher latency than centralized (consensus overhead)

**Tech Stack**:
- Language: Java (systems programming, concurrency)
- Database: CockroachDB (built-in distribution)
- Consensus: etcd
- DHT: libp2p (IPFS networking stack)
- Message Queue: Apache Kafka (for message streaming)

<!--

**Here are some ideas to get you started:**

🙋‍♀️ A short introduction - what is your organization all about?
🌈 Contribution guidelines - how can the community get involved?
👩‍💻 Useful resources - where can the community find your docs? Is there anything else the community should know?
🍿 Fun facts - what does your team eat for breakfast?
🧙 Remember, you can do mighty things with the power of [Markdown](https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
-->
