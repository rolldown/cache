# Milestones

Each feature links to its design doc and to the rolldown PRs that implement it.
Mark a feature `(in progress)` while work is open, and check it when all its
PRs have merged.

M1 and M2 can be developed in parallel with shared design work. M2 depends on
M1 being able to save and restore the cache correctly.

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
  Note where M2's sharing requirements are expected to change the storage
  design.
- [ ] **Serialization and deserialization.** Save and restore the scan data and
  side-effect records needed to continue incremental builds. Choose which scan
  data to save based on performance measurements, while keeping build results
  correct.
- [ ] **Scan dependency tracking, invalidation, and side effects.** Define and
  track the inputs needed to validate cached results after restarts, including
  changes made while Rolldown was stopped. Handle required side effects when
  cached computations are reused or invalidated, within the existing scan-stage
  incremental scope. Design: [Restart Validation](design-docs/0001-restart-validation.md).

## M2: Shared scan cache

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

## M3: Incremental computation granularity and correctness

**Goal:** Improve build performance through caching of plugin computations during
the scan stage, covering both Rust and JavaScript plugins. Explore dependency
tracking and cache granularity while preserving correctness. The cache key API
and whether to cache individual plugin hooks remain open design decisions.

**Done when:**

- When a plugin dependency changes, we must not reuse stale results. Recomputing
  unnecessarily is OK; missing a change is not. If other bundlers also struggle
  with certain dependencies, we can consider leaving those cases out of scope
  for now, but we should explain which ones and why.
- Caching plugin computations should make builds faster overall, including the
  cost of tracking dependencies, computing cache keys, and checking, reading,
  and writing the cache. Explore heuristics such as caching only when the input
  exceeds a size threshold, while keeping other options open. Prefer automatic
  decisions that don't require user configuration.

**Deliverables:**

- [ ] **Cache granularity.** Compare caching whole modules, caching individual
  steps or plugin hooks, and using plugin-provided cache keys while still
  caching whole modules. Use measurements to decide which approach works best,
  considering both build time and cache size.
- [ ] **Dependencies and cache key inputs.** Work out how plugins describe the
  inputs their results depend on, such as files, environment variables, and
  plugin options. Consider how meta-frameworks can supply additional cache key
  information, including whether Vite or Rolldown project config should expose
  a global `cacheKey`.
- [ ] **Cache key computation.** Keep communication between JavaScript and Rust
  low when computing cache keys. Explore options such as Handlebars-style
  templates instead of callbacks.

## M4: Incremental reuse in link and generate

**Goal:** Reduce rebuild time by reusing unaffected work in the link and generate
stages, with persistent and shared caching as in M1 and M2. Use their
implementation and performance measurements to select the scope; full incremental execution of both
stages is not required.

**Done when:**

- Builds using the new reuse paths produce the same output and required behavior
  as builds without caching, including after changes that affect multiple modules.
- Computations that do not support caching still run normally and produce
  correct results.
- Benchmark projects rebuild faster than with M2 alone, even after counting the
  time spent checking, loading, and updating the new caches.

**Deliverables:**

- [ ] **Baseline and research.** Measure link and generate costs after M1 and M2 and
  investigate approaches to incremental reuse. Identify repeated work and the
  opportunities most likely to improve overall rebuild performance.
- [ ] **Reuse boundaries and validation.** Design candidate computation and cache
  boundaries, including dependencies, invalidation, and required side effects.
  Test the proposed approaches and use the results to decide what to implement.
- [ ] **Implementation and testing.** Implement the selected approach and check
  whether it improves performance compared with M2.

## Template

```markdown
## M<N>: Milestone name

**Goal:** what users get when this milestone ships.

**Done when:**

- An observable behavior that shows the milestone meets its goal.

**Deliverables:**

- [ ] Feature name. Design: [0001](design-docs/0001-feature-name.md). PRs: rolldown/rolldown#0000
```

