#### 预编译优化

> packages->vite->src->node->server->index.ts

在 server 调用 listen 的时候，会去调用 initOptimizer 方法，这个方法就做了一系列的优化。

```javascript
httpServer.listen = (async (port: number, ...args: any[]) => {
      if (!isOptimized) {
        try {
          await container.buildStart({})
          initOptimizer()
          isOptimized = true
        } catch (e) {
          httpServer.emit('error', e)
          return
        }
      }
      return listen(port, ...args)
    }) as any
```

##### initOptimizer

这个函数中调用了`createOptimizedDeps`方法

```javascript
const initOptimizer = () => {
  if (!config.optimizeDeps.disabled) {
    server._optimizedDeps = createOptimizedDeps(server);
  }
};
```

##### createOptimizedDeps

> packages->vite->src->node->optimizer->registerMissing.ts

这里会创建一个缓存的 `metaData `的数据

```javascript
export function createOptimizedDeps(server: ViteDevServer): OptimizedDeps {
   ...
   const cachedMetadata = loadCachedDepOptimizationMetadata(config)
   ...
}
```

##### loadCachedDepOptimizationMetadata

获取缓存内容，如果 hash 相同的话，就返回相同的数据，否则就删除对应的文件。

```javascript
const cachedMetadataPath = path.join(depsCacheDir, "_metadata.json");
cachedMetadata = parseOptimizedDepsMetadata(
  fs.readFileSync(cachedMetadataPath, "utf-8"),
  depsCacheDir
);

if (cachedMetadata && cachedMetadata.hash === getDepHash(config)) {
  log("Hash is consistent. Skipping. Use --force to override.");
  // Nothing to commit or cancel as we are using the cache, we only
  // need to resolve the processing promise so requests can move on
  return cachedMetadata;
}
```

去查找对应的一些依赖

```javascript
const discovered = await discoverProjectDependencies(config, sessionTimestamp);
```

##### discoverProjectDependencies

使用 scanImports 去找到对应的引入，也就是 deps

```javascript
export async function discoverProjectDependencies(
  config: ResolvedConfig,
  timestamp?: string
): Promise<Record<string, OptimizedDepInfo>> {
  const { deps, missing } = await scanImports(config);
}
```

##### scanImports

针对所有的 `html` 文件去查找他们的引用

```javascript
entries = await globEntries("**/*.html", config);
...
await Promise.all(
  entries.map((entry) =>
    build({
      absWorkingDir: process.cwd(),
      write: false,
      entryPoints: [entry],
      bundle: true,
      format: "esm",
      logLevel: "error",
      plugins: [...plugins, plugin],
      ...esbuildOptions,
    })
  )
);
```

调用`esbuildScanPlugin`方法

```javascript
const plugin = esbuildScanPlugin(config, container, deps, missing, entries);
```

##### esbuildScanPlugin

如果是在 node_modules 中，就写入 deps 中（这里的 depImports 就是传参进来的 deps）

```javascript
if (resolved.includes("node_modules") || include?.includes(id)) {
  // dependency or forced included, externalize and stop crawling
  if (isOptimizable(resolved)) {
    depImports[id] = resolved;
  }
  return externalUnlessEntry({ path: id });
}
```

接下来会调用 runOptimizeDeps 这个函数

```javascript
const processingResult = await runOptimizeDeps(config, newDeps);
```

##### runOptimizeDeps

对于类似 loadash 这种包，去打包成为一个，通过`esbuildDepPlugin`去合并一些包，打包之后会生成一些文件在.vite 目录下面

```javascript
const result = await build({
  absWorkingDir: process.cwd(),
  entryPoints: Object.keys(flatIdDeps),
  bundle: true,
  format: "esm",
  target: config.build.target || undefined,
  external: config.optimizeDeps?.exclude,
  logLevel: "error",
  splitting: true,
  sourcemap: true,
  outdir: processingCacheDir,
  ignoreAnnotations: true,
  metafile: true,
  define,
  plugins: [...plugins, esbuildDepPlugin(flatIdDeps, flatIdToExports, config)],
  ...esbuildOptions,
});
```

接下来会写入`_metadata.json`文件

```javascript
const dataPath = path.join(processingCacheDir, "_metadata.json");
writeFile(dataPath, stringifyOptimizedDepsMetadata(metadata, depsCacheDir));
```
