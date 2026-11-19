# Vite

> **Note:** This document is mostly written and kept up to date by AI. Nobody
> has reviewed it in detail. Before you rely on a statement here, check it
> against the code.

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
| `viteResolvePlugin` (native) | `resolveId`, `load` | A call makes the module not cacheable when it sends a warning or a debug log to Vite's logger, or when it calls `finalizeBareSpecifier` or `finalizeOtherSpecifiers`. Rolldown cannot send these logs again, and the two callbacks return ids that depend on Vite's dependency optimizer. Vite passes the two callbacks only to dev without bundling, and the debug callback only with `DEBUG=vite:resolve-details`. See [Debug logs in cached hooks](#debug-logs-in-cached-hooks). |
| `viteDynamicImportVarsPlugin` (native) | `resolveId`, `load`, `transform` | A call to Vite's JavaScript resolver makes the module not cacheable: Rolldown cannot see its inputs. |
| `viteTransformPlugin` (native) | `transform` | It registers the tsconfig that it reads. |
| `viteReporterPlugin` (native) | `transform` | It sets a mark in the meta of the module. This needs [0003](../design-docs/0003-module-meta-and-module-side-effects.md), part of M2. See [The count of transformed modules](#the-count-of-transformed-modules). |
| `vite:bundled-dev-client-imports` | `resolveId`, `load` | |
| `vite:html-inline-proxy` | `resolveId` | |
| `vite:css` | `load`, `transform` | In some cases a `transform` call makes the module not cacheable. See [CSS](#css). |
| `vite:css-post` | `transform` | It returns the CSS of the module as module meta, which needs [0003](../design-docs/0003-module-meta-and-module-side-effects.md), part of M2. `renderChunk` reads the CSS from there. |
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
| `vite:build-html` | `transform` | It keeps the HTML and the code of inline scripts and styles in its own memory for later hooks. It calls `this.load` for the entry script of a `<script type="module" src>`, which gives no saved result (see [Caveats](../design-docs/0002-persistent-scan-cache.md#caveats)). It runs the `transformIndexHtml` hooks of other plugins, which Rolldown cannot see. |
| `vite:html-inline-proxy` | `load` | It reads the code that `vite:build-html` keeps in its memory. Only ids with a query reach it, and such ids get no saved result in M1. |
| `vite:worker-import-meta-url` | `transform` | It bundles the worker file in a nested build. The saved result of one module cannot stand for all modules of that build. See [Worker builds](#worker-builds). |
| `vite:worker` | `load`, `transform` | Only ids with a query reach them, and such ids get no saved result in M1. `load` also bundles the worker in a nested build. See [Worker builds](#worker-builds). |
| `vite:asset` | `load` | It uses an output asset file. See [Modules that use an output asset file](#modules-that-use-an-output-asset-file). |
| `vite:wasm-helper` | `load` | It uses an output asset file. See [Modules that use an output asset file](#modules-that-use-an-output-asset-file). |
| `vite:asset-import-meta-url` | `transform` | It uses an output asset file. See [Modules that use an output asset file](#modules-that-use-an-output-asset-file). It also calls Vite's JavaScript resolver, whose inputs Rolldown cannot see. |

## CSS

The `transform` hooks of `vite:css` and `vite:css-post` have the cache on. A
CSS file whose id is a plain file path, such as `/src/a.css`, can reuse its
saved result.

**The CSS text.** In build, `renderChunk` of `vite:css-post` needs the
compiled CSS of each module. The `transform` hook returns it as module meta,
under the key `vite:css-post`, and `renderChunk` reads it with
`this.getModuleInfo(id).meta`.

The key is the name of the plugin, not `vite`. A returned `meta` merges
only at the top level ([Hook results](https://github.com/rolldown/rolldown/blob/align-module-options-with-rollup/internal-docs/module-meta-and-side-effects/design.md#hook-results)). Other plugins set `meta.vite.cssScopeTo` for the same module, and
`renderChunk` reads it. If `vite:css-post` returned
`meta: { vite: { css } }`, it would remove `cssScopeTo`.

**Files in the public directory.** For `url(/logo.png)` with the file in the
public directory, the CSS text holds a placeholder until `renderChunk`. The
placeholder holds the URL itself, so `renderChunk` needs no data from the
skipped hook.

**The PostCSS config.** Vite loads the PostCSS config before the build
starts. The file that it finds is a build dependency. Its path, or the fact
that no file was found, is part of the cache key. So a change to the config
file, a new config file, and a config file in a nearer directory all cause a
miss for every module. This does not cover the files that the config
imports, and the files that a PostCSS plugin reads by itself. An example is
`.browserslistrc`, which autoprefixer reads. The user adds such a file to
`experimental.cache.buildDependencies`.

**Files that become transform dependencies.** `vite:css` registers these
files, so a change to one of them causes a miss for the module:

- the file of `composes: x from './other.module.css'`. With PostCSS, such a
  module gets no saved result. See the list below;
- a file in the public directory that a `url()` points to;
- a file in the public directory that lightningcss bundles for an `@import`.

**CSS modules that get no saved result.** `vite:css` and `vite:css-post`
mark a module as not cacheable when the hook sees one of these cases while it
runs. The result depends on something that Rolldown does not track, or the
hook does something that Rolldown cannot do again when it reuses the saved
result:

- The module uses an output asset file, for example `url(./logo.png)` with a
  file that is not inlined. See
  [Modules that use an output asset file](#modules-that-use-an-output-asset-file).
- The hook prints a message: a message to Vite's logger at any level, or a
  warning or a `@debug` message of Sass. Rolldown cannot print it again.
  With PostCSS, a CSS modules file with `composes ... from` another file
  always hits this rule: postcss-modules adds the CSS of the other file
  without the `from` option, and Vite warns about it. lightningcss does not
  cause this warning.
- The module uses a file that Vite already warned about in this build, such
  as a Git LFS placeholder file that library mode inlines. Vite prints the
  warning once for each file, so it marks every module that uses the file,
  not only the first one.
- `css.modules.getJSON` is set, and the file is a CSS modules file. The hook
  calls this function of the user for each such file.
- `DEBUG=vite:sourcemap` is set in dev. See
  [Debug logs in cached hooks](#debug-logs-in-cached-hooks).

The other modules are not affected. For example, `a.css` has
`url(./logo.png)` and `b.css` has no `url()`. `a.css` runs its hooks after
every restart. `b.css` reuses its saved result.

The CSS of an inline style in HTML and `?inline` CSS have ids with a query.
Such ids get no saved result in M1, so `vite:css-post` does not mark them.
A later milestone that saves results for such ids must mark them: the hook
keeps the CSS of an inline style in memory for `vite:build-html`, and it
prints the warnings of minifying this CSS.

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

## Debug logs in cached hooks

When the `DEBUG` environment variable turns on a namespace that prints inside
a hook with the cache on, Vite turns the cache off and prints a warning
(`resolveRolldownCacheOptions` in
`packages/vite/src/node/rolldownCache.ts`). Two namespaces do this:

- `vite:resolve-details`, in build and dev: `viteResolvePlugin` prints debug
  lines in `resolveId`.
- `vite:sourcemap`, in dev with `dev.sourcemap.css`: the `transform` hook of
  `vite:css-post` adds the source map to the code of a CSS module as a
  comment.

The hooks also mark the module as not cacheable in these cases. But a mark
only stops the saving. A run with the namespace on could still reuse a
result that a run without it saved. That module would then print no debug
lines, and its code would have no source map comment.

For example, a user runs `DEBUG=vite:resolve-details vite build` with
`experimental.cache: true`. Vite prints a warning that the cache is off.
Every hook runs, so every debug line is printed.

A user who sets `build.rolldownOptions.experimental.cache` directly gets no
such check. That user is the host, and `DEBUG` is one of the values in
guarantee 1 of
[What the host must guarantee](../design-docs/0002-persistent-scan-cache.md#what-the-host-must-guarantee).

## The count of transformed modules

`viteReporterPlugin` prints "N modules transformed" when a build ends. The
number must be the same with and without the cache. But `transform` does not
run for a module that reuses its saved result, so the plugin cannot count
the `transform` calls.

It uses [module meta](../design-docs/0003-module-meta-and-module-side-effects.md)
instead. In `transform`, the plugin sets a mark in the meta of the module.
Rolldown saves the mark with the saved result, and sets it again on reuse.
In `buildEnd`, the plugin counts the modules of the build that have the
mark.

The plugin also prints a progress line while the build runs. It counts a
module for this line in one of two hooks:

- In `transform`, when `transform` runs, as without the cache. This also
  counts the runtime module, for which `moduleParsed` does not run.
- In `moduleParsed`, when the module reused its saved result. The plugin
  counts the module only when the module has a mark that was set in this
  build, and `transform` did not count it in this build.

The meta of a module can also keep a mark from an earlier build in the same
process, such as in watch mode. Such a mark does not count. So the last
progress line shows the same number as the final count.

For example, after a restart with no changes, every module reuses its saved
result, and `transform` does not run at all. Rolldown sets the mark of each
module again from its saved result, so `moduleParsed` counts each module.
The last progress line and the final count show the same number as a build
without the cache.

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

### Old `import.meta.glob` patterns in dev

In dev, `viteImportGlobPlugin` keeps the glob patterns of each module. When
a file is created or deleted, the plugin compares its path with these
patterns. A module with a pattern that matches must run again, because its
glob now gives another list of files. So Vite sends an update for that
module, or reloads the page if the module accepts no update.

When a module no longer has a glob, the plugin removes its patterns. In one
case it does not: the module task failed after `transform`, and the module
then reuses a saved result in a lazy compile. The plugin then keeps a
pattern that the code of the module no longer has.

For example, `main.js` had `import.meta.glob('./pages/*.js')`. Now it has no
glob, but the plugin kept the pattern. A user creates `pages/new.js`. The
path matches the old pattern, so Vite updates `main.js` or reloads the page.
`main.js` does not use `pages/new.js`, so this update is not needed. The
build output is not affected.

### Worker builds

Vite bundles a worker file in a nested Rolldown build (`bundleWorkerEntry`
in `packages/vite/src/node/plugins/worker.ts`). Vite does not pass the cache
options to this build. So the modules of a worker run their hooks in every
build. The output is correct, but the build is not faster.

For example, `worker.js` imports `lib.js`, and `main.js` creates a worker
from `worker.js`. After a restart with no changes, `worker.js` and `lib.js`
run all their hooks again in the nested build.

To pass the cache options, Vite must first make the PostCSS config file of
the main config a build dependency and a part of the cache key for the
worker config. Vite creates a new config object for each worker, and it
loads the PostCSS config only for the main config. Also, each nested build
would open the cache directory once more.

### `DEBUG` with the cache on

Not decided. With the cache on, a namespace in
[Debug logs in cached hooks](#debug-logs-in-cached-hooks) turns the cache
off. Later, `DEBUG` with the cache on may mean "debug the behavior with the
cache on" instead, and keep the cache on.

For example, a user sees a problem only in a build with the cache on, and
sets `DEBUG=vite:resolve-details` to find it. Vite turns the cache off, so
this build does not show the behavior with the cache on.

### Modules that use an output asset file

An output asset file is a file that Vite copies to the output, such as an
image or a font that is too large to inline. A module that uses such a file
gets no saved result:

- an asset module, such as `import logo from './logo.png'`;
- a module with `new URL('./logo.png', import.meta.url)`;
- a `.wasm` module;
- a CSS module with `url(./logo.png)`.

A file that Vite inlines as a data URL is not an output file, so it does not
count.

The reason is how the file gets into the output. A hook must call
`this.emitFile` for it. Vite makes this call once for each file in a build:
the hook of the first module that uses the file makes the call, and Vite
keeps the result in a map. The hook of a later module that uses the same
file takes the result from the map and makes no call. Rolldown records an
`emitFile` call in the saved result of the module whose hook made it. So
the saved result of the later module would hold no call.

The example shows what would go wrong if these modules had saved results.
`a.css` and `b.css` both have `url(./logo.png)`.

1. In the first build, `a.css` is processed first, and its hook calls
   `emitFile`. The hook of `b.css` takes the result from the map.
2. A user removes the `url()` from `a.css`.
3. In the next build, `a.css` runs its hooks again and does not use
   `logo.png`. `b.css` reuses its saved result, which holds no `emitFile`
   call. No module emits `logo.png`, but the CSS of `b.css` still points at
   it, and the build fails.

Without the cache, the hook of `b.css` runs in step 3. The map is empty, so
this hook calls `emitFile` itself.

This is not data that Rolldown did not save. The result of `b.css` depends
on what the hook of `a.css` did, and Rolldown has no record of that. How to
give these modules a saved result is not decided. Three more points belong
to that decision:

- The result of `vite:asset` `load` depends on whether the module is an
  entry. This is not an input of a saved result.
- `vite:wasm-helper` compiles the `.wasm` file with the Node.js that runs
  Vite. The Node.js version is not part of the cache key.
- When one module emits a file again from its saved result, and the hook of
  another module emits the same file, the `names` list of the output asset
  holds the name two times.

### Paths that a CSS import did not find

Sass, Less, and the resolvers that Vite gives them try several paths for an
import, and use the first path where a file exists. Rolldown tracks the file
that was found. It does not track the paths that were tried before it. When
a user creates a file at one of these paths, the new file wins, but the
module that has the import did not change.

For example, `a.scss` has `@use 'theme'`. No `theme.scss` is next to
`a.scss`, so the import resolves to the package `theme` in `node_modules`.
A user creates `theme.scss` next to `a.scss`. Without the cache, the import
now resolves to the new file. With the cache, `a.scss` reuses its saved
result and keeps the styles of the package.

This is the M1 caveat that the files a resolver reads are not tracked. See
[Caveats](../design-docs/0002-persistent-scan-cache.md#caveats).

### Messages that a PostCSS plugin prints by itself

A PostCSS plugin reports a warning to PostCSS, and Vite prints it. A module
with such a warning gets no saved result. See [CSS](#css). But a plugin can
also print to the console directly, and Vite cannot see such a message. A
module that reuses its saved result does not print it again.

### Files that Vite does not watch

A CSS module can depend on a file that Vite does not watch, so the file is
not a transform dependency of the module. A rebuild without the cache does
not see a change to such a file: in `vite build --watch`, in bundled dev
and in dev, the output keeps the old content until a file that Vite watches
changes. The cache does the same: the module reuses its saved result after a
restart.

These files are:

- the files that Less reads by itself for `data-uri()`, `image-size()`,
  `image-width()`, `image-height()` and `@import (inline)`;
- a new file in a directory that a PostCSS plugin reports, or a new file
  that matches a glob that lightningcss reports;
- a file that a `url()` points to and that is created later;
- the files that Stylus finds and reads by itself, such as a new file that
  matches a glob `@import` or a file that `json()` reads.

For example, `style.less` has `width: image-width('./logo.png')`, and the
build gives `10px`. A user replaces `logo.png` with an image that is 20px
wide, and restarts. `style.less` did not change, so it reuses its saved
result, and the output still has `10px`. After a change to `style.less`,
the output has `20px`.

### Functions of the user in CSS options

Some CSS options take a function of the user, such as
`css.modules.generateScopedName`, `additionalData`, the `functions` and
`importers` of Sass, the `plugins` of Less, the `visitor` of lightningcss,
and PostCSS plugins. A module that reuses its saved result does not call
these functions. So a function must give the same result for the same
input, and must not keep state between calls. Only `css.modules.getJSON`
makes a module not cacheable, because it is called for its side effect.

For example, a function names classes with a counter: `c0`, `c1`, and so on.
The first build gives `.x` in `a.module.css` the name `c0`, and `.y` in
`b.module.css` the name `c1`. A user changes `b.module.css` and restarts.
`a.module.css` reuses its saved result and keeps `c0`. The counter starts
again at 0, so `.y` also gets `c0`, and the two rules apply to both
classes. Without the cache, a restart calls the function for every file
again, and a rebuild in the same process keeps the value of the counter. In
both cases no names are the same. The default names and a name pattern such as
`[name]__[local]___[hash:base64:5]` do not have this problem.
