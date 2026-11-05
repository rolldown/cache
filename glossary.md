# Glossary

Terms used in cache discussions and designs. These definitions do not decide
storage layout, record placement, or user-facing options.

| Term | Meaning |
| --- | --- |
| **Worktree** | An independent working directory of a project, with its own source files and configuration. No worktree is the baseline for another. |
| **Local cache** | Project-local cache storage associated with one worktree. Use **local** as the short name. |
| **Global cache** | Project-global cache storage available to worktrees of the same project. Use **global**, not **shared**, as the short name. It does not mean a cache across unrelated projects. |
| **Host** | The program that calls Rolldown and supplies its options. Vite is the usual example. A meta-framework built on Rolldown, the Rolldown CLI, and code that calls the Rolldown API directly are also hosts. |
| **Build dependency** | A file that is not a module in the graph but affects the result of every module, such as a config file or a lockfile. A change to any build dependency invalidates the whole cache. |
| **Assumed-unchanged path** | A path whose files Rolldown does not validate on restart. The host declares it and must provide a build dependency that changes whenever these files change. |

Reuse limits are defined in [goals.md](goals.md). Open questions about local
and global storage are recorded in [scratchpad.md](scratchpad.md).
