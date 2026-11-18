# Milestones

Each feature links to its design doc and to the rolldown PRs that implement it.
Mark a feature `(in progress)` while work is open, and check it when all its
PRs have merged.

M2 builds on M1, and the two can be developed together. M1 and M3 can be
developed in parallel with shared design work. M3 depends on M1 being able to
save and restore the cache correctly.

## M1: Persistent scan cache

**Goal:** Reuse scan results after restarting in the same worktree, while
preserving the module-level incremental behavior of Rolldown's existing scan
cache. This milestone does not introduce independently cached resolve, load, or
transform steps, or incremental computation in later build stages.

**Done when:**

- Restarting Rolldown in the same worktree reuses valid cached scan results.
- Changes made while Rolldown was stopped are handled correctly without manual
  cache cleanup.
- After a local source change, rebuilding updates only the affected cache data
  without rewriting the entire disk cache.
- Damaged cache data, interrupted writes, and incompatible cache versions do not
  cause incorrect results, panics, or cache-related build failures.
- After restarting Rolldown, benchmark projects build faster with the saved cache
  than without it, even after counting the time spent checking, loading, and
  updating the cache.

**Deliverables:**

- [ ] **Persistent cache storage and incremental writes.** Define and implement
  the cache layout, stored data, and incremental write model for the original
  worktree. Cover interrupted writes, damaged data, and version compatibility.
  Note where M3's sharing requirements are expected to change the storage
  design. Design: [Storage and incremental writes](design-docs/0002-persistent-scan-cache.md#storage-and-incremental-writes).
- [ ] **Serialization and deserialization.** Save and restore the scan data and
  side-effect records needed to continue incremental builds. Choose which scan
  data to save based on performance measurements, while keeping build results
  correct. Design: [Serialization](design-docs/0002-persistent-scan-cache.md#serialization).
- [ ] **Scan dependency tracking, invalidation, and side effects.** Define and
  track the inputs needed to validate cached results after restarts, including
  changes made while Rolldown was stopped. Handle required side effects when
  cached computations are reused or invalidated, within the existing scan-stage
  incremental scope. Design: [Restart Validation](design-docs/0001-restart-validation.md),
  [Cache validity](design-docs/0002-persistent-scan-cache.md#cache-validity).

## M2: Plugin state and untracked inputs

**Goal:** Let more modules reuse saved results after a restart, including
modules whose hooks keep state that Rolldown cannot see, or depend on inputs
that Rolldown does not track.

**Done when:**

- A module reuses its saved result only when the state and the inputs that it
  depends on are the same as when the result was saved. Otherwise its hooks
  run again.
- After a restart, later hooks get the same data from a reused module as in a
  build without the cache.
- After a restart, benchmark projects build faster than with M1 alone, even
  after counting the time spent checking, loading, and updating the cache.

**Deliverables:**

- [ ] **State that a plugin keeps on a module.** Track the data that a hook
  attaches to its module for later hooks, so that a reused module gives the
  later hooks the same data.
- [ ] **State that crosses modules.** Track what a hook of one module reads
  from or does for other modules, so that reusing one module does not lose
  or break the result of another.
- [ ] **Inputs that Rolldown does not track yet.** Track the inputs that a
  module's result depends on and that Rolldown can observe by itself, other
  than file contents, such as which files exist.

## M3: Shared scan cache

**Goal:** Reuse valid scan results across working directories, worktrees, and
machines on the same platform for the same project, building on M1's persistent
cache.

**Done when:**

- Moving a project or building in another worktree reuses valid scan results
  without invalidating the entire cache because the root path differs.
- Multiple worktrees of the same project can build in parallel with correct
  results. Cache data is saved correctly in each worktree's project-local cache
  directory and the shared project-global cache directory.
- A project's cache can be uploaded and restored on another machine on the same
  platform, including in CI, and valid entries are reused.
- Shared cache use preserves M1's incremental write and safety guarantees.
- Benchmark projects build faster with a shared cache than without one, even
  after counting the time spent checking, loading, and updating the cache.

**Deliverables:**

- [ ] **Project-local and project-global storage.** Define and implement what is
  stored in each directory and how concurrent worktree builds safely share and
  update cache data.
- [ ] **Cache reuse across working directories.** Handle
  path-dependent data and input differences across worktrees and project moves,
  preserving reuse where results remain valid.
- [ ] **Same-platform cache transfer.** Support saving and restoring the project's
  cache across compatible machines, including through external CI cache tools.

- [ ] **Define the test plan.** Decide how to measure the build performance gains
  from sharing caches across worktrees. One possible scenario is a single
  project with three worktrees working on non-overlapping features, comparing
  build times with and without shared caching.

## M4: Incremental computation granularity and correctness

**Goal:** Make builds faster by caching plugin computations in the scan stage,
for Rust and JavaScript plugins, without losing correctness. Explore cache
granularity and hook cache keys. The cache key API and whether to cache single
hooks are not decided.

**Done when:**

- A changed plugin dependency never leads to a stale result. Running work again
  without need is acceptable. A dependency that other bundlers also cannot
  track may stay out of scope, with the reason written down.
- Builds are faster overall, counting the cost of tracking dependencies,
  computing cache keys, and using the cache. Prefer automatic decisions over
  user configuration, such as caching only inputs above a size limit.

**Deliverables:**

- [ ] **Cache granularity.** Compare caching whole modules, single steps or
  hooks, and whole modules with hook cache keys. Decide by measured build time
  and cache size.
- [ ] **Hook cache keys.** Explore whether a plugin hook declares its own cache
  key, and what the key holds, such as environment variables and plugin
  options that only the plugin knows. Also decide whether such a key can
  replace marking a call as not cacheable. Inputs that Rolldown can observe by
  itself are part of [M2](#m2-plugin-state-and-untracked-inputs).
- [ ] **Cache key computation.** Keep calls between JavaScript and Rust few when
  computing cache keys, for example with templates instead of callbacks.

## M5: Incremental reuse in link and generate

**Goal:** Reduce rebuild time by reusing unaffected work in the link and generate
stages, with persistent and shared caching as in M1 and M3. Use their
implementation and performance measurements to select the scope; full incremental execution of both
stages is not required.

**Done when:**

- Builds using the new reuse paths produce the same output and required behavior
  as builds without caching, including after changes that affect multiple modules.
- Computations that do not support caching still run normally and produce
  correct results.
- Benchmark projects rebuild faster than with M3 alone, even after counting the
  time spent checking, loading, and updating the new caches.

**Deliverables:**

- [ ] **Baseline and research.** Measure link and generate costs after M1 and M3 and
  investigate approaches to incremental reuse. Identify repeated work and the
  opportunities most likely to improve overall rebuild performance.
- [ ] **Reuse boundaries and validation.** Design candidate computation and cache
  boundaries, including dependencies, invalidation, and required side effects.
  Test the proposed approaches and use the results to decide what to implement.
- [ ] **Implementation and testing.** Implement the selected approach and check
  whether it improves performance compared with M3.

## Template

```markdown
## M<N>: Milestone name

**Goal:** what users get when this milestone ships.

**Done when:**

- An observable behavior that shows the milestone meets its goal.

**Deliverables:**

- [ ] Feature name. Design: [0001](design-docs/0001-feature-name.md). PRs: rolldown/rolldown#0000
```

