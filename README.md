# MerkleFlow

**Production-ready gossip protocol with Merkle Search Tree anti-entropy for large-scale, eventually-consistent state replication.**

IMPORTANT: Project in research and design phase. Drafts only.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-stable-orange.svg)](https://www.rust-lang.org/)

MerkleFlow combines battle-tested distributed systems algorithms into a modern, efficient gossip subsystem designed for high availability, low latency, and sub-linear bandwidth scaling.

---

## Features

- 🔄 **SWIM + Lifeguard**: Robust membership and failure detection with local health awareness
- 🌳 **Merkle Search Tree (MST)**: Efficient anti-entropy using ordered, hash-indexed state trees
- 📊 **CRDT-First**: Built-in support for LWW-Register, OR-Set, G-Counter, PN-Counter
- 🚀 **Hybrid Dissemination**: Push gossip for hot keys (low latency) + pull anti-entropy for cold keys (bounded staleness)
- 🌐 **HyParView + Plumtree**: Self-organizing overlay with epidemic broadcast and spanning tree optimization
- 🔒 **Secure by Default**: WireGuard tunnels (via BoringTun) with Noise protocol, Curve25519, ChaCha20-Poly1305 encryption
- ⚡ **High Performance**: UDP-based WireGuard transport, sub-300ms p99 replication latency intra-DC
- 🛡️ **Production Ready**: WAL, snapshots, backpressure, rate limiting, metrics, tracing

---

## Use Cases

MerkleFlow is ideal for:

- **Service Discovery**: Membership and health propagation across microservices
- **Distributed Configuration**: Feature flags, A/B test configs, runtime parameters
- **Session Replication**: Stateful failover for web applications
- **Distributed Caching**: Share hot cache entries and invalidations
- **Real-Time Presence**: Track online users and activity in collaboration apps
- **Edge/IoT Coordination**: Sync device states with intermittent connectivity
- **Multi-Region Metadata**: Replicate routing tables and shard assignments
- **Leaderless KV Stores**: Build Dynamo-style databases with tunable consistency
- **Observability**: Aggregate cluster-wide metrics and alerts
- **Multi-Tenant SaaS**: Isolated, eventually-consistent state per tenant

See [design.md](design.md#use-cases) for detailed examples.

---

## Architecture Highlights

```text
┌────────────────────────────────────────────────────────────┐
│                    MerkleFlow Node                         │
├────────────────────────────────────────────────────────────┤
│  Membership (SWIM)  │  Overlay (HyParView)  │  Broadcast  │
│──────────────────────────────────────────────────────────  │
│         State Replication (Push + MST Anti-Entropy)        │
│──────────────────────────────────────────────────────────  │
│       CRDT Store (LWW, OR-Set, Counters) + MST Index      │
│──────────────────────────────────────────────────────────  │
│           Persistence (WAL + Snapshots)                    │
└────────────────────────────────────────────────────────────┘
          WireGuard Transport (BoringTun/Noise Protocol)
```

### Key Design Decisions

- **MST for Anti-Entropy**: Efficient state reconciliation with O(log n) depth and compact range proofs
- **Adaptive Push/Pull**: Hot keys use push gossip (low latency); cold keys use periodic pull (bandwidth efficient)
- **CRDT Conflict Resolution**: Deterministic, commutative merges eliminate coordination overhead
- **WireGuard Transport**: Lightweight UDP tunnels with Noise protocol, automatic encryption, and perfect forward secrecy
  - Simpler than QUIC (~4k LOC vs 10k+ LOC)
  - ChaCha20-Poly1305 optimized for software-only encryption
  - Battle-tested by Cloudflare and Tailscale in production
  - Zero-config security (public keys = identity)
- **Vector Clocks**: Track causality to prevent stale overwrites
- **BoringTun Integration**: Userspace WireGuard implementation in Rust for portability and performance

Read the full [design document](design.md) for technical details.

---

## Quick Start

### Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
merkleflow-node = "0.1"
merkleflow-crdt = "0.1"
```

### Basic Usage

```rust
use merkleflow_node::{Config, MerkleFlowNode};
use merkleflow_crdt::LWWRegister;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Configure and spawn a node
    let config = Config::builder()
        .bind_addr("0.0.0.0:7946".parse()?)
        .seed_nodes(vec!["10.0.1.5:7946".parse()?])
        .target_fanout(5)
        .build();
    
    let node = MerkleFlowNode::spawn(config).await?;
    
    // Store a value (LWW-Register)
    let value = LWWRegister::new("alice".to_string());
    node.put("user:123:name", value).await?;
    
    // Subscribe to updates
    let mut sub = node.subscribe(b"user:");
    while let Some(update) = sub.recv().await {
        println!("Updated: {:?}", update);
    }
    
    Ok(())
}
```

### Configuration

Key tuning parameters:

```rust
Config::builder()
    .target_fanout(5)                  // Eager push fanout
    .probe_interval_ms(1000)           // SWIM probe interval
    .anti_entropy_interval_ms(60_000)  // MST sync interval
    .tombstone_ttl_secs(300)           // 5 min (>= 3x AE interval)
    .transport(Transport::WireGuard)   // WireGuard via BoringTun (default)
    .wireguard_port(51820)             // WireGuard listen port
    .wireguard_keepalive_secs(25)      // Keep NAT mappings alive
    .build()
```

---

## Project Structure

```text
merkleflow/
├── merkleflow-core/       # Membership, overlay, broadcast
├── merkleflow-mst/        # Merkle Search Tree, proofs, range sync
├── merkleflow-crdt/       # CRDT implementations and delta encoding
├── merkleflow-transport/  # QUIC/Noise and message framing
├── merkleflow-node/       # Integration, persistence, config, metrics
└── merkleflow-cli/        # Admin and testing tool
```

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Membership false-positive rate | < 0.01% @ 1% packet loss |
| Broadcast duplication factor | < 1.5× optimal tree |
| Hot-key p99 replication latency (intra-DC) | < 300ms |
| Hot-key p99 replication latency (cross-DC) | < 800ms |
| Anti-entropy full diff (1M keys, 1% divergence) | < 3s @ 50 Mbps |
| Storage write amplification | < 2.5× |

---

## Development

### Prerequisites

- Rust 1.75+ (stable)
- Protocol Buffers compiler (for wire format)

### Build

```bash
cargo build --release
```

### Test

```bash
# Unit and integration tests
cargo test

# Property tests
cargo test --features proptest

# Benchmarks
cargo bench
```

### Simulation

MerkleFlow includes a deterministic simulation harness for testing under churn, partitions, and asymmetric loss:

```bash
cargo run --bin merkleflow-sim -- \
  --nodes 100 \
  --churn-rate 0.05 \
  --partition-prob 0.01 \
  --duration 600s
```

---

## Observability

MerkleFlow exposes Prometheus metrics and OpenTelemetry traces:

**Key Metrics**:

- `merkleflow_membership_suspect_rate`
- `merkleflow_broadcast_dup_suppressed`
- `merkleflow_ae_sessions_active`
- `merkleflow_ae_keys_repaired`
- `merkleflow_storage_wal_bytes`

**Tracing**:

- Distributed traces for Join, Ping, Broadcast, and AE sessions
- Correlation IDs include `messageId` and `peerId`

---

## Comparison to Alternatives

| Feature | MerkleFlow | Consul (Serf) | Memberlist | Cassandra |
|---------|------------|---------------|------------|-----------|
| Membership | SWIM + Lifeguard | SWIM | SWIM | Gossip |
| Anti-Entropy | MST | Periodic full sync | None | Merkle trees |
| CRDTs | ✅ Built-in | ❌ | ❌ | Limited |
| Transport | WireGuard/UDP | TCP/UDP | TCP/UDP | TCP |
| Encryption | ChaCha20-Poly1305 | TLS (optional) | None | TLS (optional) |
| Ordered Keyspace | ✅ | ❌ | ❌ | ✅ |
| Range Proofs | ✅ | ❌ | ❌ | ✅ |
| Adaptive Push/Pull | ✅ | ❌ | ❌ | ❌ |

MerkleFlow's **Merkle Search Tree** provides efficient, incremental state synchronization with O(log n) comparison depth, outperforming periodic full syncs and traditional Merkle trees for ordered keyspaces. **WireGuard tunnels** provide built-in encryption and authentication with minimal overhead.

---

## References

- [Design Document](design.md) - Complete technical specification
- [MST Implementation](https://github.com/domodwyer/merkle-search-tree) - Reference Rust MST library
- [BoringTun](https://github.com/cloudflare/boringtun) - Cloudflare's userspace WireGuard implementation in Rust
- **Papers**:
  - [SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf)
  - [Lifeguard: Local Health Awareness for More Accurate Failure Detection](https://arxiv.org/abs/1707.00788)
  - [HyParView: A Membership Protocol for Reliable Gossip-Based Broadcast](https://asc.di.fct.unl.pt/~jleitao/pdf/dsn07-leitao.pdf)
  - [Plumtree: Epidemic Broadcast Trees](https://www.gsd.inesc-id.pt/~ler/reports/srds07.pdf)
  - [Merkle Search Trees: Efficient State-Based CRDTs in Open Networks](https://hal.science/hal-02303490)
  - [WireGuard: Next Generation Kernel Network Tunnel](https://www.wireguard.com/papers/wireguard.pdf)
  - [Noise Protocol Framework](https://noiseprotocol.org/noise.html)

---

## Contributing

Contributions are welcome! Please:

1. Open an issue for discussion before major changes
2. Follow Rust style guidelines (`cargo fmt`, `cargo clippy`)
3. Add tests for new features
4. Update documentation as needed

---

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.

---

## Roadmap

- [ ] Core membership and overlay (SWIM + HyParView)
- [ ] MST anti-entropy implementation
- [ ] CRDT value types and merge logic
- [ ] WireGuard transport layer (BoringTun integration)
- [ ] Per-peer tunnel management and key rotation
- [ ] WAL and snapshot persistence
- [ ] Metrics and tracing instrumentation
- [ ] Simulation harness
- [ ] Production hardening and benchmarks
- [ ] Multi-cluster federation
- [ ] Adaptive MST fanout per subtree

---

## Acknowledgments

MerkleFlow builds on decades of distributed systems research. Special thanks to the authors of SWIM, Lifeguard, HyParView, Plumtree, and the Merkle Search Tree paper.

---

Built with ❤️ in Rust
