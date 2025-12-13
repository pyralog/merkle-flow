# MerkleFlow

## Modern Gossip Protocol with Merkle Search Tree (MST) Anti-Entropy

**MerkleFlow** is a production-ready, modern gossip subsystem for large-scale, low-latency, and highly available state dissemination. It combines:

- Membership and failure detection: SWIM with Lifeguard refinements
- Overlay management and broadcast: HyParView + Plumtree (epidemic + spanning tree)
- State dissemination: push gossip for hot deltas; pull-based anti-entropy with Merkle Search Tree (MST)
- Transport: WireGuard userspace (BoringTun) for encrypted, authenticated peer-to-peer tunnels
- State model: CRDT-first (LWW-Register, OR-Set, PN/G counters) with tombstones and vector clocks
- Production features: backpressure, rate-limits, shard-aware replication, WAL and snapshots, metrics, and tracing


### Goals

- Robust under churn, partitions, and asymmetric links
- Sub-linear bandwidth growth via selective fanout and anti-entropy
- Fast convergence for hot keys; bounded staleness for cold keys
- Observable and tunable with clear SLOs
- Secure by default: authenticated peers, integrity, replay protection
- Portable: clean Rust APIs, pluggable transport/storage/crypto


### Non-Goals

- Byzantine-fault tolerance (focus is on benign faults and churn)
- Global total order; we prefer CRDT convergence and per-key/namespace causal consistency


## Use Cases

This gossip protocol is designed for scenarios requiring eventually-consistent, highly-available state replication across distributed systems:

### Service Discovery and Health Monitoring

- Propagate service endpoints, health status, and capabilities across clusters
- Detect and disseminate node failures with sub-second convergence
- Maintain consistent views of available services for client-side load balancing
- Example: Microservices mesh where services discover and monitor each other without a central registry

### Distributed Configuration Management

- Replicate feature flags, A/B test configurations, and runtime parameters
- Push critical config changes with low latency; pull-sync ensures eventual consistency
- Support gradual rollouts and canary deployments with namespace isolation
- Example: Feature flag service serving thousands of application instances across regions

### Session State Replication

- Share user session data across web servers for stateful failover
- LWW-Register for simple session attributes; OR-Set for shopping carts
- Sub-second replication within a datacenter; bounded staleness across regions
- Example: E-commerce platform with sticky sessions but graceful failover on node crashes

### Distributed Caching and Materialized Views

- Replicate hot cache entries and query results across cache nodes
- Invalidations propagate via tombstones; anti-entropy repairs missed updates
- Namespace per tenant or cache tier for controlled fanout
- Example: CDN edge nodes sharing popularity rankings and cached content metadata

### Real-Time Collaboration and Presence

- Track online users, cursors, and activity across application servers
- OR-Set for presence lists; G-Counter for unread counts
- Push gossip for low-latency updates; periodic sync for consistency
- Example: Chat application where user presence updates within 100-300ms

### Edge Computing and IoT Coordination

- Sync device states, sensor readings, and control commands at the edge
- Handles intermittent connectivity and network partitions gracefully
- Lightweight anti-entropy suitable for resource-constrained environments
- Example: Smart home hub coordinating with cloud backend; works during internet outages

### Multi-Region Metadata Replication

- Distribute cluster metadata, shard assignments, and routing tables
- Geographic namespaces reduce cross-region traffic; anti-entropy ensures convergence
- Causal consistency via vector clocks prevents stale overwrites
- Example: Global key-value store replicating partition maps and tenant configurations

### Leaderless Key-Value Stores

- Build Dynamo-style databases with tunable consistency and availability
- CRDT values eliminate read-repair complexity; MST anti-entropy replaces merkle trees
- Decentralized replication without coordinator bottlenecks
- Example: Distributed cache or database prioritizing availability over strong consistency

### Operational Metrics and Observability

- Aggregate and disseminate cluster-wide metrics, alerts, and system states
- PN-Counter for distributed counters; LWW for latest metric snapshots
- Dashboard queries hit local replica; anti-entropy keeps data fresh
- Example: Monitoring system collecting and sharing health metrics across data centers

### Multi-Tenant SaaS Platforms

- Isolate tenant data via namespace prefixes; shard-aware replication reduces overhead
- Per-tenant rate limits and access control policies replicated via gossip
- Efficient bandwidth usage with selective fanout and interest-based sync
- Example: Platform serving millions of tenants with isolated, eventually-consistent state per tenant


## Architecture Overview

At a high level each node runs the following subsystems:

1) Membership (SWIM+Lifeguard): detects liveness and propagates member updates via gossip.
2) Overlay (HyParView): maintains small active view for eager push and a passive view for resilience.
3) Broadcast (Plumtree): routes messages via eager push; shifts to a repair tree for efficiency.
4) State Replication: push deltas for hot keys; periodic pull anti-entropy using MST range proofs.
5) Transport: WireGuard tunnels (BoringTun userspace) for encrypted, authenticated peer communication.
6) Storage: WAL + snapshot + MST index; CRDT value store with tombstones and compaction.
7) Security: WireGuard's Noise protocol (Curve25519, ChaCha20-Poly1305) for tunnel encryption; Ed25519 for node identity.
8) Observability: metrics, logs, tracing; simulation harness for fault injection.

```
┌────────────────────────────────────────────────────────────────┐
│                      MerkleFlow Node                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │  Membership  │  │   Overlay    │  │  Broadcast   │        │
│  │ SWIM+Lifeguard│──│  HyParView   │──│  Plumtree    │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
│         │                  │                  │                │
│         └──────────────────┼──────────────────┘                │
│                            │                                   │
│         ┌──────────────────┴──────────────────┐               │
│         │    State Replication Engine         │               │
│         │  ┌────────────┐   ┌──────────────┐ │               │
│         │  │Push Gossip │   │Anti-Entropy  │ │               │
│         │  │ (hot keys) │   │MST-based pull│ │               │
│         │  └─────┬──────┘   └──────┬───────┘ │               │
│         └────────┼─────────────────┼─────────┘               │
│                  │                 │                           │
│         ┌────────┴─────────────────┴─────────┐               │
│         │         CRDT Store                  │               │
│         │  ┌──────────┐  ┌──────────────┐    │               │
│         │  │   MST    │  │Value Store + │    │               │
│         │  │  Index   │  │  Tombstones  │    │               │
│         │  └────┬─────┘  └──────┬───────┘    │               │
│         └───────┼────────────────┼────────────┘               │
│                 │                │                             │
│         ┌───────┴────────────────┴────────────┐               │
│         │    Persistence Layer                │               │
│         │  ┌──────┐  ┌──────────┐            │               │
│         │  │ WAL  │  │Snapshots │            │               │
│         │  └──────┘  └──────────┘            │               │
│         └────────────────────────────────────┘               │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│               Transport Layer (WireGuard/BoringTun)            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐              │
│  │  Tunnels   │  │ UDP Socket │  │  Crypto    │              │
│  │ (Peer-to-  │  │  (Gossip   │  │(Noise IK,  │              │
│  │   Peer)    │  │  Messages) │  │ ChaCha20)  │              │
│  └────────────┘  └────────────┘  └────────────┘              │
└────────────────────────────────────────────────────────────────┘
```

### How the Layers Work Together

SWIM, HyParView, and Plumtree form a layered architecture where each component builds on the layer below:

#### SWIM (Membership Layer)

- Detects which nodes are alive or dead in the cluster
- Provides a membership list to the layers above
- Gossips membership changes (joins, leaves, failures)

#### HyParView (Overlay Layer)

- Uses SWIM's membership list to maintain connections
- Manages the network topology: which nodes should connect to which
- Maintains two views:
  - **Active view**: Small set of direct neighbors (5-8 peers) for actual communication
  - **Passive view**: Larger backup set (30-60 peers) for recovery when active peers fail
- When SWIM reports a node as dead, HyParView removes it from views and promotes a passive peer to active

#### Plumtree (Broadcast Layer)

- Uses HyParView's active view connections to disseminate messages
- Implements efficient broadcast via:
  - **Eager push**: Forward full messages to some peers (forming a tree)
  - **Lazy push**: Send only message IDs to other peers (reducing redundancy)
- Self-optimizes into a spanning tree structure over HyParView's overlay

#### Relationship Flow

```text
SWIM → provides alive/dead info → HyParView → maintains connections → Plumtree
(Who exists?)                     (Who to talk to?)                  (How to broadcast?)
```

They form a stack where each layer depends on the one below it, creating a robust gossip system that combines failure detection, self-organizing topology, and efficient broadcast.


## Membership and Failure Detection (SWIM + Lifeguard)

- Protocol: Periodic randomized probing (Ping/Ack), with indirect pings via k relays on timeout.
- Suspicion: Suspect before Confirm to reduce false positives; suspicion time scales with local RTT.
- Local health awareness (Lifeguard): adjust probe intervals and suspicion timers based on recent local failures to avoid cascading false positives under overload.
- Dissemination: Membership changes (Join, Alive, Suspect, Confirm, Leave) are gossiped via broadcast layer.
- Incarnation numbers: Monotonic per-node counters to resolve conflicts between Alive and Suspect/Confirm.
- Seed bootstrap: Nodes join via one or more seeds; seeds respond with current member list and epochs.

```
Direct Ping (Happy Path):
┌────────┐                              ┌────────┐
│ Node A │                              │ Node B │
└───┬────┘                              └───┬────┘
    │                                       │
    │──────────── Ping(nonce) ──────────>  │
    │                                       │
    │<──────────── Ack(nonce) ───────────  │
    │                                       │
    │  B is alive ✓                         │
    

Indirect Ping (Timeout Recovery):
┌────────┐           ┌────────┐           ┌────────┐
│ Node A │           │ Node K │           │ Node B │
└───┬────┘           └───┬────┘           └───┬────┘
    │                    │                    │
    │─ Ping(nonce) ──────────────────────X   │ (timeout)
    │                    │                    │
    │─ IndirectPing ──>  │                    │
    │  (target=B)        │                    │
    │                    │── Ping(nonce) ──>  │
    │                    │                    │
    │                    │<─── Ack(nonce) ─── │
    │                    │                    │
    │<─── Ack via K ───  │                    │
    │                    │                    │
    │  B is alive ✓                           │
    

State Transitions:
                                  timeout
    ┌───────┐   ping fail   ┌──────────┐  ───────────>  ┌─────────┐
    │ Alive │ ────────────> │ Suspect  │                │ Confirm │
    └───┬───┘               └────┬─────┘  <───────────  │ (dead)  │
        │                        │       k indirect      └─────────┘
        │                        │       all fail             
        │  refutation (higher    │
        └<───────incarnation)────┘
```

Config knobs:

- probeIntervalMs, probeTimeoutMs, indirectK
- suspicionTimeoutMs base and scaling factors
- lifeguardLocalHealthScore thresholds


## Overlay and Broadcast (HyParView + Plumtree)

- Active view: small, constant-size neighbor set for eager push; try to keep degree near targetFanout.
- Passive view: larger set of backup peers used for recovery and shuffling.
- Join: Active random walk + accept rules maintain connected overlay under churn.
- Broadcast (Plumtree):
  - Eager push: flood new messages with small TTL; track received IDs to suppress duplicates.
  - Lazy push: piggyback message IDs; peers request missing payloads selectively.
  - Repair/Tree: Transition to a spanning tree for repeated messages to reduce redundancy.

```
HyParView Overlay Structure:

         Node A's View:
         
         Active View (size=4-6):
         ┌─────────────────────────────────┐
         │   Eager push neighbors          │
         │   ┌───┐  ┌───┐  ┌───┐  ┌───┐  │
         │   │ B │  │ C │  │ D │  │ E │  │
         │   └───┘  └───┘  └───┘  └───┘  │
         └─────────────────────────────────┘
                      ↕
         Passive View (size=32-64):
         ┌─────────────────────────────────┐
         │  Backup/shuffle candidates      │
         │   F, G, H, I, J, K, L, M, ...   │
         └─────────────────────────────────┘


Plumtree Broadcast Pattern:

Initial Eager Push (tree overlay):
                  ┌───┐
                  │ A │  (source)
                  └─┬─┘
         ───────────┼───────────
        │           │           │
     ┌──▼──┐    ┌──▼──┐    ┌──▼──┐
     │  B  │    │  C  │    │  D  │  Eager push (full payload)
     └──┬──┘    └──┬──┘    └──┬──┘
        │           │           │
   ┌────┴─────┐    │      ┌────┴─────┐
   │          │    │      │          │
┌──▼──┐   ┌──▼──┐ │   ┌──▼──┐   ┌──▼──┐
│  E  │   │  F  │ │   │  G  │   │  H  │
└─────┘   └─────┘ │   └─────┘   └─────┘
                  │
              ┌───▼───┐
              │   I   │
              └───────┘

Lazy Push (optimization):
     ┌───┐                          ┌───┐
     │ B │ ─────────────────────>  │ J │
     └───┘  LazyIDs[msg123, ...]   └───┘
                                       │
                                       │ (J detects missing msg)
                                       │
                                       ▼
     ┌───┐  FetchMissing[msg123]   ┌───┐
     │ B │ <─────────────────────  │ J │
     └───┘                          └───┘
```

Config knobs:

- targetFanout, activeViewSize, passiveViewSize
- eagerPushTTL, lazyPushIntervalMs


## State Model and CRDT Support

- Keyspace: Bytes keys in a single logical map, optionally namespaced by prefix (e.g., tenant/region/shard).
- Values: CRDTs with merge semantics:
  - LWW-Register<String|Bytes> with wall-clock timestamp and node ID tie-breakers
  - OR-Set<Bytes> using unique element tags (dot clocks)
  - GCounter, PNCounter
- Versioning: Per-key vector clocks (or dotted version vectors) to track causality; tombstones for removals with expiration after safe convergence horizon.
- Conflicts: Resolved via CRDT merge; for LWW, resolve by (timestamp, nodeId).

```
CRDT Merge Examples:

LWW-Register (Last-Write-Wins):
┌─────────────┐              ┌─────────────┐
│   Node A    │              │   Node B    │
│ key="user"  │              │ key="user"  │
│ value="alice"│             │ value="bob" │
│ ts=100      │              │ ts=105      │
│ node=A      │              │ node=B      │
└──────┬──────┘              └──────┬──────┘
       │                            │
       │      Gossip / AE           │
       │<───────────────────────────>│
       │                            │
       ▼                            ▼
┌─────────────┐              ┌─────────────┐
│ Merge:      │              │ Merge:      │
│ max(ts)=105 │              │ max(ts)=105 │
│ value="bob" │ ✓            │ value="bob" │ ✓
└─────────────┘              └─────────────┘
       Converged to same value!


OR-Set (Observed-Remove Set):
┌──────────────────┐         ┌──────────────────┐
│   Node A         │         │   Node B         │
│ key="tags"       │         │ key="tags"       │
│ adds: {          │         │ adds: {          │
│   ("red", tag1)  │         │   ("blue", tag2) │
│   ("green",tag3) │         │ }                │
│ }                │         │ removes: {       │
│ removes: {}      │         │   ("red", tag1)  │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         │         Gossip             │
         │<───────────────────────────>│
         │                            │
         ▼                            ▼
┌──────────────────┐         ┌──────────────────┐
│ Merge:           │         │ Merge:           │
│ adds ∪ adds'     │         │ adds ∪ adds'     │
│ removes ∪ rem'   │         │ removes ∪ rem'   │
│                  │         │                  │
│ Result:          │         │ Result:          │
│ {"blue",         │ ✓       │ {"blue",         │ ✓
│  "green"}        │         │  "green"}        │
│ (tag1 removed)   │         │ (tag1 removed)   │
└──────────────────┘         └──────────────────┘
       Converged with CRDT semantics!


Vector Clock Causality:
   Node A: {A:3, B:1, C:2}    Concurrent with
   Node B: {A:2, B:2, C:2}    (neither dominates)
   
   → Keep both, merge via CRDT rules
   
   Node A: {A:5, B:3, C:2}    Dominates
   Node B: {A:4, B:2, C:2}    (all clocks ≥)
   
   → Keep A's version (causally newer)
```


## Merkle Search Tree (MST) for Anti-Entropy

We use a Merkle Search Tree as an ordered, hash-indexed map over the keyspace.

Reference implementation: [domodwyer/merkle-search-tree](https://github.com/domodwyer/merkle-search-tree) - A Rust implementation based on the 2019 paper "Merkle Search Trees: Efficient State-Based CRDTs in Open Networks" by Auvolat & Taïani.

- Structure: A balanced, ordered tree (B-tree style fanout) where each internal node stores ordered key separators and child hashes; leaves store key/value digests and optional range summaries. Each node's hash commits to its children's hashes and separator keys.
- Hashing: SHA-256 over a canonical encoding of (nodeType, separators, childHashes) or (leafKeyHashes, valueDigests). Keys are hashed (keyHash = SHA-256(key)); values use valueDigest that commits to CRDT payload and version metadata.
- Range proofs: Presence/absence proofs for individual keys and range proofs for arbitrary intervals [a, b].
- Incremental updates: Insert/update/delete recomputes hashes on the path; batched rebalancing amortizes costs.
- Snapshots: Periodic immutable MST snapshots are persisted for quick anti-entropy comparisons.

```
MST Structure (Fanout F=4):

                        ┌─────────────────────────────────┐
                        │      Root: H(root)              │
                        │  [-, "m", "p", "z"]             │
                        └─┬────────┬────────┬────────┬────┘
                          │        │        │        │
        ┌─────────────────┘        │        │        └─────────────────┐
        │                          │        │                          │
  ┌─────▼──────┐          ┌───────▼─────┐  ┌────────▼─────┐   ┌──────▼─────┐
  │ H(node1)   │          │  H(node2)   │  │  H(node3)    │   │ H(node4)   │
  │ [-, "d",   │          │ ["m", "n",  │  │ ["p", "r",   │   │ ["z", -]   │
  │  "f", "k"] │          │  "o"]       │  │  "s", "x"]   │   │            │
  └─┬──┬───┬──┬┘          └─┬──┬───┬───┘  └─┬──┬───┬───┬─┘   └─┬──────────┘
    │  │   │  │             │  │   │        │  │   │   │       │
    ▼  ▼   ▼  ▼             ▼  ▼   ▼        ▼  ▼   ▼   ▼       ▼
  Leaf Leaf...            Leaf Leaf Leaf   Leaf nodes       Leaf nodes
  nodes                    nodes            ["p":"paris"]   ["zebra":value]
  ["a":v]                 ["m":"moon"]      ["q":"quux"]    
  ["b":v]                 ["n":"node"]      ["r":"rust"]
  ...                     ["o":"ops"]       ["s":"swim"]


Leaf Node Detail:
┌──────────────────────────────────────────┐
│  Leaf Node: H(leaf)                      │
│  ┌────────────────────────────────────┐  │
│  │ key1: keyHash1, valueDigest1, vc1  │  │
│  │ key2: keyHash2, valueDigest2, vc2  │  │
│  │ key3: keyHash3, valueDigest3, vc3  │  │
│  │ ...                                │  │
│  └────────────────────────────────────┘  │
│  H(leaf) = SHA256(keyHashes +            │
│                   valueDigests +          │
│                   versions)               │
└──────────────────────────────────────────┘
```

Anti-entropy session (pull-based):

1) Initiator sends an MST summary: rootHash, snapshotEpoch, optional segment sketches (top-k subtree hashes, HyperLogLog for cardinality).
2) Responder compares against local snapshot: if rootHash matches at same epoch, done; else it binary-searches differences by requesting child hashes down the tree (hash-aided divide-and-conquer).
3) Range negotiation: Once divergent subtrees are localized, initiator requests range proofs for key intervals; responder returns:
   - Minimal set of ranges with proofs
   - Deltas for keys in those ranges (compact CRDT deltas and tombstones)
4) Apply: Initiator verifies proofs, applies CRDT merges, updates MST; optionally streams back missing items the responder lacks (two-way repair).
5) Commit: Both sides may advertise a new snapshotEpoch when durable.

```
Anti-Entropy Session Flow:

┌───────────┐                                       ┌───────────┐
│ Initiator │                                       │ Responder │
│  Node A   │                                       │  Node B   │
└─────┬─────┘                                       └─────┬─────┘
      │                                                   │
      │ 1. AE.Summary                                     │
      │    { rootHash: H1, epoch: 42,                    │
      │      subtreeHashes: [...] }                      │
      │──────────────────────────────────────────────>   │
      │                                                   │
      │                                    ┌──────────────┴────┐
      │                                    │ Compare:          │
      │                                    │ H1 != localRoot   │
      │                                    │ Find divergent    │
      │                                    │ subtrees          │
      │                                    └──────────────┬────┘
      │                                                   │
      │ 2. AE.ChildHashes                                │
      │    { ranges: [["m","p"), ["p","z")],             │
      │      childHashes: [H_m, H_p, ...] }              │
      │<─────────────────────────────────────────────────│
      │                                                   │
 ┌────┴───────┐                                          │
 │ Identify   │                                          │
 │ missing    │                                          │
 │ ranges     │                                          │
 └────┬───────┘                                          │
      │                                                   │
      │ 3. AE.Request                                    │
      │    { ranges: [["p", "s")],                       │
      │      knownSiblingHashes: [...] }                 │
      │──────────────────────────────────────────────>   │
      │                                                   │
      │                                    ┌──────────────┴────┐
      │                                    │ Generate proofs   │
      │                                    │ for ranges,       │
      │                                    │ collect deltas    │
      │                                    └──────────────┬────┘
      │                                                   │
      │ 4. AE.Proof                                      │
      │    { rangeProofs: [...],                         │
      │      deltas: [{ key, value, vc }, ...],          │
      │      tombstones: [...] }                         │
      │<─────────────────────────────────────────────────│
      │                                                   │
 ┌────┴───────┐                                          │
 │ Verify     │                                          │
 │ proofs,    │                                          │
 │ merge      │                                          │
 │ CRDTs,     │                                          │
 │ update MST │                                          │
 └────┬───────┘                                          │
      │                                                   │
      │ 5. (Optional) AE.TwoWayDelta                     │
      │    { deltas: [items B is missing] }              │
      │──────────────────────────────────────────────>   │
      │                                                   │
      │ 6. AE.Commit                                     │
      │    { snapshotEpoch: 43 }                         │
      │<─────────────────────────────────────────────────│
      │                                                   │
      │ AE.Commit                                        │
      │    { snapshotEpoch: 43 }                         │
      │──────────────────────────────────────────────>   │
      │                                                   │
      ▼                                                   ▼
   Converged                                          Converged
```

Optimizations:

- Fanout F ≥ 8 to reduce depth; cache hot internal nodes.
- Batch small deltas; coalesce range requests; use gzip/zstd.
- Sketch-assisted targeting: Use Count-Min sketches or Bloom filters per subtree to bias range requests.
- Partial replication: Namespace/shard filters; each session exchanges interest sets to skip irrelevant ranges.
- Proof compression: Share sibling hashes already known in earlier steps.

Complexities:

- Tombstone retention must exceed the max anti-entropy interval across peers.
- Clock skew mitigated by causal metadata; never rely solely on timestamps for deletion safety.


## Dissemination Strategy

- Push gossip: For hot keys, send small CRDT deltas to eager peers; include causal metadata to avoid over-merging.
- Pull anti-entropy: Periodic, staggered sessions ensure eventual convergence for cold or missed updates.
- Piggybacking: Membership messages, message IDs, and small sketches ride along push/pull traffic.

```
Push vs Pull Strategy:

Hot Keys (Push Gossip - Low Latency):
   Update to key "session:user123"
            │
            ▼
   ┌────────────────┐
   │   Node A       │
   │  (originator)  │
   └────┬───────────┘
        │ PushDelta (immediate, <50ms)
        │
        ├────────────────────────┬────────────────────┐
        │                        │                    │
        ▼                        ▼                    ▼
   ┌─────────┐            ┌─────────┐          ┌─────────┐
   │ Node B  │            │ Node C  │          │ Node D  │
   └─────────┘            └─────────┘          └─────────┘
     Active fanout ~5 peers, p99 < 300ms


Cold Keys (Pull Anti-Entropy - Bounded Staleness):
   Missed update or network partition
   
   Time ────────────────────────────────────────────>
        0s              30s             60s        90s
        │               │               │          │
        │               ▼               │          │
        │        AE Session 1           │          │
        │        (partial sync)         │          │
        │               │               ▼          │
        │               │        AE Session 2      │
        │               │        (converging)      │
        │               │               │          ▼
        │               │               │   AE Session 3
        │               │               │   (fully synced)
        │               │               │          │
        └───────────────┴───────────────┴──────────┘
                Eventual convergence < 3 × AE interval


Hybrid: Hot→Cold Transition
                                        
   ┌────────────────────────────────────────────────┐
   │ Key Access Pattern                             │
   │                                                │
   │  Update ▲                                      │
   │  Rate   │  Hot (push)                          │
   │         │   ████                                │
   │         │   ████                                │
   │         │   ████  Cold (pull AE)               │
   │         │   ████  ░░░░░░░░░░░░░░               │
   │         └────────────────────────────> Time    │
   │              Transition threshold              │
   └────────────────────────────────────────────────┘
   
   Adaptive: Track per-key update rate; switch strategies
```

Scheduling:

- pushIntervalMs adaptive based on recent update rate per key/namespace.
- antiEntropyIntervalMs jittered per node; cap concurrent sessions (see backpressure).
- prioritize peers with divergent summaries or high lag.


## Wire Protocol

Transport:

- Default: WireGuard tunnels via [BoringTun](https://github.com/cloudflare/boringtun) userspace implementation
- Encryption: Noise protocol framework (Noise_IK) with Curve25519 ECDH, ChaCha20-Poly1305 AEAD, BLAKE2s hashing
- Per-peer tunnels: Each peer pair establishes a WireGuard tunnel for authenticated, encrypted communication
- UDP-based: All messages (control, broadcast, anti-entropy) sent over WireGuard UDP tunnels
- Handshake: 1-RTT handshake with optional pre-shared keys for additional security

Why WireGuard over QUIC:

- **Simplicity**: WireGuard has ~4,000 lines of code vs QUIC's 10,000+ lines; easier to audit and verify
- **Performance**: ChaCha20-Poly1305 is faster than AES-GCM on systems without hardware acceleration
- **Battery efficiency**: Stateless design reduces keepalive overhead on mobile/IoT devices
- **NAT traversal**: Built-in roaming support; connections survive IP changes seamlessly
- **Zero-config security**: No certificate management; public keys are the identity
- **Battle-tested**: Used in production by Cloudflare, Tailscale, and millions of VPN users
- **Rust native**: BoringTun provides a pure-Rust implementation, avoiding C FFI and improving safety

Message types (selected):

- JoinRequest{ nodeId, addrs, incarnation, capabilities }
- JoinResponse{ accepted, membershipSnapshot, epoch }
- Ping{ nonce, seq }
- Ack{ nonce, seq, rtt }
- IndirectPing{ target, nonce, seq }
- MemberUpdate{ Alive|Suspect|Confirm|Leave, nodeId, incarnation }
- PushDelta{ key, deltaDigest, deltaPayload, vectorClock, tombstone? }
- LazyIDs{ messageIds[] }
- FetchMissing{ messageIds[] }
- AE.Summary{ snapshotEpoch, rootHash, subtreeSketches? }
- AE.Request{ ranges[], knownSiblingHashes[] }
- AE.Proof{ rangeProofs[], deltas[], tombstones[] }
- AE.Commit{ snapshotEpoch }

Framing:

- Length-prefixed protobuf or flatbuffers; optional varint framing for small messages.
- Messages sent through WireGuard tunnels are already encrypted and authenticated at the transport layer.
- Application-level message IDs track causality; WireGuard's built-in replay protection prevents packet replay attacks.
- Optional Ed25519 signatures for non-repudiation at the application layer (beyond WireGuard's transport auth).


## Backpressure, Flow Control, and Rate Limiting

- Token buckets per peer and per message class (membership, push, AE).
- Concurrent anti-entropy sessions limited (maxAESessions); queue with fair scheduling (per-namespace and per-peer).
- Adaptive fanout: reduce eager fanout under congestion; prefer lazy push.
- WireGuard provides transport-level congestion control via UDP; application-level rate limiting prevents overwhelming peers.
- Drop policy: Drop lazy IDs first; degrade to summary-only under extreme pressure.
- Per-tunnel send buffers prevent head-of-line blocking across peer connections.


## Security and Identity

- Node identity: Curve25519 keypair for WireGuard tunnels; nodeId = SHA-256(pubkey).
- Mutual auth: WireGuard's Noise_IK handshake provides mutual authentication via pre-shared public keys.
- Tunnel encryption: ChaCha20-Poly1305 AEAD cipher with automatic key rotation every 2 minutes or 2^64-1 packets.
- Message integrity: WireGuard's AEAD provides authenticity; replay protection built into protocol.
- Access control: Allowlists enforced at tunnel establishment; peer discovery restricted to known public keys.
- Privacy: All gossip traffic encrypted end-to-end through WireGuard tunnels; forward secrecy via ephemeral keys.

```
Security Architecture:

Node Identity:
┌──────────────────────────────────────┐
│  Curve25519 Keypair (WireGuard)      │
│  ┌────────────┐    SHA-256           │
│  │ Private Key│───────────>  NodeId  │
│  └────────────┘              (public)│
│  ┌────────────┐                      │
│  │ Public Key │  (shared with peers) │
│  └────────────┘                      │
└──────────────────────────────────────┘

WireGuard Tunnel Establishment:
┌────────┐                              ┌────────┐
│ Node A │                              │ Node B │
└───┬────┘                              └───┬────┘
    │                                       │
    │ WireGuard Handshake (Noise_IK)       │
    │<─────────────────────────────────────>│
    │  1. Initiator → Responder:            │
    │     (ephemeral_pub, static_pub_enc,   │
    │      timestamp_enc)                   │
    │                                       │
    │  2. Responder → Initiator:            │
    │     (ephemeral_pub, empty_enc)        │
    │                                       │
    │  Tunnel established ✓                 │
    │  - Mutual authentication via pubkeys  │
    │  - Perfect forward secrecy            │
    │  - Automatic key rotation             │
    │                                       │

Encrypted Message Flow:
┌──────────────────────────────────────────────────┐
│ Application Layer:                               │
│  ┌─────────────────────────────────────────┐    │
│  │ Message (protobuf/flatbuf)              │    │
│  │  - type: PushDelta                      │    │
│  │  - key, delta, vectorClock, ...         │    │
│  └─────────────┬───────────────────────────┘    │
│                │                                 │
│                ▼                                 │
│  ┌─────────────────────────────────────────┐    │
│  │ WireGuard Tunnel (per-peer)             │    │
│  │  - ChaCha20-Poly1305 encryption         │    │
│  │  - Replay protection (64-bit counter)   │    │
│  │  - MAC authentication                   │    │
│  └─────────────┬───────────────────────────┘    │
│                │                                 │
│                ▼                                 │
│           UDP Packet                             │
└──────────────────────────────────────────────────┘

Security Guarantees:
  ✓ Transport-level encryption (ChaCha20-Poly1305)
  ✓ Peer authentication (Noise_IK handshake)
  ✓ Replay protection (WireGuard counters)
  ✓ Forward secrecy (ephemeral keys)
  ✓ Key rotation (automatic every 2 min)
  ✓ Tunnel allowlist (peer public keys)
```


## Persistence and Recovery

- WAL for CRDT deltas and membership updates (fsync policy configurable).
- Periodic snapshots of value store and MST index; snapshotEpoch advertised in AE.
- Crash recovery replays WAL then loads latest snapshot; validates MST root.
- Compaction trims old tombstones once safe (based on convergence watermarks from peers).

```
Storage Layout:

┌─────────────────────────────────────────────────────┐
│  Disk Layout                                        │
│                                                     │
│  data/                                              │
│    ├── wal/                                         │
│    │    ├── 00000001.log  (active)                  │
│    │    ├── 00000002.log  (sealed)                  │
│    │    └── 00000003.log  (sealed)                  │
│    │                                                │
│    ├── snapshots/                                   │
│    │    ├── epoch_040.snap  (older)                 │
│    │    ├── epoch_041.snap                          │
│    │    └── epoch_042.snap  (latest) ✓              │
│    │                                                │
│    └── mst/                                         │
│         ├── index_042.mst   (current tree)          │
│         └── index_041.mst   (previous)              │
│                                                     │
└─────────────────────────────────────────────────────┘

Write Path:
   Application Update
         │
         ▼
   ┌──────────┐
   │ In-Memory│
   │  CRDT    │
   │  Store   │
   └────┬─────┘
        │
        ├────────────────┐
        │                │
        ▼                ▼
   ┌─────────┐     ┌──────────┐
   │   WAL   │     │   MST    │
   │ (append)│     │ (update) │
   └─────────┘     └──────────┘
        │                │
        ▼                ▼
    [Durable]       [In-Memory]
    
    Every N minutes:
         │
         ▼
    ┌──────────┐
    │ Snapshot │
    │ (MST +   │
    │  Values) │
    └──────────┘


Crash Recovery Flow:
   Node Restart
         │
         ▼
   ┌─────────────────────┐
   │ 1. Load latest      │
   │    snapshot         │
   │    (epoch 42)       │
   └──────┬──────────────┘
          │
          ▼
   ┌─────────────────────┐
   │ 2. Replay WAL       │
   │    entries > 42     │
   │    (apply deltas)   │
   └──────┬──────────────┘
          │
          ▼
   ┌─────────────────────┐
   │ 3. Rebuild MST      │
   │    Verify root hash │
   └──────┬──────────────┘
          │
          ▼
   ┌─────────────────────┐
   │ 4. Resume normal    │
   │    operations       │
   └─────────────────────┘


Compaction:
   Time ────────────────────────────────────>
         
   Tombstone created at T₀
         │
         │   Safe horizon = max(peer_convergence_times)
         │   │
         │   │              All peers synced
         │   │              │
   ──────●───┼──────────────●───────────────>
         │   │              │
         │   └──(3×AE)──────┘
         │                  │
         └──────────────────▼
                    Tombstone can be
                    safely removed
```


## Observability

Metrics (suggested):

- membership.suspect_rate, membership.flaps
- broadcast.eager_fanout, broadcast.dup_suppressed
- push.bytes, push.deltas, push.drop_lazy
- ae.sessions_active, ae.bytes_in, ae.bytes_out, ae.duration_ms
- ae.range_proofs, ae.keys_repaired, ae.root_match_rate
- storage.wal_bytes, snapshot.duration_ms, compaction.tombstones_removed
- transport.wireguard_handshakes, wireguard_tunnels_active, wireguard_rx_bytes, wireguard_tx_bytes
- transport.udp_packet_loss, wireguard_keepalives_sent

Tracing:

- Spans for Join, Ping, Broadcast, AE session (with range subtree annotations).
- Log correlation IDs include messageId and peerId.


## Configuration (indicative defaults)

- targetFanout: 5
- activeViewSize: 8, passiveViewSize: 64
- probeIntervalMs: 1000, probeTimeoutMs: 200, indirectK: 3
- suspicionTimeoutMs: adaptive (1–10s)
- eagerPushTTL: 3, lazyPushIntervalMs: 250
- pushIntervalMs: adaptive (50–500ms)
- antiEntropyIntervalMs: 30–90s jittered
- maxAESessions: min(4, numCpus)
- tombstoneTTL: ≥ 3 × maxAEInterval
- transport: wireguard (boringtun)
- wireguardPort: 51820
- wireguardKeepalive: 25s
- hash: sha256


## Rust Crate Structure and Public APIs

Crates:

- merkleflow-core: membership, overlay, broadcast
- merkleflow-mst: Merkle Search Tree, proofs, range sync
- merkleflow-crdt: CRDT implementations and delta encoding
- merkleflow-transport: WireGuard tunnels (BoringTun wrapper) and message framing
- merkleflow-node: integration, persistence, config, metrics
- merkleflow-cli: admin/testing tool

Dependencies:

- boringtun: WireGuard userspace implementation in Rust

Key traits (sketch):

```rust
pub trait Transport {
    type Conn: AsyncRead + AsyncWrite + Unpin + Send;
    async fn connect(&self, addr: SocketAddr, peer_id: NodeId) -> Result<Self::Conn>;
    async fn listen(&self, addr: SocketAddr) -> Result<Listener>;
}

pub trait CRDT: Send + Sync {
    fn merge(&mut self, other: &Self);
    fn delta_since(&self, v: &Version) -> Option<Bytes>;
    fn apply_delta(&mut self, delta: &[u8]) -> Result<()>;
}

pub trait MstIndex {
    fn root(&self) -> Hash;
    fn diff_summary(&self, depth: usize) -> Summary;
    fn range_proof(&self, start: &[u8], end: &[u8]) -> Proof;
    fn apply(&mut self, deltas: Vec<Delta>) -> Result<()>;
}

pub struct MerkleFlowNode { /* ... */ }
impl MerkleFlowNode {
    pub async fn spawn(config: Config) -> Result<Self> { /* ... */ }
    pub async fn put<K: AsRef<[u8]>>(&self, key: K, value: Value) -> Result<()> { /* ... */ }
    pub async fn delete<K: AsRef<[u8]>>(&self, key: K) -> Result<()> { /* ... */ }
    pub fn subscribe(&self, prefix: &[u8]) -> Subscription { /* ... */ }
}
```


## Testing and Simulation Plan

- Property tests: CRDT merges, MST proofs (presence/absence/range), idempotency.
- Deterministic simulations: churn, partitions, asymmetric loss; measure convergence time and bandwidth.
- Fault injection: delayed acks, paused storage, snapshot failures.
- Scale tests: up to 10k nodes (simulation) with Zipfian key popularity.


## Performance Targets

- Membership false-positive rate: < 1e-4 under 1% packet loss
- Broadcast duplication factor: < 1.5× optimal tree after warm-up
- Hot-key p99 replication latency (< 32KB delta): < 300ms intra-DC, < 800ms cross-DC
- Anti-entropy full diff for 1M keys, 1% divergence: < 3s @ 50 Mbps link
- Storage write amplification (WAL+snapshot): < 2.5× average


## Open Questions / Future Work

- Multi-cluster federation and peering policies
- Selective encryption and searchable encryption at leaves
- Adaptive MST fanout per subtree based on churn/heat
- ERASURE-coded lazy push for low-bandwidth links


## Appendix: MST Proof Sketch

Range proof for [a, b]:

- Provide path from root to first leaf ≥ a and last leaf ≤ b, including all required sibling hashes for recomputation.
- Include all leaves within [a, b] with their (keyHash, valueDigest, version) tuples.
- For gaps, include boundary separators proving absence.
- Verifier recomputes hashes bottom-up and confirms subtree root equals advertised hash.

```
Range Proof Example [m, s):

Full Tree:
                    ┌────────────────┐
                    │   Root: H(R)   │
                    │ [-, m, p, z]   │
                    └────┬───┬───┬───┘
                         │   │   │
         ┌───────────────┘   │   └───────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐        ┌────▼────┐
    │H(node1) │         │H(node2) │        │H(node3) │ ← Sibling
    │[a,d,g,k]│         │[m,n,o]  │        │[p,r,s,x]│    hash
    └─────────┘         └──┬──┬───┘        └──┬──┬───┘    needed
      (skip)               │  │               │  │
                           │  │               │  │
                      ┌────▼──▼────┐    ┌─────▼──▼─────┐
                      │   Leaves:  │    │   Leaves:    │
                      │  m: "moon" │    │  p: "paris"  │
                      │  n: "node" │    │  r: "rust"   │
                      │  o: "ops"  │    │  s: "swim"   │ ← excluded
                      └────────────┘    └──────────────┘
                           ▲                    ▲
                           └──── In Range ──────┘ (not s)


Proof Structure for Range [m, s):
┌──────────────────────────────────────────────────────┐
│ RangeProof {                                         │
│                                                      │
│   // Path from root                                  │
│   root_hash: H(R)                                    │
│   path: [                                            │
│     PathNode {                                       │
│       separators: [-, m, p, z],                      │
│       siblings: [H(node1), -, -, H(node3)']          │
│     }                                                │
│   ]                                                  │
│                                                      │
│   // Leaves in range                                 │
│   leaves: [                                          │
│     { key: "m", hash: H_m, value: "moon", vc: V1 }  │
│     { key: "n", hash: H_n, value: "node", vc: V2 }  │
│     { key: "o", hash: H_o, value: "ops",  vc: V3 }  │
│     { key: "p", hash: H_p, value: "paris",vc: V4 }  │
│     { key: "r", hash: H_r, value: "rust", vc: V5 }  │
│   ]                                                  │
│                                                      │
│   // Boundary proof (s exists but excluded)          │
│   right_boundary: Some("s")                          │
│ }                                                    │
└──────────────────────────────────────────────────────┘

Verification Steps:
   1. Compute leaf hashes: H(m), H(n), H(o), H(p), H(r)
   2. Group into node hashes: H(node2), H(node3_partial)
   3. Combine with siblings: H(node1), H(node3)
   4. Recompute root: H(R')
   5. Verify: H(R') == H(R) ✓
   
   → Proof valid: range [m,s) authenticated!
```
