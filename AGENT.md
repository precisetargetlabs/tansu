# AGENT.md - tiros-engine

> **For autonomous agents**: Read `activity-logs/2026-02-06-experiment-kickoff.md` for full context.

## TL;DR

This is a **fork of tansu** (Kafka-compatible streaming platform) that we're customizing for high-performance catalog ingestion. The goal is to replace a Python-based pipeline with a Rust implementation that can process 16K items/minute with S3-backed durable storage.

## Quick Start

```bash
# Ensure tools are available
mise install   # or: pixi install

# Verify build
cargo check --workspace
cargo test --workspace

# Run broker (for reference)
cargo run --release --bin tansu-broker
```

## Current Status

- [x] Fork cloned and configured
- [x] ptdev/main is the default branch
- [x] mise/pixi/uv environment configured
- [ ] Initial commit pending
- [ ] Phase 1 exploration not started

## Agent Work Queue

Work through these in order. Each task should result in:
1. Code changes or new files
2. Activity log entry documenting findings
3. Commit with descriptive message

### Immediate: Commit Setup

```bash
git add .mise.toml pixi.toml pyproject.toml AGENT.md activity-logs skills .gitignore
git commit -m "chore: add dev environment and agent configuration"
```

### Phase 1: Understand Tansu (Research)

**Goal**: Document how tansu works internally so we know where to hook in.

1. **Explore tansu-storage** (`tansu-storage/src/`)
   - Document the storage trait hierarchy
   - Understand S3 backend implementation
   - Identify extension points for custom sharding
   - Output: `activity-logs/tansu-storage-analysis.md`

2. **Explore tansu-broker** (`tansu-broker/src/`)
   - Document the broker architecture
   - Trace a produce request end-to-end
   - Understand partition assignment
   - Output: `activity-logs/tansu-broker-analysis.md`

3. **Explore tansu-model** (`tansu-model/src/`)
   - Document core data types
   - Understand message/record formats
   - Output: `activity-logs/tansu-model-analysis.md`

### Phase 2: Design (Planning)

4. **Design shard key scheme**
   - Based on: (year_week_day, source, merchant_id, network_id, brand_id, category_id, price_range)
   - Document in: `docs/shard-key-design.md`

5. **Design S3 path layout**
   - Hive-style partitioning compatible with Delta/Iceberg
   - Document in: `docs/s3-layout-design.md`

6. **Design catalog item schema**
   - Port FmtcCatalogItem from libtiros work
   - Document in: `docs/catalog-schema-design.md`

### Phase 3: Implement (Coding)

7. **Create tiros-model crate**
   - Add to workspace in Cargo.toml
   - Port types from libtiros (catalog.rs, shard.rs)
   - No tansu dependencies - pure Rust types

8. **Create tiros-storage crate**
   - Extend tansu-storage traits
   - Implement sharded S3 backend
   - Parquet file format

9. **Create tiros-ingest crate**
   - FMTC API client
   - Transform pipeline
   - Batch producer

## Branch Strategy

```
upstream/main (tansu-io/tansu)
       │
       ▼ (fetch + merge --ff-only)
origin/main
       │
       ▼ (merge)
ptdev/main (our work goes here)
```

### Syncing with Upstream

```bash
git fetch upstream
git checkout main
git merge upstream/main --ff-only
git push origin main
git checkout ptdev/main
git merge main -m "Merge upstream changes"
```

## Commit Guidelines

- Prefix with type: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`
- Reference activity log entries when relevant
- Keep commits atomic and focused

## File Conventions

- `activity-logs/YYYY-MM-DD-topic.md` - Daily/topic logs
- `docs/*.md` - Design documents
- `skills/*.md` - Reusable Claude Code skills
- `tiros-*/` - Our custom crates (when created)

## Key Files to Study

| File | Purpose |
|------|---------|
| `tansu-storage/src/lib.rs` | Storage trait definitions |
| `tansu-storage/src/s3.rs` | S3 backend (if exists) |
| `tansu-broker/src/main.rs` | Broker entry point |
| `tansu-model/src/lib.rs` | Core data types |
| `tansu-sans-io/src/` | Protocol parsing |

## Related Resources

- **libtiros** (andromeda/tiros/tiros-rs): Earlier experiment with PyO3 bindings
- **brainstorming.md** (was in docs/): Original architecture vision
- **tansu docs**: https://tansu.io/docs

## Success Metrics

1. Can ingest FMTC catalog items
2. Stores in S3 with correct shard/partition layout
3. 16K items/minute sustained throughput
4. Single binary, minimal dependencies

## Questions?

If blocked or uncertain:
1. Document the blocker in activity-logs
2. List options considered
3. Make a reasonable choice and proceed
4. Flag for human review in the log entry
