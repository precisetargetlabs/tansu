# 2026-02-06: Experiment Kickoff - tiros-engine

## Background

We operate a catalog ingestion pipeline (tiros) that processes product catalog data from affiliate networks (FMTC, etc.) into a unified format for downstream canonicalization and search indexing. The current implementation is Python-based and runs on Kubernetes.

### Current Pain Points

1. **Throughput ceiling**: Python's GIL limits parallel processing
2. **Memory pressure**: Large batch processing causes OOM in K8s pods
3. **Complexity**: Multiple scripts with overlapping concerns (insert, put, consumer)
4. **Operational burden**: Frequent manual intervention for failed jobs

### Performance Target

Process **2^14 (16,384) catalog items per partition** with:
- High throughput (saturate network/storage I/O)
- Low latency (sub-second item processing)
- Durability (S3-backed, 11 nines availability)
- Idempotency (content-hash change detection)
- Resumability (checkpoint-based pause/resume)

## The Experiment

**Hypothesis**: A Rust-based ingestion engine using tansu as the foundation can achieve 10x+ throughput while simplifying the operational model.

### Why Tansu?

[Tansu](https://github.com/tansu-io/tansu) is a Kafka-compatible streaming platform with:

1. **Kafka protocol compatibility**: Standard Kafka clients work out of the box
2. **S3-backed storage**: Native object storage, no local disk dependencies
3. **Delta Lake / Iceberg integration**: Table formats for analytics queries
4. **Embeddable**: Can run as a library, not just a standalone broker
5. **Modern Rust**: async/await, tokio runtime, strong type safety

### Approach: Surgical Fork

Rather than building from scratch, we fork tansu and make **minimal, modular changes**:

1. **Keep upstream compatibility**: main branch syncs with tansu-io/tansu
2. **Isolated customizations**: ptdev/main branch with our changes
3. **Progressive merges**: Regularly merge upstream improvements into our fork
4. **Modular crates**: Add new crates rather than modifying core tansu code when possible

## Repository Structure

```
tiros-engine/                    # Fork of tansu
├── .mise.toml                   # Tool versions (Rust 1.93, Python 3.12, uv, just)
├── pixi.toml                    # Reproducible environment (CI/dev)
├── pyproject.toml               # Python tooling via uv
├── AGENT.md                     # Agent instructions
├── activity-logs/               # Development logs (this file)
├── skills/                      # Claude Code skills
├── tansu-*/                     # Original tansu crates
└── [future: tiros-*/]           # Our custom crates
```

## Branch Strategy

```
upstream/main (tansu-io/tansu)
       │
       ▼ (sync)
origin/main (precisetargetlabs/tansu)
       │
       ▼ (merge)
ptdev/main (our customizations) ← default branch
       │
       ▼ (merge)
ptdev/feature-* (feature branches)
```

## Current State

- [x] Forked tansu to precisetargetlabs/tansu
- [x] Cloned into workspace/tiros-engine
- [x] Created ptdev/main branch (set as default)
- [x] Added mise/pixi/uv configuration
- [x] Added AGENT.md and activity-logs/
- [x] Verified cargo check passes (Rust 1.93, edition 2024)
- [ ] Files not yet committed (pending review)

## Next Steps (Agent Work Queue)

### Phase 1: Understand Tansu Internals

1. **Study tansu-storage**: Understand the storage abstraction layer
   - How does S3 backend work?
   - What's the partition/segment model?
   - Where would we hook in custom sharding?

2. **Study tansu-broker**: Understand the broker architecture
   - How are producers/consumers handled?
   - What's the message flow?
   - Where's the protocol handling?

3. **Study tansu-sans-io**: Understand the protocol layer
   - Kafka protocol parsing
   - Message serialization/deserialization

### Phase 2: Design Custom Ingestion

4. **Design shard key scheme**: Based on prior libtiros work
   - Shard key: (year_week_day, source, merchant_id, network_id, brand_id, category_id, price_range)
   - Partition key: source_id (xxhash % 256)
   - Max 2^14 records per partition

5. **Design S3 path layout**: Hive-style partitioning
   ```
   s3://bucket/source=fmtc/week=2026-W06/merchant=12345/network=1/category=500/price=50-100/
   ```

6. **Design producer interface**: For FMTC catalog items
   - Batch ingestion API
   - Content-hash deduplication
   - Checkpoint/resume support

### Phase 3: Implement Custom Crates

7. **Create tiros-model crate**: Catalog item types (port from libtiros)
   - FmtcCatalogItem
   - ShardKey, PartitionKey, PriceRange
   - CatalogBatch (SoA layout)

8. **Create tiros-storage crate**: Custom storage layer
   - Extend tansu-storage with sharded S3 backend
   - Parquet file format
   - Bloom filters for source_id lookup

9. **Create tiros-ingest crate**: Ingestion pipeline
   - FMTC API client
   - Transform pipeline (raw → unified)
   - Sharded writer

### Phase 4: Integration & Testing

10. **Integration tests**: End-to-end ingestion flow
11. **Benchmarks**: Compare with Python baseline
12. **Dockerize**: K8s-ready container image

## Related Work

- **libtiros** (tiros-rs branch): Earlier Rust library experiment with PyO3 bindings
  - Location: andromeda/tiros/tiros-rs
  - Status: Paused in favor of tansu-based approach
  - Reusable: Core types (FmtcCatalogItem, ShardKey, PriceRange) can be ported

- **brainstorming.md**: Original architecture document
  - Captured the Phase 1/Phase 2 vision
  - Phase 1: MPMC channels, no Kafka dependency
  - Phase 2: Tansu for Kafka compatibility (we're starting here)

## Success Criteria

1. **Functional**: Ingest FMTC catalog, store in S3 with correct sharding
2. **Performance**: 16K items/minute sustained throughput
3. **Reliability**: No OOM, graceful failure handling, resumable
4. **Operational**: Single binary, minimal configuration, observable (metrics/traces)

## Open Questions

1. Should we embed tansu or run it as a sidecar?
2. Do we need full Kafka compatibility or just the storage layer?
3. How do we handle schema evolution for catalog items?
4. What's the migration path from current Python pipeline?

---

*This experiment is exploratory. The goal is to validate the approach before committing to a full rewrite.*
