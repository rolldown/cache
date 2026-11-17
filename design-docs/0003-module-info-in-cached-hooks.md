---
title: Module Info in Cached Hooks
status: proposal
created: 2026-11-14
implementation:
  - https://github.com/rolldown/rolldown/pull/11671
---

# Module info in cached hooks

A JavaScript plugin can read and change the `meta` and `moduleSideEffects`
of a module through the object that `this.getModuleInfo` returns, or through
a `this.resolve` result. The module tasks of many modules share these values.
A saved result of the [persistent scan cache](0002-persistent-scan-cache.md)
belongs to one module task. Rolldown reuses it when the inputs of that
module did not change. A value that another module task set is not one of
these inputs.

So in a hook that has the cache on, Rolldown saves only the `meta` and
`moduleSideEffects` that the hook returns. When the hook uses these values in
another way, the module of the hook call gets no saved result. Rolldown does
not record the changes that a hook makes through these objects. Without a
JavaScript `Proxy`, Rolldown cannot see a write of the value that a key
already has. A `Proxy` gives the plugin another object than the meta object
of the module, and `structuredClone` throws for it.

This document covers these uses. How Rolldown saves a returned value, and
which values it can save, is in
[0002 Module meta](0002-persistent-scan-cache.md#module-meta).

## Contract

### Meta

A hook can read or change the meta of a module through the object that
`this.getModuleInfo` returns, or through the `meta` of a `this.resolve`
result. Rolldown does not save such a change. In a hook that has the cache
on, a read or a write of `.meta` on such an object gives the module of the
hook call no saved result. This holds for every module, also for the module
of the hook call. Rolldown then prints the warning
`PERSISTENT_CACHE_MODULE_INFO`. See [Warnings](#warnings).

Only a use of the `meta` property counts. A call of `this.getModuleInfo`
alone, or a read of another property such as `importers`, does not count. A
`resolveId` hook that returns a `this.resolve` result as it is does not count
either: the `meta` of that result is the meta object of the module, so the
hook sets no meta. For `moduleSideEffects`, see below.

A hook that returns the meta object of another module counts as a use. For
example, it changes the `id` of a `this.resolve` result and returns it, or it
returns one object as the meta of two modules.

For example, a `transform` hook can set meta on `a.ts` in two ways:

```js
// No saved result for a.ts, and a warning.
handler(code, id) {
  this.getModuleInfo(id).meta.icons = findIcons(code)
  return code
}

// A saved result for a.ts.
handler(code, id) {
  return { code, meta: { icons: findIcons(code) } }
}
```

With the first hook, the hooks of `a.ts` run again after every restart, and
the meta is set each time. With the second hook, `a.ts` reuses its saved
result, and Rolldown puts the meta back.

### `moduleSideEffects`

In a hook that has the cache on, only a `moduleSideEffects` value that the
hook returns is saved. A hook can also use the value in other ways. Then the
module of the hook call gets no saved result.

The plugin can write the hook in another way for these uses, so Rolldown
prints the warning `PERSISTENT_CACHE_MODULE_INFO`:

- The hook reads or assigns `moduleSideEffects` on the object that
  `this.getModuleInfo` returns. This holds for every module, also for the
  module of the hook call.
- The hook reads `moduleSideEffects` from a `this.resolve` result, also when
  no hook set the value.

For these uses, the hook passes on a value that another hook set. This is
the usual way to write a hook, and the plugin cannot know which value to
return instead. So Rolldown prints no warning:

- The hook returns a `this.resolve` result with a value that a hook set.
- The hook returns an object without `moduleSideEffects`, and Rolldown passes
  on the value that another hook set for the module. For a JavaScript
  `resolveId`, `load` or `transform` hook that returns an object, Rolldown
  uses the value that the module has on the JavaScript side. When a hook
  returns a string or nothing, Rolldown passes on no value. For a `load` or
  `transform` hook, a value that an earlier `load` or `transform` hook of the
  same module returned does not count, because the saved result already
  holds it.

For example, a `transform` hook assigns the value on other modules:

```js
{
  name: 'html',
  transform: {
    cache: true,
    handler(code, id) {
      if (id.endsWith('.html')) {
        for (const script of findScripts(code)) {
          const info = this.getModuleInfo(script)
          if (info) info.moduleSideEffects = true
        }
        return toJs(code)
      }
    },
  },
}
```

`index.html` has a script tag for `main.js`. The hook of `index.html` assigns
`true` on `main.js`, so that tree shaking keeps `main.js`. Rolldown saves no
result for `index.html`, and prints one warning. After a restart, the hook of
`index.html` runs again and assigns `true` again. The output is the same as
in a build without the cache.

Another example, for a value that Rolldown passes on: a JavaScript
`resolveId` hook returns `moduleSideEffects: false` for `./b`. The
`transform` hook of `b.ts` has the cache on and returns `{ code }`. Rolldown
passes on `false`, which the `resolveId` hook set. So `b.ts` gets no saved
result, and Rolldown prints no warning.

### Warnings

For each use of `meta`, and for each use of `moduleSideEffects` in the first
list above, Rolldown prints the warning `PERSISTENT_CACHE_MODULE_INFO`. The
message names the plugin and the hook,
the module that gets no saved result, and the module whose value the hook
used. For a hook that runs inside a `this.resolve` call, the module that gets
no saved result is the module of the hook that called `this.resolve`. The
message also says how to keep the saved result. Rolldown prints the warning
once for each plugin and hook, and the message says that other modules can
have the same problem. What "once" means is in **Warnings** of
[0002 Module meta](0002-persistent-scan-cache.md#module-meta).

## Caveats

**A spread of a `this.resolve` result counts as a use of `meta`.** A spread
`{ ...r }`, a rest pattern `const { id, ...rest } = r`, and
`JSON.stringify(r)` all read `r.meta`. In a hook that has the cache on, the
module then gets no saved result, also when the hook does nothing with the
meta.

```js
resolveId: {
  cache: true,
  async handler(source, importer) {
    const r = await this.resolve(source, importer, { skipSelf: true })
    // Reads r.meta: the importing module gets no saved result.
    return r && { ...r, external: false }
  },
}
```

A hook that returns `r` as it is, or that builds a new object from single
properties such as `{ id: r.id, external: false }`, does not read `r.meta`.

**A use of `meta` after the hook call ended.** Rolldown sees a use of `.meta`
only before the hook call ends. A plugin can keep the object and use it
later. Rolldown does not see that use, and the module keeps its saved
result. Then a change that the plugin makes later is missing after a
restart.

```js
transform: {
  cache: true,
  handler(code, id) {
    const info = this.getModuleInfo(id)
    // Runs after the hook call ended. Rolldown does not see it.
    queueMicrotask(() => { info.meta.icons = findIcons(code) })
    return code
  },
}
```

Without the cache, `renderChunk` reads `meta.icons` of `a.ts`. After a
restart, `a.ts` reuses its saved result, `transform` does not run, and
`meta.icons` is not set.

**A value from another module task while a hook of the module runs.**
Another module task can give a module a `meta` or `moduleSideEffects` value
while a hook of that module still runs: an assignment through
`this.getModuleInfo`, or a value that a `resolveId` hook returns for the
module. The final value then depends on which write comes last, also
without the cache. With the cache, Rolldown writes a saved value of the
other task before a hook of the module runs. Then the module can save its
result with the other value, and later builds reuse it.

For example, the `load` hook of `x.js` returns `moduleSideEffects: false`.
While the `transform` hook of `x.js` waits, a `resolveId` hook of `y.js`
returns `true` for `x.js`. Without the cache, `true` comes last. After a
restart, `y.js` reuses its saved result, so `true` comes first and `false`
comes last. `x.js` saves `false`, and later builds keep it.

**The number of reused modules can depend on the order of work.** When
Rolldown passes on a `moduleSideEffects` value that another hook set, the
module gets no saved result. See [`moduleSideEffects`](#modulesideeffects).
Whether such a value is there can depend on which module task ran first. The
output is still correct.

For example, `a.ts` and `c.ts` both import `./b`. The `transform` hook of
`b.ts` returns `moduleSideEffects: false`. A `resolveId` hook with the cache
on returns `{ id }` for `./b`, without `moduleSideEffects`. `a.ts` resolves
`./b` before `b.ts` is transformed, so Rolldown passes on nothing, and `a.ts`
gets a saved result. `c.ts` resolves `./b` after that, so Rolldown passes on
`false`, and `c.ts` gets no saved result. In another build, the order can be
the other way.

**The warning for a shared meta object.** When a hook returns a meta object
that another module also holds, the warning says that the hook read or
changed the meta of that module. It does not say that the object is shared.
To keep the saved result, return a new meta object for each module.

## Future work

1. **Let a module get a saved result when Rolldown passes on a
   `moduleSideEffects` value.**

   - **Problem:** A JavaScript hook returns an object without
     `moduleSideEffects`, and Rolldown passes on the value that another hook
     set. The module then gets no saved result. See
     [`moduleSideEffects`](#modulesideeffects).
   - **Possible fix:** Stop passing on a value that another hook set, as
     Rollup does. This changes the output of a build without the cache too,
     so the Rolldown maintainers must decide it.

## Verification

In every case, a later hook must read the same values as in a build without
the cache.

1. Meta through `this.getModuleInfo`: A hook of `a.ts` that has the cache on
   sets a key on its own meta and a key on the meta of `b.ts` through
   `this.getModuleInfo`. Expect no saved result for `a.ts` and one
   `PERSISTENT_CACHE_MODULE_INFO` warning. Restart. Expect both keys, and
   the same output as a build without the cache.
2. Assigned `moduleSideEffects`: A hook of `a.ts` that has the cache on
   assigns `moduleSideEffects = true` on `b.ts`. Expect no saved result for
   `a.ts` and one `PERSISTENT_CACHE_MODULE_INFO` warning. Restart. Expect
   the same output and the same value from `this.getModuleInfo` as a build
   without the cache.
3. A `this.resolve` result: A `resolveId` hook that has the cache on returns
   the result of `this.resolve` for `./b` as it is. No hook sets
   `moduleSideEffects` for `b.ts`. Expect a saved result for the importing
   module. Then the hook returns `{ ...result }` instead. Expect no saved
   result for the importing module and one `PERSISTENT_CACHE_MODULE_INFO`
   warning.
4. A meta object of another module: A `resolveId` hook that has the cache on
   changes the `id` of a `this.resolve` result for `./b` to `c.ts`, and
   returns it. Expect no saved result for the importing module and one
   `PERSISTENT_CACHE_MODULE_INFO` warning. Restart. Expect the same meta on
   `c.ts` as a build without the cache.
5. A passed-on `moduleSideEffects`: A `resolveId` hook returns
   `moduleSideEffects: false` for `./b`. The `transform` hook of `b.ts` has
   the cache on and returns `{ code }`. Expect no saved result for `b.ts` and
   no warning. Restart. Expect the same output as a build without the cache.
