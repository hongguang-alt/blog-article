#### pluginContainer

```javascript
const container = await createPluginContainer(config, moduleGraph, watcher);
```

> packages->vite->src->node->server->pluginContainer.ts

这里的`PluginContext`是作为 rollup 的 context

```javascript
PluginContext as RollupPluginContext
```

vite 的 context 是基于 rollup 来实现的

```javascript
class Context implements PluginContext {}
```

有些方法 vite 实现了，有些并没有

> 比如 emitFile 并没有实现

```javascript
 emitFile(assetOrFile: EmittedFile) {
      warnIncompatibleMethod(`emitFile`, this._activePlugin!.name)
      return ''
    }
```

定义了一个`container`,这个返回的就是 pluginContext,里面包含了各种 rollup（vite）的相关钩子

```javascript
const container: PluginContainer = {
  async buildStart() {
     await Promise.all(
        plugins.map((plugin) => {
          if (plugin.buildStart) {
            return plugin.buildStart.call(
              new Context(plugin) as any,
              container.options as NormalizedInputOptions
            )
          }
        })
      )
  },
  async resolveId() {},
  async load() {},
  async transform() {},
  async close() {},
};
```

每次都会 new 一个 Context,作为 this 去使用

这里的 resloveId，在找到第一个之后就会返回，后面的 resolveId 就不会去执行了

```javascript

 async resolveId(rawId, importer = join(root, 'index.html'), options) {
      const skip = options?.skip
      const ssr = options?.ssr
      const scan = !!options?.scan
      const ctx = new Context()
      ctx.ssr = !!ssr
      ctx._scan = scan
      ctx._resolveSkips = skip
      const resolveStart = isDebug ? performance.now() : 0

      let id: string | null = null
      const partial: Partial<PartialResolvedId> = {}
      for (const plugin of plugins) {
        if (!plugin.resolveId) continue
        if (skip?.has(plugin)) continue

        ctx._activePlugin = plugin

        const pluginResolveStart = isDebug ? performance.now() : 0
        const result = await plugin.resolveId.call(
          ctx as any,
          rawId,
          importer,
          { ssr, scan }
        )
        if (!result) continue

        if (typeof result === 'string') {
          id = result
        } else {
          id = result.id
          Object.assign(partial, result)
        }

        isDebug &&
          debugPluginResolve(
            timeFrom(pluginResolveStart),
            plugin.name,
            prettifyUrl(id, root)
          )

        // resolveId() is hookFirst - first non-null result is returned.
        break
      }

      if (isDebug && rawId !== id && !rawId.startsWith(FS_PREFIX)) {
        const key = rawId + id
        // avoid spamming
        if (!seenResolves[key]) {
          seenResolves[key] = true
          debugResolve(
            `${timeFrom(resolveStart)} ${colors.cyan(rawId)} -> ${colors.dim(
              id
            )}`
          )
        }
      }

      if (id) {
        partial.id = isExternalUrl(id) ? id : normalizePath(id)
        return partial as PartialResolvedId
      } else {
        return null
      }
    },
```

transform 每个钩子都会执行，会返回对应的函数

```javascript
async transform(code, id, options) {
      const inMap = options?.inMap
      const ssr = options?.ssr
      const ctx = new TransformContext(id, code, inMap as SourceMap)
      ctx.ssr = !!ssr
      for (const plugin of plugins) {
        if (!plugin.transform) continue
        ctx._activePlugin = plugin
        ctx._activeId = id
        ctx._activeCode = code
        const start = isDebug ? performance.now() : 0
        let result: TransformResult | string | undefined
        try {
          result = await plugin.transform.call(ctx as any, code, id, { ssr })
        } catch (e) {
          ctx.error(e)
        }
        if (!result) continue
        isDebug &&
          debugPluginTransform(
            timeFrom(start),
            plugin.name,
            prettifyUrl(id, root)
          )
        if (isObject(result)) {
          if (result.code !== undefined) {
            code = result.code
            if (result.map) {
              ctx.sourcemapChain.push(result.map)
            }
          }
          updateModuleInfo(id, result)
        } else {
          code = result
        }
      }
      return {
        code,
        map: ctx._getCombinedSourcemap()
      }
    },
```

container 的作用就是外界调用 container 的方法会去调用所有的钩子

```javascript
container.buildStart({});
```

相当于调用了所有插件的 `buildStart` 钩子
