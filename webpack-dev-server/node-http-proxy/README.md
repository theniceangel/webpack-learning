# node-http-proxy

在学习 webpack-dev-server 的实现过程中，发现内部依赖了 [http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware) 来实现代理，而 http-proxy-middleware 这个库是以 [node-http-proxy](https://github.com/nodejitsu/node-http-proxy) 为基础，进而提供一个中间件能力的类库。所以先从 node-http-proxy 底层库开始看起。

node-http-proxy 是一个支持 http 或者 websocket 代理的类库，有了它，可以轻松实现负载均衡、反向代理这些 feature。

先看一下基础的示例：

```js
var httpProxy = require('http-proxy');
var http = require('http')

var proxy = httpProxy.createProxyServer({})

// 代理服务器
http.createServer(function(req, res) {
  proxy.web(req, res, { target: 'http://mytarget.com:8080' });
}).listen(8000);
```

当你请求 `localhost:8000` 的时候，代理服务器会向目标服务器。即 `http://mytarget.com:8080` 发起请求，并将该服务器的响应返回给客户端。那么 `localhost:8000` 服务器就起到代理的作用。当然更简洁的启动代理服务器的方法如下：

```js
var httpProxy = require('http-proxy');
httpProxy.createServer({
  target: 'http://mytarget.com:8080'
}).listen(8000)
```

其实这种方法启动代理服务器和上面的代码差不多，源码对于 http-proxy 的 listen 方法做了封装而已，这个下面会分析。

先从入口 'lib/http-proxy.js' 开始：

```js
var ProxyServer = require('./http-proxy/index.js').Server;

function createProxyServer(options) {
  return new ProxyServer(options);
}

ProxyServer.createProxyServer = createProxyServer;
ProxyServer.createServer      = createProxyServer;
ProxyServer.createProxy       = createProxyServer;
```

入口模块到处导出 `ProxyServer` 函数，函数上面挂载了三个别名方法，都是指向 createProxyServer 工厂函数，返回的就是 `ProxyServer` 实例。

继续看 `ProxyServer` 的实现：

```js
EE3 = require('eventemitter3')
function ProxyServer(options) {
  EE3.call(this);

  options = options || {};
  options.prependPath = options.prependPath === false ? false : true;

  this.web = this.proxyRequest           = createRightProxy('web')(options);
  this.ws  = this.proxyWebsocketRequest  = createRightProxy('ws')(options);
  this.options = options;

  this.webPasses = Object.keys(web).map(function(pass) {
    return web[pass];
  });

  this.wsPasses = Object.keys(ws).map(function(pass) {
    return ws[pass];
  });

  this.on('error', this.onError, this);

}
require('util').inherits(ProxyServer, EE3);
```

`ProxyServer` 继承了 EventEmiiter。先关注下面的几行代码：

```js
this.web = this.proxyRequest           = createRightProxy('web')(options);
this.ws  = this.proxyWebsocketRequest  = createRightProxy('ws')(options);

function createRightProxy(type) {

  return function(options) {
    return function(req, res /*, [head], [opts] */) {
      // 先省略...
    };
  };
}
```

`this.web` 与 `this.ws` 其实就是 createRightProxy 函数内层的 `return function(req, res /*, [head], [opts] */)`。拿到开头的例子来说：

```js
http.createServer(function(req, res) {
  proxy.web(req, res, { target: 'http://mytarget.com:8080' });
}).listen(8000);
```

`proxy.web` 方法接收了 req、res、options 三个参数，也就是**真正执行代理**的过程，内部必须传入 req、res，并且封装了**任务队列**，即 this.webPasses 或 this.wsPasses，来细扒一下 `return function(req, res /*, [head], [opts] */)` 内部的逻辑。

```js
function createRightProxy(type) {

  return function(options) {
    return function(req, res /*, [head], [opts] */) {
      // 根据代理类型获取任务队列
      var passes = (type === 'ws') ? this.wsPasses : this.webPasses,
          args = [].slice.call(arguments),
          cntr = args.length - 1,
          head, cbl;

      // 如果最后一个参数为发生错误时的 callback
      if(typeof args[cntr] === 'function') {
        cbl = args[cntr];

        cntr--;
      }
      // 参数为配置对象，那么与实例化 Proxy 的 option 做一次 merge
      var requestOptions = options;
      if(
        !(args[cntr] instanceof Buffer) &&
        args[cntr] !== res
      ) {

        requestOptions = extend({}, options);

        extend(requestOptions, args[cntr]);

        cntr--;
      }

      if(args[cntr] instanceof Buffer) {
        head = args[cntr];
      }

      // 解析 target 以及 forward 配置项
      ['target', 'forward'].forEach(function(e) {
        if (typeof requestOptions[e] === 'string')
          requestOptions[e] = parse_url(requestOptions[e]);
      });

      // target 与 forward 必须存在一个
      if (!requestOptions.target && !requestOptions.forward) {
        return this.emit('error', new Error('Must provide a proper URL as target'));
      }

      // 执行任务队列
      for(var i=0; i < passes.length; i++) {
        if(passes[i](req, res, requestOptions, head, this, cbl)) {
          break;
        }
      }
    };
  };
}
```

内部的逻辑涵盖了**代理**的所有步骤，主要分为两部：

1. **解析配置项**
  允许灵活的入参，比如 options、errorCallback 等。
2. **执行任务队列**
  this.webPasses 或者 this.wsPasses 是一个包含所有任务的数组，每个任务各司其职，this.webPasses 的任务队列代码在 `lib/http-proxy/passes/web-incoming.js`，内部包含了 `deleteLength`、`timeout`、`XHeaders`、`stream` 这四个任务。

    - **deleteLength**

      ```js
      if((req.method === 'DELETE' || req.method === 'OPTIONS')
        && !req.headers['content-length']) {
        req.headers['content-length'] = '0';
        delete req.headers['transfer-encoding'];
      }
      ```

      处理 DELETE 或者 OPTIONS 请求。

    - **timeout**

      ```js
      function timeout(req, res, options) {
        if(options.timeout) {
          req.socket.setTimeout(options.timeout);
        }
      }
      ```

      设置超时时间。

    - **stream**

      ```js
      function stream(req, res, options, _, server, clb) {
        // 派发 start 事件，通知代理流程即将开始
        server.emit('start', req, res, options.target || options.forward);

        var agents = options.followRedirects ? followRedirects : nativeAgents;
        var http = agents.http;
        var https = agents.https;

        // 配置 forward，代理服务器仅仅将你的请求转发给目标服务器，并不会将目标服务器的 response 返回给客户端，这是 forward 与 target 最大的区别。
        if(options.forward) {
          // 向目标服务器发起请求
          var forwardReq = (options.forward.protocol === 'https:' ? https : http).request(
            common.setupOutgoing(options.ssl || {}, options, req, 'forward')
          );

          // error handler (e.g. ECONNRESET, ECONNREFUSED)
          // Handle errors on incoming request as well as it makes sense to
          var forwardError = createErrorHandler(forwardReq, options.forward);
          req.on('error', forwardError);
          forwardReq.on('error', forwardError);

          // pipe 的作用就是将客户端的请求数据输入到代理服务器的请求，内部会调用 forwardReq.end 开始代理服务器向目标服务器的请求。
          (options.buffer || req).pipe(forwardReq);
          // 如果未配置 target，那么直接响应客户端的请求。
          if(!options.target) { return res.end(); }
        }

        // 初始化代理服务器对目标服务器的请求
        var proxyReq = (options.target.protocol === 'https:' ? https : http).request(
          common.setupOutgoing(options.ssl || {}, options, req)
        );

        // 允许开发者在 headers 被发出前，能够改写 proxyReq。例如在 express 中，如果你在 body-parser 中间件之后才引入代理服务器，那么经过 body-parser 处理之后，req.body 的数据就被处理过了，那么 proxyReq 发送的 req.body 就有问题了，这个时候就需要监听 socket 事件，重新把 req.body 序列化一下。
        proxyReq.on('socket', function(socket) {
          if(server) { server.emit('proxyReq', proxyReq, req, res, options); }
        });

        // 设置代理服务器的超时时间
        if(options.proxyTimeout) {
          proxyReq.setTimeout(options.proxyTimeout, function() {
            proxyReq.abort();
          });
        }

        // Ensure we abort proxy if request is aborted
        req.on('aborted', function () {
          proxyReq.abort();
        });

        // 错误处理
        var proxyError = createErrorHandler(proxyReq, options.target);
        req.on('error', proxyError);
        proxyReq.on('error', proxyError);

        function createErrorHandler(proxyReq, url) {
          return function proxyError(err) {
            if (req.socket.destroyed && err.code === 'ECONNRESET') {
              server.emit('econnreset', err, req, res, url);
              return proxyReq.abort();
            }

            if (clb) {
              clb(err, req, res, url);
            } else {
              server.emit('error', err, req, res, url);
            }
          }
        }

        // 开始代理服务器的请求
        (options.buffer || req).pipe(proxyReq);

        // 目标服务器如果响应了代理服务器
        proxyReq.on('response', function(proxyRes) {
          if(server) { server.emit('proxyRes', proxyRes, req, res); }

          if(!res.headersSent && !options.selfHandleResponse) {
            for(var i=0; i < web_o.length; i++) {
              if(web_o[i](req, res, proxyRes, options)) { break; }
            }
          }

          if (!res.finished) {
            // Allow us to listen when the proxy has completed
            proxyRes.on('end', function () {
              if (server) server.emit('end', req, res, proxyRes);
            });
            // 代理服务器响应客户端请求
            if (!options.selfHandleResponse) proxyRes.pipe(res);
          } else {
            if (server) server.emit('end', req, res, proxyRes);
          }
        });
      }
      ```

      stream 流程可以说是这个库最核心的部分，代理服务器向目标服务器的 request 以及代理服务器给客户端的 response 都是发生在这个任务函数，stream 这个命名恰当好处，因为不管是配置了 forward 时，代理服务器转发客户端请求，还是配置 target 代理服务器响应客户端请求，通通都是通过 pipe 来实现的，而 pipe 就是 stream 的优势之一。


## 总结

所以从库的内部交互来看，代理服务器在客户端以及目标服务器之间充当一个媒介的作用。它能管理客户端发送给目标服务器的 request、以及目标服务器返回给客户端的 response。内部提供了很多灵活的配置项，来实现不同的代理定制化需求。看完这个库之后，那么是时候分析 [http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware) 这个库，看这个 middleware 到底是基于这个库做了哪些扩展。

[http-proxy-middleware 源码分析](https://github.com/theniceangel/webpack-learning/tree/master/webpack-dev-server/http-proxy-middleware)