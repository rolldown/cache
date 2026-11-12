# Vite

Which hooks of Vite's plugins have the persistent scan cache on, which do
not, and why. The values that Vite passes to Rolldown as the host are in
[0002 Example: Vite](../design-docs/0002-persistent-scan-cache.md#example-vite).

Vite's own plugins are JavaScript plugins in the Vite repository. Its native
plugins are Rust plugins in Rolldown (`crates/rolldown_plugin_vite_*`).

## Hooks with the cache on

| Plugin | Hooks | Notes |
| --- | --- | --- |
| `viteAliasPlugin` (native) | `resolveId` | |
| `viteJsonPlugin` (native) | `transform` | |
| `viteModulePreloadPolyfillPlugin` (native) | `resolveId`, `load` | |
| `viteLoadFallbackPlugin` (native) | `load` | It registers the file it reads. |
| `viteBuildImportAnalysisPlugin` (native) | `resolveId`, `load` | |
| `viteReactRefreshWrapperPlugin` (native) | `resolveId`, `transform` | |
| `viteResolvePlugin` (native) | `resolveId`, `load` | A call makes the module not cacheable when it sends a warning or a debug log to Vite's logger, or when it calls `finalizeBareSpecifier` or `finalizeOtherSpecifiers`. Rolldown cannot send these logs again, and the two callbacks return ids that depend on Vite's dependency optimizer. Vite passes the two callbacks only to dev without bundling, and the debug callback only with `DEBUG=vite:resolve-details`. |
| `viteDynamicImportVarsPlugin` (native) | `resolveId`, `load`, `transform` | A call to Vite's JavaScript resolver makes the module not cacheable: Rolldown cannot see its inputs. |
| `viteTransformPlugin` (native) | `transform` | It registers the tsconfig that it reads. |
| `vite:bundled-dev-client-imports` | `resolveId`, `load` | |
| `vite:html-inline-proxy` | `resolveId` | |
| `vite:css` | `load` | |
| `vite:wasm-helper` | `resolveId`, `transform` | |
| `vite:asset` | `resolveId` | |
| `vite:esbuild-banner-footer-compat` | `transform` | |

`viteResolvePlugin` and `vite:asset` read `package.json` files and check
whether files exist. This is the M1 caveat that files the resolver reads are
not tracked. See [Caveats](../design-docs/0002-persistent-scan-cache.md#caveats).

## Hooks with the cache off

| Plugin | Hook | Why |
| --- | --- | --- |
| `viteImportGlobPlugin` (native) | `transform` | The files in a globbed directory are not tracked ([rolldown/cache#14](https://github.com/rolldown/cache/issues/14)). In dev, `hotUpdate` reads what it stores. A hook filter makes only modules with `import.meta.glob` count. In dev, `moduleParsed` removes what it stored for a module that reused its saved result. |
| `vite:css` | `transform` | The PostCSS config is not tracked. |
| `vite:css-post` | `transform` | In build, it stores the CSS for `renderChunk`. |
| `vite:build-html` | `transform` | It stores the HTML for `generateBundle`, calls `this.load`, and assigns `moduleSideEffects`. |
| `vite:worker`, `vite:worker-import-meta-url` | `load`, `transform` | `generateBundle` emits the worker chunks that the hook stores. |
| `vite:asset` | `load` | In build, it stores URL data that later hooks read. It returns `moduleSideEffects`. |
| `vite:wasm-helper` | `load` | In SSR, it stores a random URL id that later hooks read. |
| `vite:asset-import-meta-url` | `transform` | In build, it stores a random URL id that later hooks read. When the file does not exist, it warns through Vite's logger and does not register the file. |

## Files that cover the installed packages

A package manager has two kinds of files that list packages:

- The lockfile in the project root, such as `package-lock.json`, `yarn.lock`
  or `bun.lock`. Users commit it. It says which versions to install, and it
  can change without an install, for example after `git pull`. It then does
  not match the installed packages.
- A file that the package manager writes when it installs packages, such as
  npm's `node_modules/.package-lock.json`, pnpm's
  `node_modules/.pnpm/lock.yaml`, or Yarn's `.pnp.cjs`. It changes only when
  the installed packages change.

Vite passes the file of the second kind as a build dependency. A change to
the installed packages then changes a key. This also covers the packages that
the config file imports by name, such as a plugin package: Vite does not
record those imports in `configFileDependencies`. The rule for when Vite also
sets `assumeUnchanged: ['**/node_modules/**']` is in
[0002 Example: Vite](../design-docs/0002-persistent-scan-cache.md#example-vite).
When Vite finds no file of the second kind, it turns the cache off. See
[Caveats](#caveats).

When the package manager uses a patches directory, as with patch-package, every
file in it is a build dependency. A patch changes files under `node_modules`
without changing the lockfile.

A tool can write files into `node_modules` without installing a package. Vite
does not cover this case. For example, `prisma generate` writes the Prisma
client to `node_modules/.prisma/client/`. The file that records the installed
packages does not change, so no build dependency changes. The generated files
match `**/node_modules/**`, so Rolldown does not hash them. When a build
bundles them, for example an SSR build with `ssr.noExternal: true`, the next
build reuses the saved results of the old files. The output then has the old
client. This does not meet guarantee 2 in
[What the host must guarantee](../design-docs/0002-persistent-scan-cache.md#what-the-host-must-guarantee).

With `configLoader: 'native'` on a runtime that cannot track the imports of
the config file, Vite turns the cache off and prints a warning. The files that
the config file imports must be build dependencies, and Vite cannot list them.

## The count of transformed modules

`viteReporterPlugin` prints "N modules transformed". It counts a module in
`transform`. It also counts a module in `moduleParsed` when the module reused
its saved result and so skipped `transform`. The number is then the same with
and without the cache, apart from the cases in [Caveats](#caveats).

## Caveats

### No file that records the installed packages

Vite turns the cache on only when it finds a file of the second kind in
[Files that cover the installed packages](#files-that-cover-the-installed-packages).
It uses the files of this kind in the list of its dependency optimizer. It
looks in the project root and in each parent directory up to the workspace
root that `searchForWorkspaceRoot` finds. When it finds none, it turns the
cache off and prints a warning.

Without such a file, no build dependency covers the packages that the config
file imports by name. A lockfile in the project root covers them only in part.
For example, a user switches to a branch whose `bun.lock` has a new version of
a plugin, and runs `vite build` before `bun install`. The build saves results
of the old plugin under a key with the new `bun.lock`. After `bun install`,
`bun.lock` does not change, so the next build reuses these results.

A file above the workspace root can belong to another project, and an install
in this project does not change it. For example, Vite builds
`/repo/packages/web` in a pnpm workspace whose root is `/repo`. It finds
`/repo/node_modules/.pnpm/lock.yaml`. It does not use a stray
`~/node_modules/.package-lock.json` above `/repo`.

So the cache is off in these cases:

- Bun, aube and nub: the list of the dependency optimizer has only their
  lockfile in the project root.
- Yarn with `nodeLinker: pnpm`: it writes neither
  `node_modules/.yarn-state.yml` nor `.pnp.cjs`.
- A project with only `package-lock.json` or `pnpm-lock.yaml` in its root.
- Deno: the list has no file of Deno.
- Rush: `searchForWorkspaceRoot` does not find the root of a Rush repository,
  so `.rush/temp/shrinkwrap-deps.json` is above the workspace root.

### A module that reuses a saved result after a failed task

`viteReporterPlugin` marks a module when its `transform` runs. In dev,
`viteImportGlobPlugin` does the same. In `moduleParsed`, a module without a
mark reused its saved result. Both plugins clear the marks in `buildStart`. A
module task that fails after `transform` leaves its mark. If the same module
then reuses a saved result before the next `buildStart`, the plugins take the
old mark as a sign that `transform` ran. A lazy compile in dev can do this: it
reads saved results and does not run `buildStart`.

For example, in dev with lazy compilation, `/src/lazy.js` is reached only
through a lazy entry. It has a saved result for content A, which has no glob.

1. The user changes `lazy.js` to content B, which has
   `import.meta.glob('./pages/*.js')` and an import of a file that does not
   exist. The browser loads the lazy entry. `transform` runs and marks
   `lazy.js`, and then the task fails. Rolldown does not watch `lazy.js` yet,
   because the compile failed.
2. The user changes `lazy.js` back to A and loads the lazy entry again.
   `lazy.js` reuses its saved result for A.
3. The reporter does not count `lazy.js`. The import-glob plugin keeps the
   matcher for `./pages/*.js`. When the user creates `/src/pages/new.js`, Vite
   sends an update for `lazy.js`. If `lazy.js` accepts no update, Vite reloads
   the page. Without the cache, nothing happens.

### The count of a module with a bytes module type

A `load` hook can return code with the module type `base64`, `binary` or
`dataurl`. The module then runs `transform`, and the reporter counts it. When
this module reuses its saved result, the reporter does not count it. In
`moduleParsed`, it cannot tell this module from a module that Rolldown read as
bytes, which never runs `transform`. The same holds when a `transform` hook
sets one of these module types.
