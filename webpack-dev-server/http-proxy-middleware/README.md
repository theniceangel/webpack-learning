# http-proxy-middleware

在分析了 [node-http-proxy](https://github.com/theniceangel/webpack-learning/tree/master/webpack-dev-server/node-http-proxy) 这个库之后，便马不停蹄的看了下 `http-proxy-middleware` 这个库的源码，因为 webpack-dev-server 用的就是这个库来实现代理。既然看懂了 `node-http-proxy`，那么这个中间件的代码也就非常容易理解了。

`http-proxy-middleware` 基于 `node-http-proxy` 之上，提供了**服务器分发(router)** 以及 **路径重写(pathRewrite)** 的能力。先看下官方的例子：

## 上手

```js
var express = require('express');
var proxy = require('http-proxy-middleware');

var app = express();

app.use(
  '/api',
  proxy({ target: 'http://www.example.org', changeOrigin: true })
);
app.listen(3000);
```

上面是 `http-proxy-middleware` 基本的使用，在 `express` 里面充当代理中间件的功能，效果就是访问 `http://localhost:3000/api/foo/bar` 的时候，给你代理至目标服务器 `http://www.example.org/api/foo/bar`，proxy 函数的选项不仅可以配置 `node-http-proxy` 的 [options](https://github.com/nodejitsu/node-http-proxy#options)，也支持 `http-proxy-middleware` 额外的一些[配置](https://github.com/chimurai/http-proxy-middleware#options)。

## 核心思想

官方提供的核心思想由如下配置组成：

```js
var proxy = require('http-proxy-middleware');

var apiProxy = proxy('/api', { target: 'http://www.example.org' });
//                   \____/   \_____________________________/
//                     |                    |
//                   context             options

// 'apiProxy' is now ready to be used as middleware in a server.
```

- **context**: 决定哪个客户端的请求需要被代理至目标服务器。
- **options.target**: 目标服务器. _(protocol + host)_

当然也提供了更简写(shorthand)的语法：

```js
// shorthand syntax for the example above:
var apiProxy = proxy('http://www.example.org/api');
```

这样的简写语法作用和上面的一模一样，模块内部会对 `http://www.example.org/api` 做处理，把 `/api` 赋值给 `context`，把 `http://www.example.org` 赋值给 `target`，稍后分析 [createConfig](https://github.com/chimurai/http-proxy-middleware/blob/master/src/config-factory.ts) 这个函数的作用就真想大白了。

## 入口

先从入口文件 `src/index.ts` 开始：

```typescript
import { HttpProxyMiddleware } from './http-proxy-middleware';

function proxy(context, opts) {
  const { middleware } = new HttpProxyMiddleware(context, opts);
  return middleware;
}

export = proxy;
```

导出的 proxy 方法，是返回 HttpProxyMiddleware 实例上的 middleware 方法。所以正如官网的介绍：

> The one-liner node.js http-proxy middleware for connect, express and browser-sync

这个库主要是用在 `connect`、`express`、`browser-sync` 里面提供中间件能力的。

## HttpProxyMiddleware Constructor

继续分析 `HttpProxyMiddleware` 这个类的实现。

```js
constructor(context, opts) {
  this.wsUpgradeDebounced = _.debounce(this.handleUpgrade);
  // 解析用户传入的配置项，拆分出 context 与 options。
  this.config = createConfig(context, opts);
  this.proxyOptions = this.config.options;

  // 创建代理请求
  this.proxy = httpProxy.createProxyServer({});
  this.logger.info(
    `[HPM] Proxy created: ${this.config.context}  -> ${
      this.proxyOptions.target
    }`
  );

  // 处理 pathRewrite 配置项，提供此库的核心能力——路径重写
  this.pathRewriter = PathRewriter.createPathRewriter(
    this.proxyOptions.pathRewrite
  );

  // 初始化代理服务器上的各种事件监听：
  // ['error', 'proxyReq', 'proxyReqWs', 'proxyRes', 'open', 'close']
  handlers.init(this.proxy, this.proxyOptions);

  // 注册 errorHandler
  this.proxy.on('error', this.logError);



  (this.middleware as any).upgrade = this.wsUpgradeDebounced;
  // 用来代理 websocket，用户可以通过 require('http-proxy-middleware').upgrade
  // 拿到这个函数，并且在适当的时机将 websocket 请求代理至目标服务器
  /**
   * var wsProxy = proxy('ws://echo.websocket.org', { changeOrigin: true });
   * var app = express();
   * app.use(wsProxy);
   * var server = app.listen(3000);
   * server.on('upgrade', wsProxy.upgrade);
   **／
}
```

构造函数的初始化逻辑非常的清晰，主要包括如下：

1. 暴露 `wsUpgradeDebounced` 函数用来代理 websocket 请求至目标服务器；
2. 解析传入的配置项，分离出 `context` 与 `options`；
3. 创建代理请求 this.proxy；
4. 新建 `pathRewrite` 模块，来实现**路径重写**；
5. 初始化代理服务器上的各种事件监听，例如 'error'、'proxyReq'、'proxyReqWs' 等等；
6. 注册代理服务器发生错误时的 errorHandler。

## HttpProxyMiddleware#middleware

构造函数做好了初始化的工作，暴露出去的是 middleware 方法，用户使用的就是这个方法，下面来看下这个方法做了哪些工作。

```typescript
public middleware = async (req, res, next) => {
  if (this.shouldProxy(this.config.context, req)) {
    const activeProxyOptions = this.prepareProxyRequest(req);
    this.proxy.web(req, res, activeProxyOptions);
  } else {
    next();
  }

  if (this.proxyOptions.ws === true) {
    this.catchUpgradeRequest(req.connection.server);
  }
};
```

middleware 逻辑很简单，第二个 if 分支是为了代理 websocket 请求，第一个 if 分支，是判断如果满足代理的要求，那么就用代理服务器代理这次客户端的请求，将客户端的请求转发至目标服务器， `this.shouldProxy()` 逻辑显得至关重要，因为它的返回值决定了这次请求是否需要被代理。而 `this.prepareProxyRequest()` 显然如命名一样，解析 createConfig 生成的 this.config 对象，最后将处理过后的对象传给 `node-http-proxy`，从而实现代理。

再分析 shouldProxy 之前，先看下 this.config 的数据结构到底是怎样的。代码在 `src/config-factory.ts`

```typescript
export function createConfig(context, opts?) {
  // structure of config object to be returned
  const config = {
    context: undefined,
    options: {} as any
  };

  if (isContextless(context, opts)) {
    config.context = '/';
    config.options = _.assign(config.options, context);

  } else if (isStringShortHand(context)) {
    const oUrl = url.parse(context);
    const target = [oUrl.protocol, '//', oUrl.host].join('');

    config.context = oUrl.pathname || '/';
    config.options = _.assign(config.options, { target }, opts);

    if (oUrl.protocol === 'ws:' || oUrl.protocol === 'wss:') {
      config.options.ws = true;
    }
  } else {
    config.context = context;
    config.options = _.assign(config.options, opts);
  }

  configureLogger(config.options);

  if (!config.options.target) {
    throw new Error(ERRORS.ERR_CONFIG_FACTORY_TARGET_MISSING);
  }

  return config;
}

function isContextless(context, opts) {
  return _.isPlainObject(context) && _.isEmpty(opts);
}

function isStringShortHand(context) {
  if (_.isString(context)) {
    return !!url.parse(context).host;
  }
}
```

createConfig 至少接收一个参数，并且对不同的情况做了不同的处理，目的就是为了得到 `context` 与 `options`。

1. context 是一个对象，opts 为假值，那么 `isContextless()` 这个分支就成立了，用法如下：

```js
let proxy = require('http-proxy-middleware')
proxy({target:'http://localhost:9000'})

// 处理之后

this.config = {
  context: '/'
  options: {
    target: 'http://localhost:9000'
  }
}
```

2. context 是 url 字符串，并且 url 必须包含了 host。那么 `isStringShortHand()` 这个分支就成立了，用法如下：

```js
let proxy = require('http-proxy-middleware')
proxy('http://localhost:9000/api')

// 处理之后

this.config = {
  context: '/api'
  options: {
    target: 'http://localhost:9000'
  }
}
```

3. 否则就是其他情况，比如 context 可以是一个校验函数，并且返回校验的布尔值，用法如下：

```js
var filter = function(pathname, req) {
  return pathname.match('^/api') && req.method === 'GET';
};

var apiProxy = proxy(filter, { target: 'http://www.example.org' });

// 处理之后

this.config = {
  context: filter
  options: {
    target: 'http://www.example.org'
  }
}
```

函数的尾部就是定制化 logger 的实现，因为模块内部默认使用 `console`，你可以改成其他的 `logger`，比如 [winston](https://github.com/winstonjs/winston)、[bunyan](https://github.com/trentm/node-bunyan)。

## HttpProxyMiddleware#shouldProxy

得到 context 之后，shouldProxy 就能判断这次请求是否需要被代理。

```typescript
private shouldProxy = (context, req) => {
  // 拿到用户的请求 url
  const path = req.originalUrl || req.url;
  // 匹配请求是否满足条件
  return contextMatcher.match(context, path, req);
};
```

contextMatcher 的代码位于 `src/context-matcher.ts`。

```js
export function match(context, uri, req) {
  // single path
  if (isStringPath(context)) {
    return matchSingleStringPath(context, uri);
  }

  // single glob path
  if (isGlobPath(context)) {
    return matchSingleGlobPath(context, uri);
  }

  // multi path
  if (Array.isArray(context)) {
    if (context.every(isStringPath)) {
      return matchMultiPath(context, uri);
    }
    if (context.every(isGlobPath)) {
      return matchMultiGlobPath(context, uri);
    }

    throw new Error(ERRORS.ERR_CONTEXT_MATCHER_INVALID_ARRAY);
  }

  // custom matching
  if (_.isFunction(context)) {
    const pathname = getUrlPathName(uri);
    return context(pathname, req);
  }

  throw new Error(ERRORS.ERR_CONTEXT_MATCHER_GENERIC);
}
```

contextMatcher 的匹配过程分为 4 种。

1. context 为单个字符串，那么就走进 matchSingleStringPath 的逻辑。

```js
function matchSingleStringPath(context, uri) {
  const pathname = getUrlPathName(uri);
  return pathname.indexOf(context) === 0;
}
```

假如 context 是 '/api'，req.url 是 '/api/user'，那么 match 就成功了。

2. context 为单个 glob 字符串，那么就走进 matchSingleGlobPath 的逻辑。

```js
function matchSingleGlobPath(pattern, uri) {
  const pathname = getUrlPathName(uri);
  const matches = micromatch([pathname], pattern);
  return matches && matches.length > 0;
}
```

假如 context 是 '/**'，req.url 是 '/api/user'，那么 match 就成功了。内部用的是 [micromatch](https://github.com/micromatch/micromatch) 这个库来匹配。

3. context 为单个数组，并且数组内部要么全是 url 字符串，要么全是 glob 字符串。

4. context 为一个函数，那么将请求的 pathname 以及 req 传入，根据函数内部的校验逻辑，来确定该请求是否需要代理。

## HttpProxyMiddleware#prepareProxyRequest

如果 `this.shouldProxy()` 如果返回 true，那么就走到代理的过程，先为真正的代理做准备，也就是执行 `this.prepareProxyRequest(req)`，代码如下：

```js
private prepareProxyRequest = req => {
  req.url = req.originalUrl || req.url;

  const originalPath = req.url;
  const newProxyOptions = _.assign({}, this.proxyOptions);

  this.applyRouter(req, newProxyOptions);
  this.applyPathRewrite(req, this.pathRewriter);

  // 省略...

  return newProxyOptions;
};
```

函数最核心的两个部分，一个是 `applyRouter`，掌管了 router 分发的能力，另一个是 `applyPathRewrite`，掌管了路径重写的能力。

1. **applyRouter**

```js
private applyRouter = (req, options) => {
  let newTarget;

  if (options.router) {
    // 获取新 target
    newTarget = Router.getTarget(req, options);

    // 打印请求
    if (newTarget) {
      this.logger.debug(
        '[HPM] Router new target: %s -> "%s"',
        options.target,
        newTarget
      );
      // 修改 options 的 target
      options.target = newTarget;
    }
  }
};
```

逻辑很简单，获取到新 target，所有的逻辑都包含在 Router.getTarget：

```js
export function getTarget(req, config) {
  let newTarget;
  const router = config.router;

  if (_.isPlainObject(router)) {
    newTarget = getTargetFromProxyTable(req, router);
  } else if (_.isFunction(router)) {
    newTarget = router(req);
  }

  return newTarget;
}

function getTargetFromProxyTable(req, table) {
  let result;
  const host = req.headers.host;
  const path = req.url;

  const hostAndPath = host + path;

  _.forIn(table, (value, key) => {
    if (containsPath(key)) {
      if (hostAndPath.indexOf(key) > -1) {
        // match 'localhost:3000/api'
        result = table[key];
        logger.debug('[HPM] Router table match: "%s"', key);
        return false;
      }
    } else {
      if (key === host) {
        // match 'localhost:3000'
        result = table[key];
        logger.debug('[HPM] Router table match: "%s"', host);
        return false;
      }
    }
  });

  return result;
}
```

模块允许我们在初始化的时候，传入 option.router，用来规定路由分发的逻辑，可以配置如下：

```js
router: {
    'integration.localhost:3000' : 'http://localhost:8001',  // host only
    'staging.localhost:3000'     : 'http://localhost:8002',  // host only
    'localhost:3000/api'         : 'http://localhost:8003',  // host + path
    '/rest'                      : 'http://localhost:8004'   // path only
}

// Custom router function
router: function(req) {
    return 'http://localhost:8004';
}
```

那么 getTarget 的逻辑就清晰了，就是根据配置，获取到新的 target。这就是 applyRouter 的能力，也就是路由分发的效果，它能指定最后的目标服务器到底应该请求哪个。

2. **applyPathRewrite**

```js
private applyPathRewrite = (req, pathRewriter) => {
  if (pathRewriter) {
    const path = pathRewriter(req.url, req);

    if (typeof path === 'string') {
      req.url = path;
    } else {
      this.logger.info(
        '[HPM] pathRewrite: No rewritten path found. (%s)',
        req.url
      );
    }
  }
};
```

applyPathRewrite 内部根据 `req.url` 调用 pathRewriter 来重写 url。先看下 pathRewriter 的逻辑。构造函数内部有这么一行代码：

```js
this.pathRewriter = PathRewriter.createPathRewriter(
  this.proxyOptions.pathRewrite
);
```

createPathRewriter 接收用户传入的 pathRewrite，生成一个新的 pathRewriter 函数，也就是在 applyPathRewrite 内部用到的。createPathRewriter 位于 `src/path-rewriter.ts`。

```js
export function createPathRewriter(rewriteConfig) {
  let rulesCache;

  if (!isValidRewriteConfig(rewriteConfig)) {
    return;
  }

  if (_.isFunction(rewriteConfig)) {
    const customRewriteFn = rewriteConfig;
    return customRewriteFn;
  } else {
    rulesCache = parsePathRewriteRules(rewriteConfig);
    return rewritePath;
  }

  function rewritePath(path) {
    let result = path;

    _.forEach(rulesCache, rule => {
      if (rule.regex.test(path)) {
        result = result.replace(rule.regex, rule.value);
        logger.debug('[HPM] Rewriting path from "%s" to "%s"', path, result);
        return false;
      }
    });

    return result;
  }
}
```

createPathRewriter 接收用户传入的 pathRewrite 配置项，如果该配置项是一个函数，那么就直接返回这个函数，大多数情况下允许配置为如下的一个对象：

```js
// rewrite path
pathRewrite: {'^/old/api' : '/new/api'}

// remove path
pathRewrite: {'^/remove/api' : ''}

// add base path
pathRewrite: {'^/' : '/basepath/'}

// custom rewriting
pathRewrite: function (path, req) { return path.replace('/api', '/base/api') }
```

那么就会走到 `parsePathRewriteRules` 的逻辑来生成一个 rulesCache 数组，并且返回 rewritePath 函数。 `parsePathRewriteRules` 的逻辑很简单：

```js
function parsePathRewriteRules(rewriteConfig) {
  const rules = [];

  if (_.isPlainObject(rewriteConfig)) {
    _.forIn(rewriteConfig, (value, key) => {
      rules.push({
        regex: new RegExp(key),
        value: rewriteConfig[key]
      });
      logger.info(
        '[HPM] Proxy rewrite rule created: "%s" ~> "%s"',
        key,
        rewriteConfig[key]
      );
    });
  }

  return rules;
}
```

rewriteConfig 是一个对象，最后返回的 rules 数组的结构如下:

```js
rules = [{
  regex: /^/old/api/
  value: '/new/api'
}]
```

绕了这么一小圈，也就是 `this.pathRewriter` 在用户传入的 pathRewrite 是一个 object，返回 rewritePath 函数，内部引用了 rules 数组。所以对于 `this.applyPathRewrite()` 的函数内部 `const path = pathRewriter(req.url, req);` 这一段代码，也就是将 `req.url` 传入 rewritePath 之后得到重写之后的 path。

```js
function rewritePath(path) {
  let result = path;

  _.forEach(rulesCache, rule => {
    if (rule.regex.test(path)) {
      result = result.replace(rule.regex, rule.value);
      logger.debug('[HPM] Rewriting path from "%s" to "%s"', path, result);
      return false;
    }
  });

  return result;
}
```

至此 prepareProxyRequest 的全部逻辑就已经执行完成，返回 newProxyOptions。最后就走到代理的过程了，这个时候所有的前期准备工作都做好了，实现了最核心的两个功能，**路由分发**以及**路径改写**。最后只需要调用 `node-http-proxy` 的 `web` 方法来开始真正的代理。

```js
this.proxy.web(req, res, activeProxyOptions);
```

## End

至此，所有的逻辑已经分析完毕，从源码的组织上来看，不管是逻辑的拆分，还是命名的语义化都是做的非常有效，学到了！
