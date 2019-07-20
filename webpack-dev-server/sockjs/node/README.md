# sockjs-node

看完了 sockjs-client 的源码之后，必须看 sockjs-node 的源码，因为它们是配套的，客户端的降级方案都必须在服务端的支持下才能得以实施。

## 提醒

目前 sockjs-node 的 npm 版本是 `0.3.19`，这个版本是用 `coffee` 写的，但是作者已经重构了 `0.4.0-rc.1` 版本，`index.js` 有一处 bug。并且 `examples` 目录下也是有一些 bug的，不过我自己跑示例代码的时候，修复了这些问题，因此这次分析的源码版本是 `0.4.0-rc.1`。

## 入口

`index.js` 的代码如下：

```js
const Server = require('./lib/server');

module.exports.createServer = function createServer(options) {
  return new Server(options);
};

module.exports.listen = function listen(http_server, options) {
  const srv = exports.createServer(options);
  if (http_server) {
    srv.installHandlers(http_server);
  }
  return srv;
};

```

导出 `listen` 与 `createServer` 两个方法，目前 `srv.installHandlers(http_server)` 这一处是有问题的，找了全部代码，都没发现有 `installHandlers` 方法。看了 `0.3.19` 的 dist 包才有这个方法。最新版的正确使用方式应该是这样的：

```js
const http = require('http');
const sockjs = require('sockjs');
const sockjs_opts = {
  prefix: '/echo'
};

const sockjs_echo = sockjs.createServer(sockjs_opts);
sockjs_echo.on('connection', function(conn) {
  conn.on('data', function(message) {
    conn.write(message);
  });
});

// 3. Usual http stuff
const server = http.createServer();

sockjs_echo.attach(server);
server.listen(9999, '0.0.0.0')
```

`createServer` 方法返回 sockjs_server 实例，再调用 attach 方法，将 sockjs_server 挂载到由 http 生成的 server 上。

因此所有的逻辑都集中在 `Server` 这个类上。

## Server

```js
class Server extends events.EventEmitter {
  constructor(user_options) {
    super();
    this.options = Object.assign(
      {
        prefix: '',
        transports: [
          'eventsource',
          'htmlfile',
          'jsonp-polling',
          'websocket',
          'websocket-raw',
          'xhr-polling',
          'xhr-streaming'
        ],
        response_limit: 128 * 1024,
        faye_server_options: null,
        jsessionid: false,
        heartbeat_delay: 25000,
        disconnect_delay: 5000,
        log() {},
        sockjs_url: 'https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js'
      },
      user_options
    );

    if (user_options.websocket === false) {
      const trs = new Set(this.options.transports);
      trs.delete('websocket');
      trs.delete('websocket-raw');
      this.options.transports = Array.from(trs.values());
    }

    this._prefixMatches = () => true;
    if (this.options.prefix) {
      this.options.prefix = this.options.prefix.replace(/\/$/, '');
      this._prefixMatches = requrl => url.parse(requrl).pathname.startsWith(this.options.prefix);
    }

    this.options.log(
      'debug',
      `SockJS v${pkg.version} bound to ${JSON.stringify(this.options.prefix)}`
    );
    this.handler = webjs.generateHandler(this, listener.generateDispatcher(this.options));
  }

  attach(server) {
    this._rlisteners = this._installListener(server, 'request');
    this._ulisteners = this._installListener(server, 'upgrade');
  }

  detach(server) {
    if (this._rlisteners) {
      this._removeListener(server, 'request', this._rlisteners);
      this._rlisteners = null;
    }
    if (this._ulisteners) {
      this._removeListener(server, 'upgrade', this._ulisteners);
      this._ulisteners = null;
    }
  }

  _removeListener(server, eventName, listeners) {
    server.removeListener(eventName, this.handler);
    listeners.forEach(l => server.on(eventName, l));
  }

  _installListener(server, eventName) {
    const listeners = server.listeners(eventName).filter(x => x !== this.handler);
    server.removeAllListeners(eventName);
    server.on(eventName, (req, res, head) => {
      if (this._prefixMatches(req.url)) {
        debug('prefix match', eventName, req.url, this.options.prefix);
        this.handler(req, res, head);
      } else {
        listeners.forEach(l => l.call(server, req, res, head));
      }
    });
    return listeners;
  }
}
```

先来看下 Server 的构造函数做了哪些事：

1. **合并 options**
内部先将用户传入的 options 与默认的合并。
    + `prefix`：客户端访问服务端的前缀 url，比如 webpackDevServer 的前缀地址就是 `/sockjs-node`。服务端会匹配 `req.url` 决定是否对这次请求做处理。
    + `transports`：允许的 transport 方案。
    + `response_limit`：长连接响应的 buffer 阈值，达到阈值之后，服务端会关闭这次长链接，并且将 buffer 都 flush 给客户端。
    + `faye_server_options`：[faye](https://github.com/faye/faye-websocket-node)的配置项，模块内部是依赖它来实现的 websocket。
    + `heartbeat_delay`：心跳包的发送频率，默认每隔 25s 发送一次心跳包。
    + `disconnect_delay`：session 连接的超时时间。
    + `log`：log 方法的实现。
    + `sockjs_url`：`sockjs-client` 的 CDN 地址。如果前端用到了 `iframe` 的降级方案，后端返回的 html 必须要引入 `sockjs-client` 脚本地址。也就是如下的模板里面的 `sockjs_url` 变量会被替换：

      ```html
      <!DOCTYPE html>
        <html>
        <head>
          <meta http-equiv="X-UA-Compatible" content="IE=edge" />
          <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
          <script src="{{ sockjs_url }}"></script>
          <script>
            document.domain = document.domain;
            SockJS.bootstrap_iframe();
          </script>
        </head>
        <body>
          <h2>Don't panic!</h2>
          <p>This is a SockJS hidden iframe. It's used for cross domain magic.</p>
        </body>
        </html>
      ```
2. **构建 handler**
  `listener.generateDispatcher(this.options)` 的返回值作为 `webjs.generateHandler` 的第二个参数，先来看下它的返回值是什么？

  ```js
  module.exports.generateDispatcher = function generateDispatcher(options) {
    const p = s => new RegExp(`^${options.prefix}${s}[/]?$`);
    const t = s => [p(`/([^/.]+)/([^/.]+)${s}`), 'server', 'session'];
    const prefix_dispatcher = [
      ['GET', p(''), [handlers.welcome_screen]],
      ['OPTIONS', p('/info'), [middleware.h_sid, middleware.xhr_cors, info.info_options]],
      ['GET', p('/info'), [middleware.xhr_cors, middleware.h_no_cache, info.info]]
    ];
    if (!options.disable_cors) {
      prefix_dispatcher.push(['GET', p('/iframe[0-9-.a-z_]*.html'), [iframe.iframe]]);
    }

    const transport_dispatcher = [];

    for (const name of options.transports) {
      const tr = transportList[name];
      if (!tr) {
        throw new Error(`unknown transport ${name}`);
      }
      debug('enabling transport', name);

      for (const route of tr.routes) {
        const d = route.transport ? transport_dispatcher : prefix_dispatcher;
        const path = route.transport ? t(route.path) : p(route.path);
        const fullroute = [route.method, path, route.handlers];
        if (!d.some(x => x[0] == route.method && x[1].toString() === path.toString())) {
          d.push(fullroute);
        }
      }
    }
    return prefix_dispatcher.concat(transport_dispatcher);
  };
  ```

  首先看函数的返回，是 prefix_dispatcher 数组 concat 了 transport_dispatcher 数组。prefix_dispatcher 是一个二维数组，并且带有默认的三条数据，数组里面的数据结构如下：

  ```js
  [
    method, // http 请求方法
    path, // 正则或者 [正则, 'server', 'session'] 的数据结构
    handlers // [middleware] 中间件链
  ]
  ```

  也就是说 `generateDispatcher` 函数返回的是带有特定数据结构的数组，暂且称之为 dispatchers，传给 `webjs.generateHandler` 作为第二个参数。`generateHandler` 的逻辑如下：

  ```js
  module.exports.generateHandler = function generateHandler(server, dispatcher) {
    return function(req, res, head) {
      if (res.writeHead === undefined) {
        utils.fake_response(req, res);
      }
      const parsedUrl = url.parse(req.url, true);
      req.pathname = parsedUrl.pathname || '';
      req.query = parsedUrl.query;
      req.start_date = Date.now();

      let found = false;
      const allowed_methods = [];
      for (const row of dispatcher) {
        let [method, path, funs] = row;
        if (!Array.isArray(path)) {
          path = [path];
        }
        // path[0] must be a regexp
        const m = req.pathname.match(path[0]);
        if (!m) {
          continue;
        }
        if (req.method !== method) {
          allowed_methods.push(method);
          continue;
        }
        for (let i = 1; i < path.length; i++) {
          req[path[i]] = m[i];
        }
        funs = funs.slice(0);
        funs.push(middleware.log_request);
        execute_async_request(server, funs, req, res, head);
        found = true;
        break;
      }

      if (!found) {
        if (allowed_methods.length !== 0) {
          handlers.handle_405.call(server, req, res, allowed_methods);
        } else {
          handlers.handle_404.call(server, req, res);
        }
        middleware.log_request.call(server, req, res, true, () => {});
      }
    };
  };
  ```

  generateHandler 函数返回的也是一个中间件，最后 `this.handler` 被赋值为 generateHandler 函数返回的中间件。细品一下这个中间件的逻辑。


  首先拿到跟请求 url 有关的信息，比如 pathname、query。接着开始 `dispatcher` 的遍历，根据请求方法与地址来匹配对应的 handlers，我们可以把它理解为对应 transport 的中间件链。如果所有的规则都没匹配上，那么就走到 `if (!found)` 的逻辑，返回 404 或者 405.

  单独的把 dispatch 的迭代拿出来讲一下：

  ```js
  for (const row of dispatcher) {
    let [method, path, funs] = row;
    if (!Array.isArray(path)) {
      path = [path];
    }
    // path[0] must be a regexp
    const m = req.pathname.match(path[0]);
    if (!m) {
      continue;
    }
    if (req.method !== method) {
      allowed_methods.push(method);
      continue;
    }
    // 通过正则捕获组拿到 session 以及 server 对应的值，并放在 req 上
    for (let i = 1; i < path.length; i++) {
      req[path[i]] = m[i];
    }
    funs = funs.slice(0);
    funs.push(middleware.log_request);
    execute_async_request(server, funs, req, res, head);
    found = true;
    break;
  }
  ```

  逻辑很清晰，先通过请求路径过滤，再通过请求方法过滤，接着通过 `t = s => [正则, 'server', 'session']` 里面的正则捕获组拿到客户端连接的 server 以及 session 的字符串。比如 `webpackDevServer` 的 websocket 请求地址是这样的：`ws://localhost:8080/sockjs-node/078/vfwmtb20/websocket`。那么 `server` 就是 "078"，`session` 就是 "vfwmtb20"。**为什么要拿到 session 呢？**，是因为对于 long polling 这种方案来说，每一个请求一般是先被服务端挂起，直到缓存区的 buffer 要推送给客户端了，才会断开连接，客户端监听 close 事件之后，发起下次请求 request。而这一次的 request 和 上一次的 session 和 server 都是不变的，这样服务端就能通过 `session` 去缓存一些信息在内存当中。而在 `session.js` 当中，就有 map 结构来存同一个请求的 session。

  ```js
  const MAP = new Map();
  ```

  扯得有点远了。接下来再看 `execute_async_request`，这是一个很经典的异步任务队列的实现方式，主要的思想是递归。来看下 `execute_async_request` 的实现：

  ```js
  function execute_async_request(server, funs, req, res, head) {
    function next(err) {
      if (err) {
        if (err.status) {
          const handlerName = `handle_${err.status}`;
          if (handlers[handlerName]) {
            return handlers[handlerName].call(server, req, res, err);
          }
        }
        return handlers.handle_error.call(server, err, req, res);
      }
      if (!funs.length) {
        return;
      }
      const fun = funs.shift();
      debug('call', fun);
      fun.call(server, req, res, head, next);
    }
    next();
  }
  ```

  funs 就是不同 transport 导出的 route 的 handlers 属性，它是一个中间件链。每个中间件的最后一个参数是 next 函数，这样只要在中间件调用了 `next()`，又能走到下一个中间件的逻辑了。

  对于每一个 dispatcher 来说，都有自己的 handlers。因为 dispatcher 太多，只对几个经典的分析。

## 请求分析

假设我们在本地起了如下的一个 sockjs-node 的 server。

```js
const http = require('http');
const express = require('express');
const sockjs = require('sockjs');

const sockjs_opts = {
  prefix: '/echo'
};

const sockjs_echo = sockjs.createServer(sockjs_opts);
sockjs_echo.on('connection', conn => {
  conn.on('data', msg => conn.write(msg));
});

const app = express();
app.get('/', (req, res) => res.sendFile(__dirname + '/index.html'));

const server = http.createServer(app);
sockjs_echo.attach(server);

server.listen(9999, '0.0.0.0', () => {
  console.log(' [*] Listening on 0.0.0.0:9999');
});
```

prefix 是 `/echo`，也就是说所有以 `/echo` 开头的请求才会被 sockjs-node 处理。

1. **默认请求**

    假如我们发起了如下的请求：`http://localhost:9999/echo`，那么上面 `for (const row of dispatcher)` 的迭代命中的 row 就是 ['GET', /^\/echo[\/]?$/, [welcome_screen]]，所以走到 welcome_screen 函数，它在 `lib/handlers.js` 定义。

    ```js
    res.setHeader('Content-Type', 'text/plain; charset=UTF-8');
    res.writeHead(200);
    res.end('Welcome to SockJS!\n');
    ```

    作用就是返回 `Welcome to SockJS!\n`。

2. **/info 请求**

    假如我们发起了如下的请求：`http://localhost:9999/echo/info`，这个请求是客户端在与服务端连接之前，返回一些基本信息的。[sockjs-protocol](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26) 定义了这条规则。那么上面 `for (const row of dispatcher)` 的迭代命中的 row 就是 `['GET', /^\/echo\/info[\/]?$/, [xhr_cors, h_no_cache, info]]`，所以会走到 `xhr_cors`、`h_no_cache`、`info` 三个中间件。

    ```js
    // middleware.js
    xhr_cors(req, res, _head, next) {
      if (this.options.disable_cors) {
        return next();
      }

      let origin;
      if (!req.headers['origin']) {
        origin = '*';
      } else {
        origin = req.headers['origin'];
        res.setHeader('Access-Control-Allow-Credentials', 'true');
      }
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Vary', 'Origin');
      const headers = req.headers['access-control-request-headers'];
      if (headers) {
        res.setHeader('Access-Control-Allow-Headers', headers);
      }
      next();
    }

    // middleware.js
    h_no_cache(req, res, _head, next) {
      res.setHeader('Cache-Control', 'no-store, no-cache, no-transform, must-revalidate, max-age=0');
      next();
    },

    // info.js
    info(req, res) {
      const info = {
        websocket: this.options.transports.includes('websocket'),
        transports: this.options.transports,
        origins: this.options.disable_cors ? undefined : ['*:*'],
        cookie_needed: !!this.options.jsessionid,
        entropy: utils.random32()
      };

      if (typeof this.options.base_url === 'function') {
        info.base_url = this.options.base_url();
      } else if (this.options.base_url) {
        info.base_url = this.options.base_url;
      }
      res.setHeader('Content-Type', 'application/json; charset=UTF-8');
      res.writeHead(200);
      res.end(JSON.stringify(info));
    }
    ```

    `xhr_cors` 的作用是用来设置与跨域相关的一些响应头。`h_no_cache` 是设置请求不被缓存。最后交由 `info` 中间件来返回数据，结构如下：

    ```js
    const info = {
      websocket: Boolean,
      transports: String [],
      origins?: ['*:*'],
      cookie_needed: Boolean,
      entropy: Number // 32位
    };
    ```

    那么 sockjs-client 拿到这些信息之后就能做响应的处理，换句话来说，服务端也能通过 /info 接口返回数据来控制客户端的行为，正如上面 [sockjs-protocol](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26) 描述的。

3. **/websocket 请求**

    `http://localhost:9999/echo/info` 请求返回之后，`sockjs-client` 会调用 `_connect` 方法，开始真正的选择一种 tranport 来进行通信，首先是选择 Websocket 方案。如果 Websocket transport 会先给服务端发送 `ws://localhost:9999/echo/852/vcpouwun/websocket` 请求来升级协议，其中 `852` 和 `vcpouwun` 都是随机生成的，服务端会用到他们用在 session 连接上。那么上面 `for (const row of dispatcher)` 的迭代命中的 row 就是 `['GET', [
      '/^\/echo\/([^\/.]+)\/([^\/.]+)\/websocket[\/]?$/',
      'server',
      'session'
    ], [websocket_check, sockjs_websocket]]`，所以会走到 `websocket_check`、`sockjs_websocket` 三个中间件，并且 `req.session` 就是 `vcpouwun`，`req.server` 就是 `852`

    ```js
    // middleware.js
    websocket_check(req, _socket, _head, next) {
      if (!FayeWebsocket.isWebSocket(req)) {
        return next({
          status: 400,
          message: 'Not a valid websocket request'
        });
      }
      next();
    },

    // lib/transport/websocket.js
    function sockjs_websocket(req, socket, head, next) {
      const ws = new FayeWebsocket(req, socket, head, null, this.options.faye_server_options);
      ws.once('open', () => {
        // websockets possess no session_id
        Session.registerNoSession(req, this, new WebSocketReceiver(ws, socket));
      });
      next();
    }
    ```

    `websocket_check` 是校验这次请求是不是合法 websocket 请求。

    `sockjs_websocket` 函数的作用是实例化一个 websocket 服务器，内部使用的是 [faye-websocket](https://github.com/faye/faye-websocket-node)，这是一个比较标准的 Websocket 实现，并且通过 `Session.registerNoSession` 方法将这次请求加入 session 当中。Session 类抽象了客户端与服务端请求连接的行为，在客户端发起请求之后，服务端是通过 Session 实例来记录这次请求的行为。`registerNoSession` 方法接收三个参数，一个是 httpServer 的 `req`，第二个是 `sockjs-node` 的实例，第三个是 WebSocketReceiver 实例。先看下 `WebSocketReceiver` 的定义。

    ```js
    class WebSocketReceiver extends BaseReceiver {
      constructor(ws, socket) {
        super(socket);
        debug('new connection');
        this.protocol = 'websocket';
        this.ws = ws;
        try {
          socket.setKeepAlive(true, 5000);
        } catch (x) {
          // intentionally empty
        }
        this.ws.once('close', this.abort);
        this.ws.on('message', m => this.didMessage(m.data));
        this.heartbeatTimeout = this.heartbeatTimeout.bind(this);
      }

      tearDown() {
        if (this.ws) {
          this.ws.removeEventListener('close', this.abort);
        }
        super.tearDown();
      }

      didMessage(payload) {
        debug('message');
        if (this.ws && this.session && payload.length > 0) {
          let message;
          try {
            message = JSON.parse(payload);
          } catch (x) {
            return this.close(3000, 'Broken framing.');
          }
          if (payload[0] === '[') {
            message.forEach(msg => this.session.didMessage(msg));
          } else {
            this.session.didMessage(message);
          }
        }
      }

      sendFrame(payload) {
        debug('send');
        if (this.ws) {
          try {
            this.ws.send(payload);
            return true;
          } catch (x) {
            // intentionally empty
          }
        }
        return false;
      }

      close(status = 1000, reason = 'Normal closure') {
        super.close(status, reason);
        if (this.ws) {
          try {
            this.ws.close(status, reason, false);
          } catch (x) {
            // intentionally empty
          }
        }
        this.ws = null;
      }

      heartbeat() {
        const supportsHeartbeats = this.ws.ping(null, () => clearTimeout(this.hto_ref));

        if (supportsHeartbeats) {
          this.hto_ref = setTimeout(this.heartbeatTimeout, 10000);
        } else {
          super.heartbeat();
        }
      }

      heartbeatTimeout() {
        if (this.session) {
          this.session.close(3000, 'No response from heartbeat');
        }
      }
    }

    // BaseReceiver
    class BaseReceiver {
      constructor(socket) {
        this.abort = this.abort.bind(this);
        this.socket = socket;
        this.socket.on('close', this.abort);
        this.socket.on('end', this.abort);
      }

      tearDown() {
        if (!this.socket) {
          return;
        }
        debug('tearDown', this.session && this.session.id);
        this.socket.removeListener('close', this.abort);
        this.socket.removeListener('end', this.abort);
        this.socket = null;
      }

      abort() {
        debug('abort', this.session && this.session.id);
        this.delay_disconnect = false;
        this.close();
      }

      close() {
        debug('close', this.session && this.session.id);
        this.tearDown();
        if (this.session) {
          this.session.unregister();
        }
      }

      sendBulk(messages) {
        const q_msgs = messages.map(m => utils.quote(m)).join(',');
        return this.sendFrame(`a[${q_msgs}]`);
      }

      heartbeat() {
        return this.sendFrame('h');
      }
    }
    ```

    WebSocketReceiver 继承于 BaseReceiver 类，BaseReceiver 是管理服务端向客户端推送消息。不管是 `sendBulk` 还是 `heartbeat` 都是调用 `sendFrame` 来给客户端推送信息。继承了 BaseReceiver 类的子类都应该事先 `sendFrame` 方法，就如上面 WebSocketReceiver 的 `sendFrame` 函数内部就是拿到 `faye-websocket` 这个实例并且调用 send 方法，这个时候客户端就能接收到消息了。

    再回到 `Session.registerNoSession`

    ```js
    const MAP = new Map();

    class Session {
      static bySessionId(session_id) {
        if (!session_id) {
          return null;
        }
        return MAP.get(session_id) || null;
      }

      static _register(req, server, session_id, receiver) {
        let session = Session.bySessionId(session_id);
        if (!session) {
          debug('create new session', session_id);
          session = new Session(session_id, server);
        }
        session.register(req, receiver);
        return session;
      }

      static register(req, server, receiver) {
        debug('static register', req.session);
        return Session._register(req, server, req.session, receiver);
      }

      static registerNoSession(req, server, receiver) {
        debug('static registerNoSession');
        return Session._register(req, server, undefined, receiver);
      }

      constructor(session_id, server) {
        this.session_id = session_id;
        this.heartbeat_delay = server.options.heartbeat_delay;
        this.disconnect_delay = server.options.disconnect_delay;
        this.prefix = server.options.prefix;
        this.send_buffer = [];
        this.is_closing = false;
        this.readyState = Transport.CONNECTING;
        debug('readyState', 'CONNECTING', this.session_id);
        if (this.session_id) {
          MAP.set(this.session_id, this);
        }
        this.didTimeout = this.didTimeout.bind(this);
        this.to_tref = setTimeout(this.didTimeout, this.disconnect_delay);
        this.connection = new SockJSConnection(this);
        this.emit_open = () => {
          this.emit_open = null;
          server.emit('connection', this.connection);
        };
      }

      get id() {
        return this.session_id;
      }

      register(req, recv) {
        if (this.recv) {
          recv.sendFrame(closeFrame(2010, 'Another connection still open'));
          recv.close();
          return;
        }
        if (this.to_tref) {
          clearTimeout(this.to_tref);
          this.to_tref = null;
        }
        if (this.readyState === Transport.CLOSING) {
          this.flushToRecv(recv);
          recv.sendFrame(this.close_frame);
          recv.close();
          this.to_tref = setTimeout(this.didTimeout, this.disconnect_delay);
          return;
        }
        // Registering. From now on 'unregister' is responsible for
        // setting the timer.
        this.recv = recv;
        this.recv.session = this;

        // Save parameters from request
        this.decorateConnection(req);

        // first, send the open frame
        if (this.readyState === Transport.CONNECTING) {
          this.recv.sendFrame('o');
          this.readyState = Transport.OPEN;
          debug('readyState', 'OPEN', this.session_id);
          // Emit the open event, but not right now
          process.nextTick(this.emit_open);
        }

        // At this point the transport might have gotten away (jsonp).
        if (!this.recv) {
          return;
        }
        this.tryFlush();
      }

      decorateConnection(req) {
        Session.decorateConnection(req, this.connection, this.recv);
      }

      static decorateConnection(req, connection, recv) {
        let socket = recv.socket;
        if (!socket) {
          socket = recv.response.socket;
        }
        // Store the last known address.
        let remoteAddress, remotePort, address;
        try {
          remoteAddress = socket.remoteAddress;
          remotePort = socket.remotePort;
          address = socket.address();
        } catch (x) {
          // intentionally empty
        }

        if (remoteAddress) {
          // All-or-nothing
          connection.remoteAddress = remoteAddress;
          connection.remotePort = remotePort;
          connection.address = address;
        }

        connection.url = req.url;
        connection.pathname = req.pathname;
        connection.protocol = recv.protocol;

        const headers = {};
        const allowedHeaders = [
          'referer',
          'x-client-ip',
          'x-forwarded-for',
          'x-cluster-client-ip',
          'via',
          'x-real-ip',
          'x-forwarded-proto',
          'x-ssl',
          'dnt',
          'host',
          'user-agent',
          'accept-language'
        ];
        for (const key of allowedHeaders) {
          if (req.headers[key]) {
            headers[key] = req.headers[key];
          }
        }

        if (headers) {
          connection.headers = headers;
        }
      }

      unregister() {
        debug('unregister', this.session_id);
        const delay = this.recv.delay_disconnect;
        this.recv.session = null;
        this.recv = null;
        if (this.to_tref) {
          clearTimeout(this.to_tref);
        }

        if (delay) {
          debug('delay timeout', this.session_id);
          this.to_tref = setTimeout(this.didTimeout, this.disconnect_delay);
        } else {
          debug('immediate timeout', this.session_id);
          this.didTimeout();
        }
      }

      flushToRecv(recv) {
        if (this.send_buffer.length > 0) {
          const sb = this.send_buffer;
          this.send_buffer = [];
          recv.sendBulk(sb);
          return true;
        }
        return false;
      }

      tryFlush() {
        if (!this.flushToRecv(this.recv) || !this.to_tref) {
          if (this.to_tref) {
            clearTimeout(this.to_tref);
          }
          const x = () => {
            if (this.recv) {
              this.to_tref = setTimeout(x, this.heartbeat_delay);
              this.recv.heartbeat();
            }
          };
          this.to_tref = setTimeout(x, this.heartbeat_delay);
        }
      }

      didTimeout() {
        if (this.to_tref) {
          clearTimeout(this.to_tref);
          this.to_tref = null;
        }
        if (
          this.readyState !== Transport.CONNECTING &&
          this.readyState !== Transport.OPEN &&
          this.readyState !== Transport.CLOSING
        ) {
          throw new Error('INVALID_STATE_ERR');
        }
        if (this.recv) {
          throw new Error('RECV_STILL_THERE');
        }
        debug('readyState', 'CLOSED', this.session_id);
        this.readyState = Transport.CLOSED;
        this.connection.push(null);
        this.connection = null;
        if (this.session_id) {
          MAP.delete(this.session_id);
          debug('delete session', this.session_id, MAP.size);
          this.session_id = null;
        }
      }

      didMessage(payload) {
        if (this.readyState === Transport.OPEN) {
          this.connection.push(payload);
        }
      }

      send(payload) {
        if (this.readyState !== Transport.OPEN) {
          return false;
        }
        this.send_buffer.push(payload);
        if (this.recv) {
          this.tryFlush();
        }
        return true;
      }

      close(status = 1000, reason = 'Normal closure') {
        debug('close', status, reason);
        if (this.readyState !== Transport.OPEN) {
          return false;
        }
        this.readyState = Transport.CLOSING;
        debug('readyState', 'CLOSING', this.session_id);
        this.close_frame = closeFrame(status, reason);
        if (this.recv) {
          // Go away. sendFrame can trigger close which can
          // trigger unregister. Make sure this.recv is not null.
          this.recv.sendFrame(this.close_frame);
          if (this.recv) {
            this.recv.close();
          }
          if (this.recv) {
            this.unregister();
          }
        }
        return true;
      }
    }
    ```

    `session.js` 模块内部有个 MAP 对象来保存所有请求的 session 实例，它 的 key 就是上文说到的 `vcpouwun`。

    Session 类上的静态方法 `registerNoSession` 内部会调用静态方法 `_register` 并且第三个参数传 undefined，也就是对于 websocket 来说，请求没必要缓存在 Map 当中，最后会走到 `session = new Session(session_id, server);` 新创建一个 session。

    ```js
    constructor(session_id, server) {
      this.session_id = session_id;
      this.heartbeat_delay = server.options.heartbeat_delay;
      this.disconnect_delay = server.options.disconnect_delay;
      this.prefix = server.options.prefix;
      this.send_buffer = [];
      this.is_closing = false;
      this.readyState = Transport.CONNECTING;
      debug('readyState', 'CONNECTING', this.session_id);
      if (this.session_id) {
        MAP.set(this.session_id, this);
      }
      this.didTimeout = this.didTimeout.bind(this);
      this.to_tref = setTimeout(this.didTimeout, this.disconnect_delay);
      this.connection = new SockJSConnection(this);
      this.emit_open = () => {
        this.emit_open = null;
        server.emit('connection', this.connection);
      };
    }
    ```

    session 上有很多属性与方法，`heartbeat_delay` 表示服务端给客户端发心跳包的间隔，`disconnect_delay` 表示连接超时时间，`send_buffer` 用来缓存服务端给客户端的消息，在调用 `tryFlush` 的时候 `flushToRecv`，在 `flushToRecv` 的内部是获取到对应的 Receiver 实例，比如 Websocket 请求就是 WebSocketReceiver 实例，调用 `sendBulk` 最后调用 `sendFrame` 来将消息 flush 给客户端。

    `SockJSConnection` 类是抽象出来的连接类，并且作为第三方使用者，监听 `sockjs-node` 的 connection 事件之后，回调函数的第一个参数是 SockJSConnection 实例.

    ```js
    const sockjs_echo = sockjs.createServer(sockjs_opts);

    sockjs_echo.on('connection', function(conn) {
      conn.on('data', function(message) {
        conn.write(message);
      });
    });
    ```

    上述的 conn 就是 SockJSConnection 实例。调用 `conn.write(message)` 就是给客户端推送消息。SockJSConnection 的实现如下：

    ```js
    class SockJSConnection extends stream.Duplex {
      constructor(session) {
        super({ decodeStrings: false, encoding: 'utf8', readableObjectMode: true });
        this._session = session;
        this.id = uuid();
        this.headers = {};
        this.prefix = this._session.prefix;
        debug('new connection', this.id, this.prefix);
      }

      toString() {
        return `<SockJSConnection ${this.id}>`;
      }

      _write(chunk, encoding, callback) {
        if (Buffer.isBuffer(chunk)) {
          chunk = chunk.toString();
        }
        this._session.send(chunk);
        callback();
      }

      _read() {}

      end(chunk, encoding, callback) {
        super.end(chunk, encoding, callback);
        this.close();
      }

      close(code, reason) {
        debug('close', code, reason);
        return this._session.close(code, reason);
      }

      get readyState() {
        return this._session.readyState;
      }
    ```

    SockJSConnection 继承了双工流，如果你执行 `conn.write(msg)`，会走到 `this._write()`，在重写的 _write 方法里面，会调用 `  this._session.send(chunk)`，send 方法里面会调用 `tryFlush` 这样便可以给客户端发送消息。


## 总结

上述只是分析了一下 Websocket tranport 的方式。那么理一下 `sockjs-node` 与 `sockjs-client` 的整体交互流程。

1. client 先发送一个 `/info` 请求给 server，来获取 info 信息。
2. 接着 client 内部调用 `_connect` 方法建立 `Websocket` 连接，请求的格式类似于 `/{prefix}/{server}/{session}/websocket`
3. 服务端接收请求之后，升级为 Websocket 协议，接着实例化 WebSocketReceiver，内部会实例化 `faye_websocket`，并且传给 Session，Session 上会保存着 SockJSConnection 实例，并且 server 会触发 `connection` 事件将 sockJSConnection 作为参数传给回调函数，那么作为使用方就能通过 sockJSConnection 实例，调用 write 方法触发 `session.send` 进而沟通 `websocket` 给客户端推送消息。