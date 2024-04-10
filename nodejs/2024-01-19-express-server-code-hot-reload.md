---
category: nodejs
author: forhot2000@qq.com
date: 2024/01/19
---

# 实现 Express 服务器端代码热加载

## 创建项目

首先，创建一个 express 的创建项目

```shell
mkdir your-project
cd your-project
npm init -y
npm install --save express
```

添加 `src/main.js`

```js
// ./src/main.js
const express = require("express");

function bootstrap() {
  const app = express();

  app.all("*", (req, res) => {
    res.send(`hello, world!\n`);
  });

  app.listen(3000, function () {
    console.log("listen on http://localhost:3000");
  });
}

bootstrap();
```

修改 `package.json`，增加 start 命令

```json
{
  ...
  "scripts": {
    "start": "node src/main.js"
  },
  ...
}
```

启动服务

```shell
npm run start
```

OK，这就启动了一个基本的 express app 了。

但是，每次修改代码后，都需要手动重新启动服务，对于开发来说，这是个非常烦人的工作，下面让我们一步一步来优化它。

## 使用 nodemon 自动重启

使用 nodemon 检测文件变动，重启服务，这种方式很简单，不需要修改现有代码。

安装 nodemon

```shell
npm install --save-dev nodemon
```

修改 package.json，增加 dev 命令，使用 nodemon 启动，其它都不用改

```json
{
  ...
  "scripts": {
    "dev": "nodemon src/main.js",
    "start": "node src/main.js"
  }
  ...
}
```

配置好后，使用 `npm run dev` 启动服务，nodemon 会检测文件改动自动重启服务器，这样你不用再频繁的重启服务，欢快地去写代码了。

如果你需要排除一些文件的监控，比如仅检测 `src` 目录下的 js 文件，并忽略测试代码，可以添加 nodemon 的配置文件 `nodemon.json`

```json
{
  "watch": ["src/"],
  "ext": "js",
  "ignore": ["*.test.js", "*.spec.js"]
}
```

如上所示，nodemon 的使用非常简单，配合 ts-node 它还能支持 typescript，已经能满足大多数用户的使用场景了。

不过，当项目变的越来越大，每次改动一个地方就重新启动服务就变得有点麻烦了。

## 使用 webpack HMR 实现模块热加载

webpack 的 HMR 功能会通知到哪些文件发生了变化需要重新加载，这个功能被广泛用在前端开发框架中，修改代码后立即刷新页面，其实它也还可以被用在服务器端代码的加载过程中，让我们来看看如何实现。

首先，添加依赖包

```shell
npm install --save-dev webpack webpack-cli webpack-node-externals run-script-webpack-plugin rimraf
```

添加两个新文件，用来测试热加载

```js
// ./src/count.js
let n = 0;

export function inc() {
  n++;
  return n;
}
```

```js
// ./src/hello.js
export function greet(name = "World") {
  return `Hello, ${name}!`;
}
```

修改 `src/main.js`，引入上面的文件，并添加响应热加载的代码，由于使用了 webpack，现在可以在代码中使用 ES6 的 import 了，改动后的 `main.js` 代码如下：

```js
// ./src/main.js
import express from "express";
import { createServer } from "http";
import { inc } from "./count";
import { greet } from "./hello";

function bootstrap() {
  const app = express();

  // 默认情况下 express 会自动创建 server，这里手动创建 server
  // 是为了在后面调用 server.close() 关闭旧的服务
  const server = createServer(app);

  app.all("*", (req, res) => {
    const n = inc();
    res.send(`${n}: ${greet("bob")}\n`);
  });

  server.listen(3000, function () {
    console.log("listen on http://localhost:3000");
  });

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => server.close());
  }
}

bootstrap();
```

修改 `package.json`

```json
{
  ...
  "scripts": {
    "dev": "rimraf dist && webpack --config webpack-hmr.config.js --watch",
    "build": "rimraf dist && webpack --config webpack.config.js",
    "start": "node dist/server.js"
  },
  ...
}
```

添加 build 的 webpack 配置文件 `webpack.config.js`

```js
// ./webpack.config.js
const path = require("path");
const nodeExternals = require("webpack-node-externals");

module.exports = {
  mode: "production",
  entry: ["./src/main.js"],
  target: "node",
  externals: [nodeExternals()],
  resolve: {
    extensions: [".js"],
  },
  output: {
    path: path.join(__dirname, "dist"),
    filename: "server.js",
  },
};
```

添加 dev 的 webpack 配置文件 `webpack-hmr.config.js`

```js
// ./webpack-hmr.config.js
const webpack = require("webpack");
const path = require("path");
const nodeExternals = require("webpack-node-externals");
const { RunScriptWebpackPlugin } = require("run-script-webpack-plugin");

module.exports = {
  mode: "development",
  entry: ["webpack/hot/poll?100", "./src/main.js"],
  target: "node",
  externals: [
    nodeExternals({
      allowlist: ["webpack/hot/poll?100"],
    }),
  ],
  resolve: {
    extensions: [".js"],
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new RunScriptWebpackPlugin({ name: "server.js", autoRestart: false }),
  ],
  output: {
    path: path.join(__dirname, "dist"),
    filename: "server.js",
  },
};
```

开发模式下，我们使用 `npm run dev` 启动服务，启用热加载功能。

接下来，我们来测试一下热加载

```shell
curl http://localhost:3000/
```

输出

```
1: Hello, boy!
```

当我们修改 `main.js` 后，服务器端仅需要重新编译并加载 `main.js` 一个文件

```log
...
asset server.js 46.1 KiB [emitted] (name: main)
asset main.44ac5c7bc9372be2efa5.hot-update.js 2.58 KiB [emitted] [immutable] [hmr] (name: main)
asset main.44ac5c7bc9372be2efa5.hot-update.json 28 bytes [emitted] [immutable] [hmr]
Entrypoint main 48.7 KiB = server.js 46.1 KiB main.44ac5c7bc9372be2efa5.hot-update.js 2.58 KiB
runtime modules 23.5 KiB 9 modules
cached modules 4.69 KiB [cached] 7 modules
./src/main.js 677 bytes [built] [code generated]
webpack 5.89.0 compiled successfully in 32 ms
[HMR] Updated modules:
[HMR]  - ./src/main.js
[HMR] Update applied.
listen on http://localhost:3000/
```

再次访问服务

```shell
curl http://localhost:3000/
```

输出

```
2: Hello, boy!
```

我们可以看到，`count.js` 中的计数并没有重置

再试下修改 `hello.js`，webpack 会重新编译并加载 `hello.js`，因为 `main.js` 引用了 `hello.js`，所以虽然不会重新编译 `main.js`，但是它也会被重新加载。

```js
...
asset server.js 46.1 KiB [emitted] (name: main)
asset main.56dcdc90891aac75fe1c.hot-update.js 1.37 KiB [emitted] [immutable] [hmr] (name: main)
asset main.56dcdc90891aac75fe1c.hot-update.json 28 bytes [emitted] [immutable] [hmr]
Entrypoint main 47.5 KiB = server.js 46.1 KiB main.56dcdc90891aac75fe1c.hot-update.js 1.37 KiB
runtime modules 23.5 KiB 9 modules
cached modules 5.28 KiB [cached] 7 modules
./src/hello.js 72 bytes [built] [code generated]
webpack 5.89.0 compiled successfully in 65 ms
[HMR] Updated modules:
[HMR]  - ./src/hello.js
[HMR]  - ./src/main.js
[HMR] Update applied.
listen on http://localhost:3000/
```

再次访问服务

```shell
curl http://localhost:3000/
```

输出

```
3: Hello, boy!
```

可以看到，`count.js` 中的计数任然没有被重置，说明只要不修改 `count.js` 及其依赖项，`count.js` 就不会被重新加载。

### 发布到生产环境

生产环境下，我们不需要热加载功能，那么我们可以运行 `npm run build` 构建代码，然后再运行 `npm start`，使用构建后的代码启动服务，这样优先保证线上环境的性能。

### 动态加载目录的问题

另外还有一个常见的问题，有时候我们需要动态的加载某个目录下的所有文件，这个可以用 await import 来加载模块来完成。

让我们来改一下 `main.js`，将 `bootstrap` 改成 `async` 方法，再增加一个 `loadControllers` 方法

```js
// ./src/main.js
// ...
import { readdir } from "fs/promises";
import { resolve } from "path";

async function bootstrap() {
  // ...
  const server = createServer(app);

  // 动态加载 controllers 目录下的所有文件
  await loadControllers(app);

  app.all("*", (req, res) => {
    /* ... */
  });
  //...
}

async function loadControllers(app) {
  try {
    // 注意，这里应该是扫描 src 目录，而不是 dist 目录
    const files = await readdir(resolve("./src/controllers"));
    for (let i = 0; i < files.length; i++) {
      const file = files[i];
      const name = file.substring(0, file.length - 3); // remove ext '.js'
      const module = await import("./controllers/" + name);
      app.use(`/${name}`, module.default);
    }
  } catch (err) {
    console.error(err);
  }
}

bootstrap();
```

> **提示**
>
> 遍历目录文件的时候需要使用源文件的目录，而不能使用文件的相对目录
>
> 错误的代码
>
> ```js
> // 编译后的代码会提示找不到目录 ./dist/controllers
> readdir(path.join(__dirname, "controllers"));
> ```
>
> 正确的代码
>
> ```js
> readdir(path.resolve("./src/controllers"));
> ```

再添加一个 controller 文件 `src/controllers/posts.js`

```js
// ./src/controllers/posts.js
import { Router } from "express";

const router = Router();

router.get("/", (req, res) => {
  res.send([
    {
      id: 1,
      title: "post 1",
      content: "content of the post",
    },
  ]);
});

export default router;
```

测试下新加的 controller

```shell
curl http://localhost:3000/posts
```

输出

```json
[{ "id": 1, "title": "post 1", "content": "content of the post" }]
```

这种方式存在一个问题，每次都要去扫描 src 目录，导致部署的时候还需要将 src 目录复制到服务器，而这些 src 目录下的文件除了提供一个 filename 就没有其它作用了，我认为这不是一个好的代码。

如果要避免这种隐式的动态加载，可以将它改成如下代码：

```js
// 显示声明有哪些 controllers
const controllerNames = ["posts"];

async function loadControllers(app) {
  const controllers = controllerNames.map((name) => ({ name }));
  for (let i = 0; i < controllers.length; i++) {
    const controller = controllers[i];
    try {
      const module = await import(`./controllers/${controller.name}`);
      controller.router = module.default;
    } catch (err) {
      // console.error(err);
    }
  }

  controllers.forEach(({ name, router }) => {
    if (router) {
      const path = `/${name}`;
      app.use(path, router);
      console.log(`mount controller '${name}' on '${path}'`);
    } else {
      console.error(`cannot find controller '${name}'`);
    }
  });
}
```
