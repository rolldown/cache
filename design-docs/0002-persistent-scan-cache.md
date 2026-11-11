---
title: Persistent Scan Cache
status: proposal
created: 2026-11-04
implementation: []
---

# Persistent scan cache

Rolldown keeps a scan cache in memory. [`ScanStageCache`][cache] holds a
snapshot of the scanned module graph and the maps from module id to module
index. When files change, a partial scan re-scans only the changed modules and
[`ScanStageCache::merge`][merge] splices the result into the snapshot. The
cache lives in the `Bundler` and is lost when the process stops, so every start
[begins with a full scan][full-scan].

This design saves scan results to disk and restores them on the next start in
the same worktree. A restart then skips the saved scan work for modules whose
inputs did not change while Rolldown was stopped.

Main decision (`decided`, from the [M1 goal][m1]): keep the existing
module-level incremental model. Partial scans and the merge path stay as they
are; the saved data feeds into them.

Which scan data is saved is `open` and decided in [Serialization](#serialization).
It may be only a part of the snapshot, such as transform results keyed by
stable module id. The saved data may therefore differ from the in-memory
snapshot.

[cache]: https://github.com/rolldown/rolldown/blob/41f7dd782/crates/rolldown/src/types/scan_stage_cache.rs#L23
[merge]: https://github.com/rolldown/rolldown/blob/41f7dd782/crates/rolldown/src/types/scan_stage_cache.rs#L123
[full-scan]: https://github.com/rolldown/rolldown/blob/41f7dd782/crates/rolldown/src/bundler/impl_bundler_incremental_build.rs#L65-L68
[m1]: ../milestones.md#m1-persistent-scan-cache

## Scope

In scope:

- Reuse of saved scan results after a restart in the same worktree.
- Changes made while Rolldown was stopped. Detecting them is defined in
  [Restart Validation](0001-restart-validation.md).
- Saving a module's scan result in a reduced form, such as its transform
  output. A saved result is still reused or invalidated per module.

Out of scope:

- New cache boundaries inside the scan stage with their own keys and
  invalidation rules, such as a separate cache for `resolveId` results. See
  [M3](../milestones.md#m3-incremental-computation-granularity-and-correctness).
- Incremental work in the link and generate stages. See
  [M4](../milestones.md#m4-incremental-reuse-in-link-and-generate).
- Reuse across worktrees, project moves, or machines. See
  [M2](../milestones.md#m2-shared-scan-cache).
- Cross-platform and cross-project reuse. See [goals.md](../goals.md#non-goals).

This scope is M1's. Later milestones change this document in place where they
change its decisions, following the rules in
[AGENTS.md](../AGENTS.md#keeping-designs-current).

## Restart flow

## Storage and incremental writes

## Serialization

## Cache validity

## Open questions

## Verification
