接下来我们看一下 resolveConfig 函数做了什么。

##### resolveConfig

在 resolveConfig 中，里面加载了配置文件

```javascript
const loadResult = await loadConfigFromFile(
  configEnv,
  configFile,
  config.root,
  config.logLevel
);
```

我们来看下具体实现，对于不同的文件做了不同的处理

- package.json 中的 type === 'module'——isESM 为 true
- 存在 vite.config.ts——isTS 为 true

使用 esbuild 将文件转换，通过 isESM 来进行配置

```
 format: isESM ? 'esm' : 'cjs',
```

再获取配置文件

```
  userConfig = (await dynamicImport(`...`)).default

  // 合并
  config = mergeConfig(loadResult.config, config)
```

#### rawUserPlugins

获取对应的插件

```javascript
const rawUserPlugins = (config.plugins || []).flat(Infinity).filter((p) => {
    if (!p) {
      return false
    } else if (!p.apply) {
      return true
    } else if (typeof p.apply === 'function') {
      return p.apply({ ...config, mode }, configEnv)
    } else {
      return p.apply === command
    }
```

对于筛选插件

```javascript
const [prePlugins, normalPlugins, postPlugins] =
  sortUserPlugins(rawUserPlugins);
```

##### sortUserPlugins

对插件的执行顺序进行排序

```typescript
export function sortUserPlugins(
  plugins: (Plugin | Plugin[])[] | undefined
): [Plugin[], Plugin[], Plugin[]] {
  const prePlugins: Plugin[] = [];
  const postPlugins: Plugin[] = [];
  const normalPlugins: Plugin[] = [];

  if (plugins) {
    plugins.flat().forEach((p) => {
      if (p.enforce === "pre") prePlugins.push(p);
      else if (p.enforce === "post") postPlugins.push(p);
      else normalPlugins.push(p);
    });
  }

  return [prePlugins, normalPlugins, postPlugins];
}
```

按照对应的顺序去执行 config 这个 钩子

```javaScript
 const userPlugins = [...prePlugins, ...normalPlugins, ...postPlugins]
  for (const p of userPlugins) {
    if (p.config) {
      const res = await p.config(config, configEnv)
      if (res) {
        config = mergeConfig(config, res)
      }
    }
  }
```

##### resolvePlugins

整合插件

```javaScript
export async function resolvePlugins(
  config: ResolvedConfig,
  prePlugins: Plugin[],
  normalPlugins: Plugin[],
  postPlugins: Plugin[]
): Promise<Plugin[]> {
  const isBuild = config.command === 'build'
  const isWatch = isBuild && !!config.build.watch

  const buildPlugins = isBuild
    ? (await import('../build')).resolveBuildPlugins(config)
    : { pre: [], post: [] }

  return [
    isWatch ? ensureWatchPlugin() : null,
    isBuild ? metadataPlugin() : null,
    isBuild ? null : preAliasPlugin(),
    aliasPlugin({ entries: config.resolve.alias }),
    ...prePlugins,
    config.build.polyfillModulePreload
      ? modulePreloadPolyfillPlugin(config)
      : null,
    resolvePlugin({
      ...config.resolve,
      root: config.root,
      isProduction: config.isProduction,
      isBuild,
      packageCache: config.packageCache,
      ssrConfig: config.ssr,
      asSrc: true
    }),
    isBuild ? null : optimizedDepsPlugin(),
    htmlInlineProxyPlugin(config),
    cssPlugin(config),
    config.esbuild !== false ? esbuildPlugin(config.esbuild) : null,
    jsonPlugin(
      {
        namedExports: true,
        ...config.json
      },
      isBuild
    ),
    wasmHelperPlugin(config),
    webWorkerPlugin(config),
    assetPlugin(config),
    ...normalPlugins,
    wasmFallbackPlugin(),
    definePlugin(config),
    cssPostPlugin(config),
    config.build.ssr ? ssrRequireHookPlugin(config) : null,
    isBuild && buildHtmlPlugin(config),
    workerImportMetaUrlPlugin(config),
    ...buildPlugins.pre,
    dynamicImportVarsPlugin(config),
    importGlobPlugin(config),
    ...postPlugins,
    ...buildPlugins.post,
    // internal server-only plugins are always applied after everything else
    ...(isBuild
      ? []
      : [clientInjectionsPlugin(config), importAnalysisPlugin(config)])
  ].filter(Boolean) as Plugin[]
}

```

执行 configResolved 钩子

```javascript
// 执行worker相关的钩子
await Promise.all(
  resolved.worker.plugins.map((p) => p.configResolved?.(workerResolved))
);
// 执行用户相关的钩子
await Promise.all(userPlugins.map((p) => p.configResolved?.(resolved)));
```

返回 resloved
