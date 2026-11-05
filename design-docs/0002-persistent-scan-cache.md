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

A saved scan result is reused only when every input that produced it is
unchanged. This section defines those inputs, how Rolldown compares them, and
how much of the cache a change invalidates. How Rolldown detects that a tracked
file changed is defined in [Restart Validation](0001-restart-validation.md).

### Tracked inputs

Rolldown does not see config files or `process.env`. It receives options that
are already evaluated, and plugins. Some inputs are therefore known only to the
host.

The **host** is the program that calls Rolldown and supplies its options. This
document uses Vite as the example host. The same rules apply to any other host,
such as a meta-framework built on Rolldown, the Rolldown CLI, or code that calls
the Rolldown API directly.

| Input | Provided by | How a change is detected | What a change invalidates | Status |
| --- | --- | --- | --- | --- |
| A module file in the graph | Rolldown | The content hash differs from the saved hash | That module | `decided` |
| A file that a plugin registers with `addWatchFile` ([`transform_dependencies`][td]) | Rolldown | The content hash differs from the saved hash | The modules that registered it | `decided` |
| Normalized options ([`NormalizedBundlerOptions`][nbo]) | Rolldown | The [options hash](#options-hash) differs from the saved hash | The whole cache | `proposed` |
| Cache key | Host | The string differs from the saved string | The whole cache | `proposed` |
| A build dependency | Host | The content hash differs from the saved hash | The whole cache | `proposed` |

File content hashes are compared as defined in
[Restart Validation](0001-restart-validation.md).

Seen from one module, its saved result is reused only when all of these hold:

1. The cache key is the same.
2. The options hash is the same.
3. No build dependency changed.
4. The module file did not change.
5. No file that the module registered with `addWatchFile` changed.

If 1, 2, or 3 fails, Rolldown discards the whole cache and runs a full scan.
If 4 or 5 fails, Rolldown scans that module again and keeps the rest.
For a file under an [`assumeUnchanged`](#host-inputs) path, Rolldown does not
check 4 and 5.

The first two rows of the table follow from the [M1 deliverables][m1].
`transform_dependencies` lives in memory today and is lost on restart, so it
must be saved with the cache. See [Serialization](#serialization).

The Rolldown version is also an input. It is handled with cache format versions
in [Storage and incremental writes](#storage-and-incremental-writes).

### Options hash

Status: `proposed`.

Rolldown computes one hash from [`NormalizedBundlerOptions`][nbo] and saves it
with the cache. On restart it computes the hash again and compares the two.

The hash is taken from the normalized options, not from the options as the user
wrote them. A change in how Rolldown fills in defaults then also changes the
hash.

Rules:

- **Values that can be serialized** are hashed. This covers strings, numbers,
  booleans, lists, and maps, for example `define`, `resolve`, `platform`, and
  `moduleTypes`.
- **Functions are skipped.** Examples are an `external` function and a
  `treeshake.moduleSideEffects` function. Rolldown cannot tell whether a
  function behaves the same as before. The host must cover it. See
  [Host inputs](#host-inputs).
- **Plugins** contribute only their names, in order. Plugin options and plugin
  code are not hashed. The host must cover them as well.

M1 hashes all serializable options, including options that only affect later
stages, such as output file names. Changing one of those discards the cache
without need. This is safe but wasteful. See [Future work](#future-work).

M2 is expected to change this rule. Options such as `cwd` hold absolute paths,
which differ between worktrees, so hashing them as they are would prevent
sharing.

### Host inputs

Status: `proposed`. Neither Rolldown nor Vite has these options today.

The host passes three values under `experimental.cache`:

```ts
experimental: {
  cache: {
    cacheKey?: string,
    buildDependencies?: string[],
    assumeUnchanged?: (string | RegExp)[],
  },
}
```

**`cacheKey`** is a string. Rolldown saves it with the cache. If the string
differs on restart, Rolldown discards the whole cache.

**`buildDependencies`** is a list of file paths. Rolldown saves the content
hash of each file. If any of them changed on restart, Rolldown discards the
whole cache. Rolldown does not follow their imports, so the host must list
every file.

**`assumeUnchanged`** is a list of path patterns. Rolldown does not validate a
tracked file on restart when its path matches. The host may set it only when
module paths are real paths, with symbolic links resolved. See
[guarantee 3](#what-the-host-must-guarantee). Rolldown ignores it when
`resolve.symlinks` is `false` and then validates every tracked file.

#### What the host must guarantee

Status: `decided`.

1. The cache key and the build dependencies together cover every input that
   Rolldown cannot see. These are the functions in the options, the plugin
   options, the plugin code, and the files and values that they read. For
   example, the host lists the config file and the files it imports as build
   dependencies, so a change to the source of a function in the config
   discards the cache.
2. `assumeUnchanged` is safe only because the build dependencies did not
   change. So when a file under an `assumeUnchanged` path changes, a build
   dependency must change too. For example, the host lists
   `node_modules/.pnpm/lock.yaml`, which pnpm writes when it installs packages.
3. When the host sets `assumeUnchanged`, module paths must be real paths, with
   symbolic links resolved. Otherwise a package from the same monorepo, which
   is linked into `node_modules`, matches a `node_modules` pattern, and
   Rolldown misses changes to it.

Rolldown cannot check these guarantees. When a host sets none of the three
values, Rolldown treats functions and plugins as unchanged, and the caller is
responsible for them.

#### Example: Vite

Status: `proposed`. This is one way for a host to meet the guarantees. Other
hosts may do it in another way.

| Vite passes | What it covers |
| --- | --- |
| `buildDependencies`: the config file, the files it imports, and the files in the user's `experimental.cache.buildDependencies` | A change to the content of these files, including the source of functions in the config |
| `buildDependencies`: the file that records the state of the installed packages, such as their versions, for example pnpm's `node_modules/.pnpm/lock.yaml` | A change to a package that the config file imports.<br>A change to a package that the user's code imports. |
| `cacheKey`: a hash of the normalized config, the Vite version, and the user's `experimental.cache.cacheKey` | A change to a normalized config value while the config file and the files it imports stay the same. CLI flags, inline config, mode, and environment variables that the config reads can cause such a change. |
| `assumeUnchanged`: `node_modules` | Nothing. It is an optimization: Rolldown does not validate the files under `node_modules` one by one. This is safe because the file that records the state of the installed packages is a build dependency. When a package is installed, updated, or removed, that file changes and Rolldown discards the whole cache. |

Vite hashes the normalized config with the same rules as the
[options hash](#options-hash), so functions are skipped. To detect a change to
a function, Vite lists the config file that holds it as a build dependency.

Vite sets `assumeUnchanged` only when it finds a file that records the state of
the installed packages, and `resolve.preserveSymlinks` is `false`.

Vite cannot see every input in guarantee 1. The user covers the rest:

- For a value that a function or a plugin hook reads when it runs, such as
  `process.env.THEME` inside a `transform` hook, the user puts it into
  `experimental.cache.cacheKey`. `process.env.NODE_ENV` and the variables with
  the `VITE_` prefix need no action, because they are part of the normalized
  config.
- For a file that the config reads without importing it, such as with
  `fs.readFileSync`, the user adds it to `experimental.cache.buildDependencies`.

### Open questions

1. **How does Vite cover a workspace package that the config file imports by
   its package name?** The way that Vite tracks the dependencies of the config
   file today does not track this case. So the build dependencies that Vite
   passes are not complete: a change to a plugin in the same monorepo that is
   imported by its package name is not detected. This needs support in
   `fresh-import`, or another way to find these files. Until then, the user
   must add the files to `experimental.cache.buildDependencies`.

### Future work

1. **Let the host add build dependencies after the build starts.** A config
   file can load another file with a dynamic import, and that import may run
   late. So the host knows the full list only when the build ends. An option
   that is set at the start cannot hold these files.
2. **Hash only the options that affect the scan.** M1 hashes all serializable
   options. A change to an option that only affects later stages, such as an
   output file name, discards the cache. Dev and build also cannot share a
   cache, because their options differ.
3. **Let a plugin declare the values it reads when it runs.** M1 does not track
   a value such as `process.env.THEME` that a plugin hook reads. The user must
   put it into the cache key. A plugin could declare such values by name, so
   that Rolldown tracks them. This is part of
   [M3](../milestones.md#m3-incremental-computation-granularity-and-correctness).

[td]: https://github.com/rolldown/rolldown/blob/41f7dd782/crates/rolldown_plugin/src/plugin_driver/mod.rs#L44
[nbo]: https://github.com/rolldown/rolldown/blob/41f7dd782/crates/rolldown_common/src/inner_bundler_options/types/normalized_bundler_options.rs#L44

## Future work

1. **Invalidate only the installed package that changed.** Today a change to
   an installed package discards the whole cache. Narrowing it needs three
   things:

   1. The cache uses the package name and version as the key for the modules
      of a package.
   2. The incremental build revokes the old package and resolves the import
      again from the importer. Rolldown must then record, on each import
      record, the paths that the resolver read or found missing.
   3. The watcher and Restart Validation check whether the file behind a
      symbolic link changed, not only the link path. With pnpm,
      `node_modules/react` keeps its path but points to another version after
      an update.

   Today Rolldown [watches only the real path][watch] that an import resolved
   to. It cannot see that a link changed, so it does not build an updated
   dependency correctly after `pnpm update`. It also does not resolve the
   importer again when a user file moves or a new file takes over an import.

[watch]: https://github.com/rolldown/rolldown/blob/41f7dd782/internal-docs/dev-engine/implementation.md

## Verification
