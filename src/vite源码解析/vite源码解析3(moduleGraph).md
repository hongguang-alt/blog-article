接着往下看

```javascript
 const middlewares = connect() as Connect.Server
  const httpServer = middlewareMode
    ? null
    : await resolveHttpServer(serverConfig, middlewares, httpsOptions)
  const ws = createWebSocketServer(httpServer, config, httpsOptions)

  const { ignored = [], ...watchOptions } = serverConfig.watch || {}
  const watcher = chokidar.watch(path.resolve(root), {
    ignored: [
      '**/node_modules/**',
      '**/.git/**',
      ...(Array.isArray(ignored) ? ignored : [ignored])
    ],
    ignoreInitial: true,
    ignorePermissionErrors: true,
    disableGlobbing: true,
    ...watchOptions
  }) as FSWatcher
```

这里的中间件是通过 connect 来实现的，这个 connect 是一个经典的 node 包，express 的中间件也是这样实现的。

这里使用 chokidar 去监听的一些文件的变化，这也是一个开源的包。

#### ModuleGraph

模块图谱，创建了一个模块图谱

```javascript
const moduleGraph: ModuleGraph = new ModuleGraph((url, ssr) =>
  container.resolveId(url, undefined, { ssr })
);
```

我们来看看这个模块图谱做了什么事情：

> 在 vite 源码中，packages->vite->src->node->server->moduleGraph.ts

提供了一些方法

- getModuleByUrl——通过 url 获取模块
- getModuleById——通过 id 获取模块
- getModulesByFile——通过文件获取模块

这里需要注意的是 file 可能对应的是多个模块，典型的就是 vue，里面的 template 以及 js 和 css 对应了不同的模块，所以这里使用的是 set

```javascript
fileToModulesMap = new Map<string, Set<ModuleNode>>()
urlToModuleMap = new Map<string, ModuleNode>()
idToModuleMap = new Map<string, ModuleNode>()
...
```

##### updateModuleInfo

- 更新模块之间的依赖关系,维护每一个模块

```javascript
 async updateModuleInfo(
    mod: ModuleNode,
    importedModules: Set<string | ModuleNode>,
    acceptedModules: Set<string | ModuleNode>,
    isSelfAccepting: boolean,
    ssr?: boolean
  ): Promise<Set<ModuleNode> | undefined> {
    mod.isSelfAccepting = isSelfAccepting
    const prevImports = mod.importedModules
    const nextImports = (mod.importedModules = new Set())
    let noLongerImported: Set<ModuleNode> | undefined
    // update import graph
    for (const imported of importedModules) {
      const dep =
        typeof imported === 'string'
          ? await this.ensureEntryFromUrl(imported, ssr)
          : imported
      dep.importers.add(mod)
      nextImports.add(dep)
    }
    // remove the importer from deps that were imported but no longer are.
    prevImports.forEach((dep) => {
      if (!nextImports.has(dep)) {
        dep.importers.delete(mod)
        if (!dep.importers.size) {
          // dependency no longer imported
          ;(noLongerImported || (noLongerImported = new Set())).add(dep)
        }
      }
    })
    // update accepted hmr deps
    const deps = (mod.acceptedHmrDeps = new Set())
    for (const accepted of acceptedModules) {
      const dep =
        typeof accepted === 'string'
          ? await this.ensureEntryFromUrl(accepted, ssr)
          : accepted
      deps.add(dep)
    }
    return noLongerImported
  }

```

updateModuleInfo 会在 importAnalysis 这个插件中被调用

```javascript
const prunedImports = await moduleGraph.updateModuleInfo(
  importerModule,
  importedUrls,
  normalizedAcceptedUrls,
  isSelfAccepting,
  ssr
);
```

在 middlewares 的 transformMiddleware 中也用到了

> 在 vite 源码中，packages->vite->src->node->server->middlewares->transform.ts

##### transformMiddleware

```javascript
// Keep the named function. The name is visible in debug logs via `DEBUG=connect:dispatcher ...`
  return async function viteTransformMiddleware(req, res, next) {
    if (req.method !== 'GET' || knownIgnoreList.has(req.url!)) {
      return next()
    }

```

对于一些请求的处理，其中包含了缓存部分内容

```javascript
// check if we can return 304 early
const ifNoneMatch = req.headers["if-none-match"];
if (
  ifNoneMatch &&
  (await moduleGraph.getModuleByUrl(url, false))?.transformResult?.etag ===
    ifNoneMatch
) {
  isDebug && debugCache(`[304] ${prettifyUrl(url, root)}`);
  res.statusCode = 304;
  return res.end();
}
```

然后开始对文件进行处理，调用的是 transformRequest 方法

```javascript
const result = await transformRequest(url, server, {
  html: req.headers.accept?.includes("text/html"),
});
```

在这里面调用所有插件的 `resloveId`，直到第一个 resloveId 返回对应的路径

```javascript
// resolve
const id =
  (await pluginContainer.resolveId(url, undefined, { ssr }))?.id || url;
```

调用插件的 `load`

```javascript
const loadResult = await pluginContainer.load(id, { ssr });
```

调用 ensureEntryFromUrl 方法，将这个模块存放到对应的 map 中,以方便后续通过不同的方式去获取模块

```javascript
const mod = await moduleGraph.ensureEntryFromUrl(url, ssr);
```

通过 ensureWatchedFile 方法，保证当前的模块被监听了

```javascript
ensureWatchedFile(watcher, mod.file, root);
```

调用插件的`transform`钩子

```javascript
const transformResult = await pluginContainer.transform(code, id, {
  inMap: map,
  ssr,
});
```
