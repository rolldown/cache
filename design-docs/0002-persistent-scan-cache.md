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
module-level incremental model. [`ScanStageCache`][cache], partial scans, and
the merge path stay as they are. They serve rebuilds inside one process and
are not saved.

The persistent cache is separate from `ScanStageCache` (`decided`). It holds
the results of the plugin hooks for each module. The first scan after a
restart is still a full scan. For a module whose inputs did not change, it
reuses these results and does not run the hooks again.
[Serialization](#serialization) defines what is saved.

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

This section defines what Rolldown saves for each module.
[Cache validity](#cache-validity) defines when a saved result is reused.
[Storage and incremental writes](#storage-and-incremental-writes) defines how
saved results are found, written, and removed.

### What a restart skips

Status: `decided`.

A [module task][task] runs these steps for one module. After a restart, the
steps with a dashed border are skipped for a module whose saved result is
reused. The other steps run again.

```mermaid
flowchart LR
    L[load] --> T[transform] --> P[parse and AstScanner] --> R[resolveId for each import] --> M[moduleParsed]
    classDef skipped stroke-dasharray: 5 5
    class L,T,R skipped
```

- **`load`, `transform`, and `resolveId` are skipped.** These plugin hooks
  compute the result of a module. [Goal 1](../goals.md#goals) asks that a
  cache hit skips them.
- **Parsing and `AstScanner` run again.** So Rolldown saves no AST and no
  symbol table. The saved data is plain data, such as strings and paths.
- **`moduleParsed` runs again.** Plugins then see every module, as in a build
  without the cache.

A module has one saved result for the three skipped steps. It is reused as a
whole or not at all. [M1](../milestones.md#m1-persistent-scan-cache) does not
cache the three steps independently.

The `resolveId` call for an entry has no importing module, so no saved result
can hold it. It runs on every restart (`proposed`).

For example, `a.ts` imports `./b`, and a plugin transforms `a.ts`. After a
restart with no changes, Rolldown takes the transformed code of `a.ts` from
its saved result and parses it again. Parsing finds the import `./b`. Rolldown
takes the `resolveId` result for `./b` from the same saved result, and
continues with `b.ts`.

### What a saved result holds

Status: `decided`.

A value that a skipped step produces is saved only when something that is not
skipped reads it:

- a step that runs again, for example the parser reads the code that
  `transform` returns;
- a later stage, for example generate reads the sourcemaps;
- the module task of another module, for example the task of `b.ts` reads
  the `resolveId` result that the task of `a.ts` saved for `./b`.

For example, `load` returns code. Only `transform` reads it, and `transform`
is skipped too, so this code is not saved.

This rule gives a saved result three parts.

**State after `transform`.** It replaces `load` and `transform`. These are the
values that [`run_inner`][task] holds after `load_source` returns:

| Saved | Value in Rolldown |
| --- | --- |
| The code | `source: StrOrBytes` |
| The module type | `module_type: ModuleType` |
| The `moduleSideEffects` value from the hooks | `hook_side_effects: Option<HookSideEffects>` |
| The sourcemaps of `load` and of each `transform` | `sourcemap_chain: Vec<SourcemapChainElement>` |
| Whether Rolldown read the file from disk | `is_read_from_disk` in [`load_source`][load_source]. Rolldown reads it again in [Side effects](#side-effects). |

**Resolve results.** They replace `resolveId`, once for each import. The
result of `resolveId` is a [`ResolvedId`][resolved_id]. Rolldown saves these
fields for each import:

| Saved | Field |
| --- | --- |
| The resolved module id | `id` |
| The import attributes of the import | `attributes` |
| Whether it is external | `external`, `is_external_without_side_effects` |
| The path of its `package.json` | the path of `package_json` |
| The `moduleSideEffects` value from the hook | `side_effects` |
| Or: the module was not found | [`ResolveError::NotFound`][not_found] |

The other fields are not saved. Rolldown fills them in again from the saved
fields:

| Field | Filled in from |
| --- | --- |
| `package_json` | The saved path. Rolldown reads the file again. |
| `module_def_format` | The extension of `id` and the `type` field of `package_json`, with [`infer_module_def_format`][format]. |
| `normalize_external_id` | Not needed. Only `resolveId` itself reads it. |

**Side effects.** See [Side effects](#side-effects).

Two notes:

- **`package.json`** (`proposed`): only the path is saved. On a restart,
  Rolldown gives the path to [`try_get_package_json_or_create`][pkg], the same
  function that `resolveId` uses today. It reads the file once and shares the
  result across all modules through the resolver's cache. A change to the
  file's `type` or `sideEffects` then takes effect without a new saved
  result. The saved path still points at the `package.json` that `resolveId`
  found last time.
- **Paths** are saved as absolute paths. M2 is expected to change this,
  because absolute paths differ between worktrees.

### Side effects

Status: `decided`.

A skipped step can do more than return a value. Rolldown records every such
action that happens during the module task, whichever hook caused it, and
does it again when it reuses the saved result:

| Action in a skipped step | On reuse, Rolldown |
| --- | --- |
| A hook calls `this.addWatchFile` | Adds the file to the watched files. From `load` and `transform`, also to the inputs of the module. |
| A hook calls `this.emitFile` | Emits the same asset, chunk, or prebuilt chunk with the same arguments. The reference id stays the same, because Rolldown [computes it from the file's content or name][refid]. |
| A native plugin calls [`associate_module_with_file_ref`][assoc] | Links the module to the emitted file again. The `new URL()` finalizer reads this link. |
| A hook calls `this.warn`, `this.info`, or `this.debug` | Prints the same log. |
| Rolldown reads the module file from disk | Adds the file, and the tsconfig that governs it, to the watched files, as [`load_source`][load_source] does today. |
| `resolveId` finds no module | Gives the saved "not found" result to the code in [`resolve_dependencies`][not_found] that runs after resolution. It treats the import as external and prints the warning, as it does today. |

### Caveats

**Modules that get no saved result.** When a skipped hook of a module does
one of these, Rolldown saves no result for the module. Its hooks run on every
restart. The build result is correct, but the module gets no benefit from the
cache.

- **A hook returns a `meta` value that is not empty.** `meta` never reaches
  Rust. It lives only in a map on the JavaScript side
  ([`moduleOptionMap`][optmap]), and it can hold values that cannot be
  serialized. A later milestone is expected to use `meta` as the way for a
  plugin to keep state across restarts. See [Future work](#future-work).
- **A hook calls `this.load`.** The call runs the whole module task of
  another module and returns its `ModuleInfo`. The hook can use that
  information in its result, so the other module is an input that Rolldown
  does not track. See open question 3 in [Open questions](#open-questions).
- **A hook assigns to `moduleSideEffects` on an object from
  `this.getModuleInfo`.** The assignment is stored on the JavaScript side and
  applied to the other module after the scan
  ([`defer_sync_scan_data`][defer]). When the hook is skipped, the assignment
  is missing. Vite's `vite:build-html` plugin does this in its `transform`
  hook. See open question 4 in [Open questions](#open-questions).

**No cache with `experimental.nativeMagicString`.** When this option and
sourcemaps are both on, the sourcemap of a `transform` is
[added to the module after all modules are loaded][magic], not during the
module task. A result saved during the task would miss it. Rolldown does not
use the persistent cache in this mode.

**Inputs that M1 does not track.**

- `resolveId` is skipped, but the files that the resolver reads are not
  tracked. See item 1 of [Future work](#future-work-2).
- A user who writes `with { type: ... }` must turn on
  `experimental.importAttributes`. The attributes are then part of the module
  id. Without the option, the module type that the importer asserts is not an
  input of the saved result.
- A skipped hook can read values that Rolldown does not track, such as the
  result of `this.resolve`, the `ModuleInfo` of another module from
  `this.getModuleInfo`, or a file through `this.fs`. The host covers them. See
  [What the host must guarantee](#what-the-host-must-guarantee).

### Open questions

1. **How are the module options on the JavaScript side saved and restored?**
   This needs an answer before this design is accepted. `moduleSideEffects`
   and `meta` of each module live in [`moduleOptionMap`][optmap]. The
   JavaScript binding code writes a hook's return value into the map and
   passes the map's value to Rust ([`bindingifyLoad`][bind] and the other two
   hooks). Rust receives `moduleSideEffects` but not `meta`. This causes three
   problems:

   1. After a restart, the map is empty for a module whose saved result is
      reused. `this.getModuleInfo(id).moduleSideEffects` then differs from a
      build without the cache.
   2. Rust cannot tell a value that the module's own hooks returned from a
      value that came from the `resolveId` hook of the importing module. For
      example, the `resolveId` hook of `a.ts` returns `moduleSideEffects:
      false` for `./b`, and the hooks of `b.ts` return no value. The binding
      code then passes the map's value, `false`, to Rust after `load` and
      `transform` of `b.ts`, and the saved result of `b.ts` holds `false` as
      its own value. When the `resolveId` hook of `a.ts` later returns `true`
      and `b.ts` did not change, the saved result of `b.ts` is reused with
      the old `false`.
   3. Rolldown has no call from Rust to JavaScript that writes this map. The
      only bridge today runs the other way: after the scan, Rust asks
      JavaScript for the entries that a plugin changed through
      `moduleInfo.moduleSideEffects = v`
      ([`defer_sync_scan_data`][defer]), and it carries `moduleSideEffects`
      only.

2. **What does it cost to parse every module again?** The
   [M1 deliverable](../milestones.md#m1-persistent-scan-cache) asks that the
   choice of saved data rests on measurements. The decision in
   [What a restart skips](#what-a-restart-skips) needs a measurement of the
   steps that run again.

3. **Can a hook that calls `this.load` get a saved result later?** The call
   runs the whole module task of another module and returns its
   `ModuleInfo`, so the other module is an input of the calling hook.

4. **Can a hook that assigns to `moduleSideEffects` get a saved result
   later?** The assignment `this.getModuleInfo(id).moduleSideEffects = v`
   runs a setter on the JavaScript side. The setter writes `v` into
   [`moduleOptionMap`][optmap] for module `id` and marks the entry as changed.
   Rust does not see this. After the scan, Rust calls the JavaScript function
   `deferSyncScanData`, which returns every marked entry as
   `{ id, sideEffects }`, and [`defer_sync_scan_data`][defer] writes each
   value into the side effects of that module. So the assignment reaches Rust
   late, through one call for the whole scan, and only for marked entries.

5. **Can the persistent cache work with `experimental.nativeMagicString`?**
   The saved result would have to be written after the sourcemaps are added.

### Future work

1. **Let a plugin keep state across restarts through `meta`.** Today a plugin
   can keep state in its own memory, for example in `transform`, and read it
   in a hook that runs after the scan, such as `renderChunk` or
   `generateBundle`. After a restart with a reused result, that state is
   missing, and Rolldown cannot see it. The milestone for this is not decided.
   It is not M1, and the sooner the better. The expected design: a plugin
   turns the cache on for a hook. It then puts the state that
   `renderChunk`, `generateBundle`, and other later hooks need into `meta`, as
   values that can be serialized. Rolldown saves `meta` with the result and
   restores it on a restart. A hook that does not turn the cache on runs on
   every restart.

[task]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown/src/module_loader/module_task.rs#L99
[load_source]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown/src/module_loader/module_task.rs#L255
[resolved_id]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown_common/src/types/resolved_id.rs#L41
[not_found]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown/src/module_loader/resolve_utils.rs#L109
[format]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown_plugin/src/utils/resolve_id_with_plugins.rs#L21
[pkg]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown_resolver/src/resolver.rs#L113
[refid]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown_common/src/file_emitter.rs#L279
[assoc]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown_common/src/file_emitter.rs#L468
[optmap]: https://github.com/rolldown/rolldown/blob/2842dcd7c/packages/rolldown/src/plugin/plugin-context-data.ts#L47
[defer]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown/src/module_loader/deferred_scan_data.rs#L10
[magic]: https://github.com/rolldown/rolldown/blob/2842dcd7c/crates/rolldown/src/stages/scan_stage.rs#L241
[bind]: https://github.com/rolldown/rolldown/blob/2842dcd7c/packages/rolldown/src/plugin/bindingify-build-hooks.ts#L280

## Cache validity

A saved scan result is reused only when every input that produced it is
unchanged. This section defines those inputs, how Rolldown compares them, and
how much of the cache a change invalidates. How Rolldown detects that a tracked
file changed is defined in [Restart Validation](0001-restart-validation.md).

### Tracked inputs

Rolldown does not see config files or `process.env`. It receives options that
are already evaluated, and plugins. Some inputs are therefore known only to the
host.

See the glossary for the definition of [host](../glossary.md). This document
uses Vite as the example host. The same rules apply to any other host.

| Input | Provided by | How a change is detected | What a change invalidates | Status |
| --- | --- | --- | --- | --- |
| A module file in the graph | Rolldown | The content hash differs from the saved hash | That module | `decided` |
| A file that a plugin registers with `addWatchFile` ([`transform_dependencies`][td]) | Rolldown | The content hash differs from the saved hash | The modules that registered it | `decided` |
| Normalized options ([`NormalizedBundlerOptions`][nbo]) | Rolldown | The [options hash](#options-hash) differs from the saved hash | The whole cache | `proposed` |
| Cache key | Host | The string differs from the saved string | The whole cache | `proposed` |
| A [build dependency](../glossary.md) | Host | The content hash differs from the saved hash | The whole cache | `proposed` |

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
without need. This is safe but wasteful. See [Future work](#future-work-1).

M2 is expected to change this rule. Options such as `cwd` hold absolute paths,
which differ between worktrees, so hashing them as they are would prevent
sharing.

### Host inputs

Status: `proposed`. Neither Rolldown nor Vite has these options today.

The host passes these values under `experimental.cache`:

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

Rolldown cannot check these guarantees. When a host sets none of `cacheKey`,
`buildDependencies`, and `assumeUnchanged`, Rolldown treats functions and
plugins as unchanged, and the caller is responsible for them.

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
2. **Hash only the options that affect the scan.** Today a change to an option
   that only affects later stages, such as an output file name, discards the
   cache. See [Options hash](#options-hash).
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
   to. So it does not build an updated dependency correctly after
   `pnpm update`, and it does not resolve the importer again when a user file
   moves or a new file takes over an import. This defect is already in
   Rolldown's incremental build. A restart repairs it today, because a restart
   runs a full scan. With the cache, a restart reuses the saved result, so it
   no longer repairs it.

[watch]: https://github.com/rolldown/rolldown/blob/41f7dd782/internal-docs/dev-engine/implementation.md

## Verification
