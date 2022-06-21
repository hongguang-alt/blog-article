#### handleHMRUpdate

> pagekages->vite->src->node->server->index.ts

```javascript
watcher.on("change", async (file) => {
  file = normalizePath(file);
  if (file.endsWith("/package.json")) {
    return invalidatePackageData(packageCache, file);
  }
  // invalidate module graph cache on file change
  moduleGraph.onFileChange(file);
  if (serverConfig.hmr !== false) {
    try {
      await handleHMRUpdate(file, server);
    } catch (err) {
      ws.send({
        type: "error",
        err: prepareError(err),
      });
    }
  }
});
```

这里会有一个 `watcher` 去监听文件的变化，获取一个路径，然后去触发 `onFileChange` 这个方法，然后去执行了`handleHMRUpdate`这个方法

##### onFileChange

通过 file 返回的模块有可能不止一个

```javascript
 onFileChange(file: string): void {
    const mods = this.getModulesByFile(file)
    if (mods) {
      const seen = new Set<ModuleNode>()
      mods.forEach((mod) => {
        this.invalidateModule(mod, seen)
      })
    }
  }
```

##### handleHMRUpdate

> packages->vite->src->node->server->hmr.ts

生成一个 hmr 的 context，调用插件的 handleHotUpdate 的钩子，然后把对应的模块赋值给 hmrContext 的 modules。调用的 updateModule 函数

```javascript
const hmrContext: HmrContext = {
  file,
  timestamp,
  modules: mods ? [...mods] : [],
  read: () => readModifiedFile(file),
  server,
};

for (const plugin of config.plugins) {
  if (plugin.handleHotUpdate) {
    const filteredModules = await plugin.handleHotUpdate(hmrContext);
    if (filteredModules) {
      hmrContext.modules = filteredModules;
    }
  }
}
...

 updateModules(shortFile, hmrContext.modules, timestamp, server)
```

#### readModifiedFile

- 等到文件的更新时间和老的时间不一样，才去读取这个文件（在 change 文件的时候，不一定写进了文件，确保一定能够读取到文件正确的内容）

```javascript
async function readModifiedFile(file: string): Promise<string> {
  const content = fs.readFileSync(file, "utf-8");
  if (!content) {
    const mtime = fs.statSync(file).mtimeMs;
    await new Promise((r) => {
      let n = 0;
      const poll = async () => {
        n++;
        const newMtime = fs.statSync(file).mtimeMs;
        if (newMtime !== mtime || n > 10) {
          r(0);
        } else {
          setTimeout(poll, 10);
        }
      };
      setTimeout(poll, 10);
    });
    return fs.readFileSync(file, "utf-8");
  } else {
    return content;
  }
}
```

#### updateModules

`updates`里面会推入 `boundaries` 的数据，然后发送 `update`

```javascript
function updateModules(
  file: string,
  modules: ModuleNode[],
  timestamp: number,
  { config, ws }: ViteDevServer
) {
  const updates: Update[] = []
  const invalidatedModules = new Set<ModuleNode>()
  let needFullReload = false

  for (const mod of modules) {
    invalidate(mod, timestamp, invalidatedModules)
    if (needFullReload) {
      continue
    }

    const boundaries = new Set<{
      boundary: ModuleNode
      acceptedVia: ModuleNode
    }>()
    const hasDeadEnd = propagateUpdate(mod, boundaries)
    if (hasDeadEnd) {
      needFullReload = true
      continue
    }

    updates.push(
      ...[...boundaries].map(({ boundary, acceptedVia }) => ({
        type: `${boundary.type}-update` as Update['type'],
        timestamp,
        path: boundary.url,
        acceptedPath: acceptedVia.url
      }))
    )
  }

  if (needFullReload) {
    config.logger.info(colors.green(`page reload `) + colors.dim(file), {
      clear: true,
      timestamp: true
    })
    ws.send({
      type: 'full-reload'
    })
  } else {
    config.logger.info(
      updates
        .map(({ path }) => colors.green(`hmr update `) + colors.dim(path))
        .join('\n'),
      { clear: true, timestamp: true }
    )
    ws.send({
      type: 'update',
      updates
    })
  }
}
```

##### propagateUpdate

对于每个模块逐向上层遍历，不断推入 boundaries，hasDeadEnd 为 true 直接 fullReload，这就表明产生的循环引入。

```javascript
if (currentChain.includes(importer)) {
  // circular deps is considered dead end
  return true;
}

if (propagateUpdate(importer, boundaries, subChain)) {
  return true;
}
```
