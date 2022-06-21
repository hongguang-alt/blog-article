### 前言

主要是针对 vite 的 cli 的 node 部分进行解析，由于水平有限，只能够简单的解析~

#### createServer

> 在 vite 源码中，packages->vite->src->node->cli.ts

```javascript
import { cac } from 'cac'
const cli = cac('vite')
cli
  .command('[root]', 'start dev server') // default command
  .alias('serve') // the command is called 'serve' in Vite's API
  .alias('dev') // alias to align with the script name
  .option(
    '--force',
    `[boolean] force the optimizer to ignore the cache and re-bundle`
  )
  ...
   .action(async (root: string, options: ServerOptions & GlobalCLIOptions) => {

   }
```

在这里主要是使用了 cac 这个包作为命令行的工具去使用。可以去了解一下。

接下来是比较关键的一个函数 createServer

```javaScript
 const server = await createServer({
        root,
        base: options.base,
        mode: options.mode,
        configFile: options.config,
        logLevel: options.logLevel,
        clearScreen: options.clearScreen,
        server: cleanOptions(options)
      })
```

> 在 vite 源码中，packages->vite->src->node->server->index.ts

```javascript
export async function createServer(
  inlineConfig: InlineConfig = {}
): Promise<ViteDevServer> {
  const config = await resolveConfig(inlineConfig, 'serve', 'development')
  const { root, server: serverConfig } = config
  const httpsOptions = await resolveHttpsConfig(
    config.server.https,
    config.cacheDir
  )
  ...
}
```

我们来看看这个 createServer 做了什么。

```
const config = await resolveConfig(inlineConfig, 'serve', 'development')
```

这里的 inlineConfig 主要是通过命令行接受的一些命令。