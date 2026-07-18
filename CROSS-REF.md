# CROSS-REF — mycelium-std-error

Mycelium-internal dependencies only (steer handoff §6.1; external crates stay in Cargo
metadata). Pinned revs are the fixed (buildable) tips recorded by the Phase-B wave;
content hash = git tree hash of the pinned rev.

| Interface consumed | Repo | Pinned rev | Content hash | Notes |
|---|---|---|---|---|
| mycelium-core | https://github.com/tzervas/mycelium-core | `46d2515cbd86d2ae4d1365f4adcd2796737e9f0b` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-core` (see monorepo `docs/api-index/INDEX.md#mycelium-core`) |
| mycelium-std-core | https://github.com/tzervas/mycelium-std-core | `376762cc17853e1582684ececf9e760426bcfb0c` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-std-core` (see monorepo `docs/api-index/INDEX.md#mycelium-std-core`) |
| mycelium-std-recover | https://github.com/tzervas/mycelium-std-recover | `ad8787428c0d8c1eb2bf3a8cd6504cc39bca00bf` | tree `(tree hash: fetch dep rev locally to resolve)` | Rust API of `mycelium-std-recover` (see monorepo `docs/api-index/INDEX.md#mycelium-std-recover`) |

**Owning docs:** `docs/spec/stdlib/error.md` (slice in this repo) · RFC-0016.
**Source provenance:** extracted from `tzervas/mycelium` archive `aad96b7a…`; fixed by
the course-correction Phase B (workspace root, git pins, toolchain + supply-chain
replicas, CI v2). Full program record: monorepo
`docs/planning/course-correction-2026-07-18/PROGRAM.md`.
