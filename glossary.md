# Glossary

Terms used in cache discussions and designs. These definitions do not decide
storage layout, record placement, or user-facing options.

| Term | Meaning |
| --- | --- |
| **Worktree** | An independent working directory of a project, with its own source files and configuration. No worktree is the baseline for another. |
| **Local cache** | Project-local cache storage associated with one worktree. Use **local** as the short name. |
| **Global cache** | Project-global cache storage available to worktrees of the same project. Use **global**, not **shared**, as the short name. It does not mean a cache across unrelated projects. |
| **Host** | The program that calls Rolldown and supplies its options, such as a meta-framework built on Rolldown, the Rolldown CLI, or code that calls the Rolldown API directly. |
| **Build dependency** | A file that is not a module in the graph but affects the result of every module, such as a config file or a lockfile. A change to any build dependency causes a cache miss for every module. |
| **Transform dependency** | A file that is not the module file itself but affects the result of one module, such as a file that a `load` or `transform` hook registers with `this.addWatchFile`. When it changes, Rolldown runs `load`, `transform`, parsing, and import resolution again for that module, both in a rebuild and after a restart. |
| **Assumed-unchanged path** | A path whose module files and transform dependencies Rolldown does not hash. It treats them as unchanged. The host declares it and must provide a build dependency that changes whenever these files change. |
| **Cache hit** | Rolldown finds a saved result for a module and reuses it. It does not run the hooks that the saved result replaces. |
| **Cache miss** | Rolldown finds no saved result for a module. It runs the hooks. It may save a new result. For example, it saves none when a hook returns `meta` that cannot be saved. See [0002 Caveats](design-docs/0002-persistent-scan-cache.md#caveats). |
| **Module meta** | Data that a plugin attaches to one module, for other hooks to read. JavaScript plugins and Rust plugins have separate module meta. |
| **Module task** | The work of the scan stage for one module: `load`, `transform`, parsing, and the `resolveId` or `resolveDynamicImport` call of each import of the module. |
| **Saved result** | What Rolldown saves for one module under key 1 and key 2 of the persistent scan cache. On a cache hit, Rolldown reuses it as a whole. See [0002](design-docs/0002-persistent-scan-cache.md#what-a-saved-result-holds). |
| **Cached hook** | A plugin hook that has the persistent scan cache on: it sets `cache: true`, or the host turns the cache on for it with `assumeCacheableHooks`. |
| **Creating resolution** | The resolution of a module id that reaches Rolldown's module loader first in a build. It creates the module and sets its first `meta` and `moduleSideEffects`. See [0003 Rules without the cache](design-docs/0003-module-meta-and-module-side-effects.md#rules-without-the-cache). |

Reuse limits are defined in [goals.md](goals.md). Open questions about local
and global storage are recorded in [scratchpad.md](scratchpad.md).
