---
title: Restart Validation
status: proposal
created: 2026-10-31
implementation: []
---

# Restart Validation

## Why this is needed

A persistent cache survives a process restart. A file watcher attached to the
process does not observe changes while that process is stopped. Before reusing
cached work, Rolldown needs to check whether its inputs still match.

For example, a plugin transforms `a.ts` using `theme.json`:

```javascript
transform(code, id) {
  this.addWatchFile("theme.json");
  return convert(code, readTheme());
}
```

If `theme.json` changes while Rolldown is stopped, the saved result may be stale
even though `a.ts` has not changed.

Restart Validation is how Rolldown detects such a change.

## Proposed approach

Use content hashes to detect file changes across restarts.

The state of a tracked file is one of two values:

- "missing", if no file exists at the path;
- the content hash of the file, if it exists.

Rolldown reads the file to get its state. Check each file once, even if
multiple modules depend on it.

The persistent scan cache uses the state as an input of its keys. See
[Keys](0002-persistent-scan-cache.md#keys). A file that was created, deleted,
or changed has another state. The key then changes, and the saved result is a
cache miss. Rolldown saves no state to compare it later.

## Research: bundlers and task executors

These observations describe the inspected implementations, not guarantees for
every release or file type. Bazel is included as a build-system reference.

| System | Restart validation | Tradeoff |
| --- | --- | --- |
| webpack | Dependency snapshots. Module snapshots default to timestamps in development and timestamps plus hashes in production. Uses a safety window for timestamp precision. | Passing timestamp checks avoid content reads, but rely on timestamp behavior. |
| Turbopack | File reads are session-dependent tasks. A new session rereads files and compares results against persisted hashes. | Reads inputs again; unchanged results can avoid downstream recomputation. |
| Parcel v2 | Saves a watcher snapshot and queries changes since that snapshot. FSEvents and Watchman provide historical queries; traversal backends compare paths and mtimes. | Cost and detection behavior depend on the backend. |
| Parcel v3 branch | The inspected startup path creates new build state and supports in-process incremental updates. No persistent build-cache restoration path was found. | No restart-validation strategy confirmed in the inspected revision. |
| Bazel | Obtains input digests and uses them to validate action-cache entries. Existing digests may be available; otherwise it reads files to compute them. | Fast digest access requires support that ordinary local files may not have. |

### Sources

- **webpack:** [snapshot defaults](https://github.com/webpack/webpack/blob/main/lib/config/defaults.js), [`applyMtime()` and validation](https://github.com/webpack/webpack/blob/main/lib/fs/FileSystemInfo.js).
- **Turbopack:** [session-dependent reads](https://github.com/vercel/next.js/blob/cfa2956c852b226de09213513196510165def4ea/turbopack/crates/turbo-tasks-fs/src/disk.rs#L881-L910),  [`FileContent` persistence](https://github.com/vercel/next.js/blob/cfa2956c852b226de09213513196510165def4ea/turbopack/crates/turbo-tasks-fs/src/content.rs#L77-L96).
- **Parcel v2:** [`loadRequestGraph()`](https://github.com/parcel-bundler/parcel/blob/v2/packages/core/core/src/RequestTracker.js#L1443-L1483), [watcher backends](https://github.com/parcel-bundler/watcher#querying), [directory comparison](https://github.com/parcel-bundler/watcher/blob/master/src/DirTree.cc#L122-L141).
- **Parcel v3:** [startup and watch loop](https://github.com/parcel-bundler/parcel/blob/d290a66d4252cc1df81bf620ba852ed63a69a957/crates/parcel/src/lib.rs#L46-L84).
- **Bazel:** [digest acquisition](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/vfs/DigestUtils.java#L152-L190), [`ActionCacheChecker::isUpToDate()`](https://github.com/bazelbuild/bazel/blob/master/src/main/java/com/google/devtools/build/lib/actions/ActionCacheChecker.java#L197-L256).

### Timestamp safety windows

webpack uses `safeTime = mtime + estimated precision` to account for changes that may share the same timestamp. It trusts a matching timestamp only if `safeTime` is no later than the previous build’s start time.

This still relies on a sufficient precision estimate and does not detect content changes that preserve an old timestamp. [Source](https://github.com/webpack/webpack/blob/main/lib/fs/FileSystemInfo.js).

## Open questions

1. How should we handle a file changing between a plugin reading it and Rolldown hashing it? We defer this edge case for now.
2. The hash algorithm, validation timing, and whether to track file metadata remain open.

## Future work

- Directories. `this.addWatchFile` accepts a directory, and the file watcher watches everything below it. This design covers files only. How a directory is validated after a restart is not decided. See [rolldown/cache#14](https://github.com/rolldown/cache/issues/14) for the related `import.meta.glob` case.
- A timestamp check that skips hashing when `mtime` and `size` match, with webpack's `safeTime` rule for timestamp precision. It must be opt-in, because some writes keep the old `mtime` (verification case 5). Turning it on by default needs a way to show why a file or module was reused or not, for example a `cache inspect` command.
- Platform-specific optimizations, such as using a saved FSEvents event ID on macOS to query changes since the previous run. This needs a fallback for when event history is unavailable or incomplete.

## Verification

1. Unchanged inputs: Save a cache and restart without editing any tracked files. Expect a cache hit for every saved result.
2. Changed contents: Edit `a.ts` or its registered dependency `theme.json` while stopped. Expect a cache miss for `a.ts`.
3. Deleted files: Delete a tracked `theme.json` while stopped. Expect a cache miss for `a.ts`.
4. Previously missing files: Build `a.ts` with `theme.json` registered as a dependency but missing. Create `theme.json` while stopped, then restart. Expect a cache miss for `a.ts`.
5. Preserved `mtime`: Change `theme.json` contents, restore its original `mtime`, and restart. Expect a cache miss for `a.ts`.
6. Shared files: Register `theme.json` as a dependency of both `a.ts` and `b.ts`. Change it while stopped. Expect that Rolldown reads it once, and a cache miss for both `a.ts` and `b.ts`.
7. Changes during the build: Change `theme.json` after the plugin reads it but before Rolldown hashes it, for example from a test plugin hook that rewrites the file after reading it. Restart without further edits. The key of the saved result already holds the hash of the new contents, so the lookup alone gives a cache hit. Use this case to verify the policy for open question 1 once it is defined. A build using the saved cache must not reuse the result made from the old contents.
8. Build correctness: After editing `theme.json`, compare the output of a build using the saved cache with a clean build using the same inputs. Expect matching output.
9. Performance: Measure restart builds with no edits and with one edited file. Include validation, restoration, and cache-update costs, and compare total time with a build without saved cache.

The existing [M1 acceptance criteria](../milestones.md#m1-persistent-scan-cache) remain the delivery requirements.
