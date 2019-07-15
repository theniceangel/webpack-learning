# sockjs-client

## 介绍

sockjs-client 是一个浏览器 JavaScript 库，提供类似WebSocket的对象。 SockJS为您提供了一个连贯的跨浏览器Javascript API，可在浏览器和Web服务器之间创建低延迟，全双工，跨域通信通道。默认使用 Websocket 作为 transport，否则会逐步降级使用如下的 transport：`xhr-streaming`、`xdr-streaming`、`eventsource`、`iframe-wrap-eventsource`、`htmlfile`、`iframe-wrap-htmlfile`、`xhr-polling`、`xdr-polling`、`iframe-wrap-xhr-polling`、`jsonp-polling`。所有的 transports 都在 `lib/transport-list.js` 文件可以查到。从这些降级的方案可以来看，兼容性做的非常的好，而且降级的方法的使用和你使用原生的 Websocket 对象没有多大的区别，库自己磨平了不同方案的差异。

## 流程

sockjs-client 与 server 大致的交互如下：

![流程图](https://raw.githubusercontent.com/theniceangel/webpack-learning/master/images/client-with-server.png)

sockjs-client 核心的步骤分为两步：

1. 获取到可用的 transportsList，并且发送 /info 请求先获取一定的信息，比如服务端是否支持 websocket，计算 client 与 server 通信的来返时间，具体可以查看 [sockjs-protocol](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26) 协议。
2. 建立真实的连接，会对 transportsList 队列支持的 transport 做一次遍历，优先取队头的 transport，否则会不断的往后取 transport 作为降级方案。

上面的大致流程，可以让你对源码有一个基础的轮廓，这样在阅读更细节的源码的时候，思维是不会被绕进去的，因为支持的 transport 非常多，所以代码比较长，但是只要心中有了上面那个图，你至少能大致了解 sockjs-client 干了什么。

## 具体细节

先从 `lib/entry.js` 入手，代码如下：

```js
var transportList = require('./transport-list');

module.exports = require('./main')(transportList);

if ('_sockjs_onload' in global) {
  setTimeout(global._sockjs_onload, 1);
}
```

得到所有的 transportList，传入 `./main` 模块。先看 `./transport-list`：

```js
module.exports = [
  // streaming transports
  require('./transport/websocket')
, require('./transport/xhr-streaming')
, require('./transport/xdr-streaming')
, require('./transport/eventsource')
, require('./transport/lib/iframe-wrap')(require('./transport/eventsource'))

  // polling transports
, require('./transport/htmlfile')
, require('./transport/lib/iframe-wrap')(require('./transport/htmlfile'))
, require('./transport/xhr-polling')
, require('./transport/xdr-polling')
, require('./transport/lib/iframe-wrap')(require('./transport/xhr-polling'))
, require('./transport/jsonp-polling')
];
```

transportList 可以看做是一个 transport 队列，优先取的是 websocket，次之是 stream transport，最后就是各种 polling。得到这些 transports 之后，传入 `.main` 导出的函数，代码如下：

```js
module.exports = function(availableTransports) {
  transports = transport(availableTransports);
  require('./iframe-bootstrap')(SockJS, availableTransports);
  return SockJS;
};
```

拿到 transportList 之后，调用 transport 方法，注意最后导出的是 `SockJs` 这个类，也就是用户需要用到的类。transport 位于 `lib/utils/transport` 模块，

```js
module.exports = function(availableTransports) {
  return {
    filterToEnabled: function(transportsWhitelist, info) {
      var transports = {
        main: []
      , facade: []
      };
      if (!transportsWhitelist) {
        transportsWhitelist = [];
      } else if (typeof transportsWhitelist === 'string') {
        transportsWhitelist = [transportsWhitelist];
      }

      availableTransports.forEach(function(trans) {
        if (!trans) {
          return;
        }

        if (trans.transportName === 'websocket' && info.websocket === false) {
          debug('disabled from server', 'websocket');
          return;
        }

        if (transportsWhitelist.length &&
            transportsWhitelist.indexOf(trans.transportName) === -1) {
          debug('not in whitelist', trans.transportName);
          return;
        }

        if (trans.enabled(info)) {
          debug('enabled', trans.transportName);
          transports.main.push(trans);
          if (trans.facadeTransport) {
            transports.facade.push(trans.facadeTransport);
          }
        } else {
          debug('disabled', trans.transportName);
        }
      });
      return transports;
    }
  };
};
```

transport 方法返回的是一个对象，提供了 filterToEnabled 方法来过滤可用的 transport。这个方法在接下来的 `sockjs._receiveInfo` 函数内部会使用，作用是获取有效的 transport 队列。

接下来的 `require('./iframe-bootstrap')(SockJS, availableTransports);` 这段代码很迷惑，查了 [sockjs-protocol IFrame page](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-15) 协议之后，才明白，这段 js 是为了服务器返回的 `iframe.html` 预留的口子。来看下代码：

```js
module.exports = function(SockJS, availableTransports) {
  var transportMap = {};
  availableTransports.forEach(function(at) {
    if (at.facadeTransport) {
      transportMap[at.facadeTransport.transportName] = at.facadeTransport;
    }
  });

  transportMap[InfoIframeReceiver.transportName] = InfoIframeReceiver;
  var parentOrigin;

  SockJS.bootstrap_iframe = function() {
    var facade;
    iframeUtils.currentWindowId = loc.hash.slice(1);
    var onMessage = function(e) {
      if (e.source !== parent) {
        return;
      }
      if (typeof parentOrigin === 'undefined') {
        parentOrigin = e.origin;
      }
      if (e.origin !== parentOrigin) {
        return;
      }

      var iframeMessage;
      try {
        iframeMessage = JSON3.parse(e.data);
      } catch (ignored) {
        debug('bad json', e.data);
        return;
      }

      if (iframeMessage.windowId !== iframeUtils.currentWindowId) {
        return;
      }
      switch (iframeMessage.type) {
      case 's':
        var p;
        try {
          p = JSON3.parse(iframeMessage.data);
        } catch (ignored) {
          debug('bad json', iframeMessage.data);
          break;
        }
        var version = p[0];
        var transport = p[1];
        var transUrl = p[2];
        var baseUrl = p[3];
        debug(version, transport, transUrl, baseUrl);
        // change this to semver logic
        if (version !== SockJS.version) {
          throw new Error('Incompatible SockJS! Main site uses:' +
                    ' "' + version + '", the iframe:' +
                    ' "' + SockJS.version + '".');
        }

        if (!urlUtils.isOriginEqual(transUrl, loc.href) ||
            !urlUtils.isOriginEqual(baseUrl, loc.href)) {
          throw new Error('Can\'t connect to different domain from within an ' +
                    'iframe. (' + loc.href + ', ' + transUrl + ', ' + baseUrl + ')');
        }
        facade = new FacadeJS(new transportMap[transport](transUrl, baseUrl));
        break;
      case 'm':
        facade._send(iframeMessage.data);
        break;
      case 'c':
        if (facade) {
          facade._close();
        }
        facade = null;
        break;
      }
    };

    eventUtils.attachEvent('message', onMessage);

    // Start
    iframeUtils.postMessage('s');
  };
};
```

根据协议的描述，客户端请求 `/iframe*.html` 接口，返回的就是一段 iframe html 字符串。这段字符串会调用 `SockJS.bootstrap_iframe` 方法，而这个方法就是为了向我们 parent window 发起 start 的信号

最后 `.main` 导出的函数返回 SockJs，也就是我们是用这个类去创建一个 sockjs-client 实例。那重点关注一下 SockJs 这个类的逻辑。先看构造函数：

```js
function SockJS(url, protocols, options) {
  if (!(this instanceof SockJS)) {
    return new SockJS(url, protocols, options);
  }
  if (arguments.length < 1) {
    throw new TypeError("Failed to construct 'SockJS: 1 argument required, but only 0 present");
  }
  EventTarget.call(this);

  this.readyState = SockJS.CONNECTING;
  this.extensions = '';
  this.protocol = '';

  // non-standard extension
  options = options || {};
  if (options.protocols_whitelist) {
    log.warn("'protocols_whitelist' is DEPRECATED. Use 'transports' instead.");
  }
  this._transportsWhitelist = options.transports;
  this._transportOptions = options.transportOptions || {};

  var sessionId = options.sessionId || 8;
  if (typeof sessionId === 'function') {
    this._generateSessionId = sessionId;
  } else if (typeof sessionId === 'number') {
    this._generateSessionId = function() {
      return random.string(sessionId);
    };
  } else {
    throw new TypeError('If sessionId is used in the options, it needs to be a number or a function.');
  }

  this._server = options.server || random.numberString(1000);

  // Step 1 of WS spec - parse and validate the url. Issue #8
  var parsedUrl = new URL(url);
  if (!parsedUrl.host || !parsedUrl.protocol) {
    throw new SyntaxError("The URL '" + url + "' is invalid");
  } else if (parsedUrl.hash) {
    throw new SyntaxError('The URL must not contain a fragment');
  } else if (parsedUrl.protocol !== 'http:' && parsedUrl.protocol !== 'https:') {
    throw new SyntaxError("The URL's scheme must be either 'http:' or 'https:'. '" + parsedUrl.protocol + "' is not allowed.");
  }

  var secure = parsedUrl.protocol === 'https:';
  // Step 2 - don't allow secure origin with an insecure protocol
  if (loc.protocol === 'https:' && !secure) {
    throw new Error('SecurityError: An insecure SockJS connection may not be initiated from a page loaded over HTTPS');
  }

  // Step 3 - check port access - no need here
  // Step 4 - parse protocols argument
  if (!protocols) {
    protocols = [];
  } else if (!Array.isArray(protocols)) {
    protocols = [protocols];
  }

  // Step 5 - check protocols argument
  var sortedProtocols = protocols.sort();
  sortedProtocols.forEach(function(proto, i) {
    if (!proto) {
      throw new SyntaxError("The protocols entry '" + proto + "' is invalid.");
    }
    if (i < (sortedProtocols.length - 1) && proto === sortedProtocols[i + 1]) {
      throw new SyntaxError("The protocols entry '" + proto + "' is duplicated.");
    }
  });

  // Step 6 - convert origin
  var o = urlUtils.getOrigin(loc.href);
  this._origin = o ? o.toLowerCase() : null;

  // remove the trailing slash
  parsedUrl.set('pathname', parsedUrl.pathname.replace(/\/+$/, ''));

  // store the sanitized url
  this.url = parsedUrl.href;
  debug('using url', this.url);

  // Step 7 - start connection in background
  // obtain server info
  // http://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26
  this._urlInfo = {
    nullOrigin: !browser.hasDomain()
  , sameOrigin: urlUtils.isOriginEqual(this.url, loc.href)
  , sameScheme: urlUtils.isSchemeEqual(this.url, loc.href)
  };

  this._ir = new InfoReceiver(this.url, this._urlInfo);
  this._ir.once('finish', this._receiveInfo.bind(this));
}

inherits(SockJS, EventTarget);
```

SockJs 的构造函数逻辑很简单，主要是对 options 的处理。比如 `transports`、`transportOptions`、`sessionId` 等等，[查看配置项](https://github.com/sockjs/sockjs-client#sockjs-client-api)。最关键的是 `this._ir = new InfoReceiver(this.url, this._urlInfo);` 这段代码。InfoReceiver 的作用是先向服务端发送 `/info` 接口来获取一些信息，这一步是必须的，这是 [sockjs-protocol](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26) 协议规定的一步。原因也在协议里面写的很清楚。那么先看下 `InfoReceiver` 的实现：

```js
function InfoReceiver(baseUrl, urlInfo) {
  debug(baseUrl);
  var self = this;
  EventEmitter.call(this);

  setTimeout(function() {
    self.doXhr(baseUrl, urlInfo);
  }, 0);
}

inherits(InfoReceiver, EventEmitter);


InfoReceiver._getReceiver = function(baseUrl, url, urlInfo) {
  // determine method of CORS support (if needed)
  if (urlInfo.sameOrigin) {
    return new InfoAjax(url, XHRLocal);
  }
  if (XHRCors.enabled) {
    return new InfoAjax(url, XHRCors);
  }
  if (XDR.enabled && urlInfo.sameScheme) {
    return new InfoAjax(url, XDR);
  }
  if (InfoIframe.enabled()) {
    return new InfoIframe(baseUrl, url);
  }
  return new InfoAjax(url, XHRFake);
};

InfoReceiver.prototype.doXhr = function(baseUrl, urlInfo) {
  var self = this
    , url = urlUtils.addPath(baseUrl, '/info')
    ;
  debug('doXhr', url);

  this.xo = InfoReceiver._getReceiver(baseUrl, url, urlInfo);

  this.timeoutRef = setTimeout(function() {
    debug('timeout');
    self._cleanup(false);
    self.emit('finish');
  }, InfoReceiver.timeout);

  this.xo.once('finish', function(info, rtt) {
    debug('finish', info, rtt);
    self._cleanup(true);
    self.emit('finish', info, rtt);
  });
};
```

InfoReceiver 构造函数会在下一帧调用 doXhr 方法。函数内部先拼接请求路径 `urlUtils.addPath(baseUrl, '/info')`。最后调用构造函数的静态属性 _getReceiver。_getReceiver 根据 urlInfo 来决定使用 InfoAjax 还是 InfoIframe，并且传入不同的 XHR 实现。

我们主要关注 InfoAjax 的代码：

```js
function InfoAjax(url, AjaxObject) {
  EventEmitter.call(this);

  var self = this;
  var t0 = +new Date();
  this.xo = new AjaxObject('GET', url);

  this.xo.once('finish', function(status, text) {
    var info, rtt;
    if (status === 200) {
      rtt = (+new Date()) - t0;
      if (text) {
        try {
          info = JSON3.parse(text);
        } catch (e) {
          debug('bad json', text);
        }
      }

      if (!objectUtils.isObject(info)) {
        info = {};
      }
    }
    self.emit('finish', info, rtt);
    self.removeAllListeners();
  });
}

inherits(InfoAjax, EventEmitter);
```

代码很简单，就是通过传入的 XHR 类来给服务端发送 `/info` 请求，这样拿到服务端返回的数据 info 之后，就派发 `finish` 事件通知给 InfoReceiver。拿 WebpackDevServer 的例子来看，http request 的路径和info 返回的数据结构如下：

```js
request = 'http://localhost:8080/sockjs-node/info?t=1563079675064'
info = {
  {"websocket":true,"origins":["*:*"],"cookie_needed":false,"entropy":3777137375}
}
```

那么至此，核心步骤的第一步就完成了，获取了 transportsList 并且发送 `/info` 请求拿到了服务端给出的信息 info。接下来的第二步，那么就是 connect 连接的问题了。一般服务端与客户端双向通信，首选的是 Websocket，次之就是 long polling 了。

既然 InfoReceiver 已经发送 `/info` 请求并且得到响应了，那么 SockJs 的最后一步就是下面这段代码了：

```js
function SockJS () {
// 省略....

this._ir.once('finish', this._receiveInfo.bind(this));
}


SockJS.prototype._receiveInfo = function(info, rtt) {
  debug('_receiveInfo', rtt);
  this._ir = null;
  if (!info) {
    this._close(1002, 'Cannot connect to server');
    return;
  }

  // establish a round-trip timeout (RTO) based on the
  // round-trip time (RTT)
  this._rto = this.countRTO(rtt);
  // allow server to override url used for the actual transport
  this._transUrl = info.base_url ? info.base_url : this.url;
  info = objectUtils.extend(info, this._urlInfo);
  debug('info', info);
  // determine list of desired and supported transports
  var enabledTransports = transports.filterToEnabled(this._transportsWhitelist, info);
  this._transports = enabledTransports.main;
  debug(this._transports.length + ' enabled transports');

  this._connect();
};

SockJS.prototype._connect = function() {
  for (var Transport = this._transports.shift(); Transport; Transport = this._transports.shift()) {
    debug('attempt', Transport.transportName);
    if (Transport.needBody) {
      if (!global.document.body ||
          (typeof global.document.readyState !== 'undefined' &&
            global.document.readyState !== 'complete' &&
            global.document.readyState !== 'interactive')) {
        debug('waiting for body');
        this._transports.unshift(Transport);
        eventUtils.attachEvent('load', this._connect.bind(this));
        return;
      }
    }

    // calculate timeout based on RTO and round trips. Default to 5s
    var timeoutMs = (this._rto * Transport.roundTrips) || 5000;
    this._transportTimeoutId = setTimeout(this._transportTimeout.bind(this), timeoutMs);
    debug('using timeout', timeoutMs);

    var transportUrl = urlUtils.addPath(this._transUrl, '/' + this._server + '/' + this._generateSessionId());
    var options = this._transportOptions[Transport.transportName];
    debug('transport url', transportUrl);
    var transportObj = new Transport(transportUrl, this._transUrl, options);
    transportObj.on('message', this._transportMessage.bind(this));
    transportObj.once('close', this._transportClose.bind(this));
    transportObj.transportName = Transport.transportName;
    this._transport = transportObj;

    return;
  }
  this._close(2000, 'All transports failed', false);
};
```

finish 的事件监听器内部调用 `_receiveInfo`，`_receiveInfo` 内部调用 `_connect`。而这就是真正开始利用 transport 来建立 client 与 server 的双向通信通道。

`_receiveInfo` 会调用 transports.filterToEnabled 方法来过滤出能用的 transport，还记得位于 `lib/main` 的这段代码么。

```js
module.exports = function(availableTransports) {
  transports = transport(availableTransports);
};
```

transports 就是 transport 函数返回的带有 filterToEnabled 方法的对象，并且在 SockJS.prototype._connect 方法里面用来过滤出有效的 transportList 队列，之后开始执行 _connect 连接。

_connect 方法里面会逐个使用 _transports 队列里面的 transport。`this._transportTimeoutId` 是用来做降级方案的，优先使用的是 Websocket。代码位于 `lib/transport/websocket.js` 文件，如果这个 WebsocketTransport 抛出了错误，那么就会执行到 `this._transportTimeoutId` 里面的 `this._transportTimeout.bind(this)`，里面会尝试调用 `this._connect` 进行降级 transport 的连接。拿到可用的 Tranport 之后，就开始实例化 Transport 与服务端建立双向通信。先看一下 WebSocketTransport 的实现。

1. **WebSocketTransport**

```js
function WebSocketTransport(transUrl, ignore, options) {
  if (!WebSocketTransport.enabled()) {
    throw new Error('Transport created when disabled');
  }

  EventEmitter.call(this);
  debug('constructor', transUrl);

  var self = this;
  var url = urlUtils.addPath(transUrl, '/websocket');
  if (url.slice(0, 5) === 'https') {
    url = 'wss' + url.slice(5);
  } else {
    url = 'ws' + url.slice(4);
  }
  this.url = url;

  this.ws = new WebsocketDriver(this.url, [], options);
  this.ws.onmessage = function(e) {
    debug('message event', e.data);
    self.emit('message', e.data);
  };
  this.unloadRef = utils.unloadAdd(function() {
    debug('unload');
    self.ws.close();
  });
  this.ws.onclose = function(e) {
    debug('close event', e.code, e.reason);
    self.emit('close', e.code, e.reason);
    self._cleanup();
  };
  this.ws.onerror = function(e) {
    debug('error event', e);
    self.emit('close', 1006, 'WebSocket connection broken');
    self._cleanup();
  };
}

inherits(WebSocketTransport, EventEmitter);
```

WebSocketTransport 函数内部先判断是否支持 WebSocketTransport，否则函数抛出错误，这样外层的 `this._transportTimeoutId` 就会执行，进行降级方案的 `_connect`。函数内部先拿到实例化 SockJs 的 url，替换成 `wss://` 或者 `ws://` 协议的连接，进行真正的客户端与服务端的双向通信。注意到 `new WebsocketDriver(this.url, [], options)` 这段代码中的 `WebsocketDriver`，你会好奇 `WebsocketDriver` 为什么是 `require('faye-websocket').Client` 的导出，`sockjs-client` 难道不是允许在浏览器端的吗？为啥不用原生的 Websocket，原因很简单，如果在 node 环境需要模拟浏览器的行为，那么 global 对象并没有 Websocket，翻看项目的 package.json，会看到如下的配置：

```js
"browser": {
  "./lib/transport/driver/websocket.js": "./lib/transport/browser/websocket.js",
  "eventsource": "./lib/transport/browser/eventsource.js",
  "./lib/transport/driver/xhr.js": "./lib/transport/browser/abstract-xhr.js",
  "crypto": "./lib/utils/browser-crypto.js",
  "events": "./lib/event/emitter.js"
},
```

这个代表，实际打包 `sockjs-client` 的时候，会打包 `./lib/transport/browser/websocket.js` 路径下导出的 websocket，导出的内容如下：

```js
var Driver = global.WebSocket || global.MozWebSocket;
if (Driver) {
	module.exports = function WebSocketBrowserDriver(url) {
		return new Driver(url);
	};
} else {
	module.exports = undefined;
}
```

从内容来看，取的就是 window 对象（global在打包之后会被替换成 window）上的 WebSocket 属性，包括 `./lib/transport/driver/xhr.js` 映射的也是 browser 版本的 `abstract-xhr`。总而言之，如果打包 browser 可用的版本的时候，会做一次替换。这样不管是 Websocket 还是 XHR 都是浏览器原生的，之所以会有两份，是因为可能会在 node 环境模拟浏览器的行为，所以需要额外的实现，Websocket 用的就是 `faye-websocket`这个库，XHR 用的就是 `http` 或者 `https` 自带的 `request` 方法。

`_connect` 方法执行完之后，客户端与服务端就以 Websocket 协议为基础，建立了双向通信通道。这只是最基础，也是最简单的一种双向通信的实现方式，`sockjs-client` 的代码之所以这么多，完全是因为为了兼容这些降级方案，这也是 [sockjs-procotol](https://sockjs.github.io/sockjs-protocol/sockjs-protocol-0.3.3.html#section-26) 的由来，再来分析一下一些降级方案。

2. **XhrStreamingTransport**

XhrStreamingTransport 通道的支持的前提是浏览器支持 XHR 的 readyState = 3，在这个状态下，能不断的拿到 Buffer 数据。[这里有个体验连接](http://www.debugtheweb.com/test/teststreaming.aspx)，能够感受到 XhrStreaming 对于用户体验上带来的增强。它的定义在 `lib/transport/xhr-streaming.js`。

```js
function XhrStreamingTransport(transUrl) {
  if (!XHRLocalObject.enabled && !XHRCorsObject.enabled) {
    throw new Error('Transport created when disabled');
  }
  AjaxBasedTransport.call(this, transUrl, '/xhr_streaming', XhrReceiver, XHRCorsObject);
}

inherits(XhrStreamingTransport, AjaxBasedTransport);

// lib/transport/lib/ajax-based.js 的 AjaxBasedTransport
function createAjaxSender(AjaxObject) {
  return function(url, payload, callback) {
    debug('create ajax sender', url, payload);
    var opt = {};
    if (typeof payload === 'string') {
      opt.headers = {'Content-type': 'text/plain'};
    }
    var ajaxUrl = urlUtils.addPath(url, '/xhr_send');
    var xo = new AjaxObject('POST', ajaxUrl, payload, opt);
    xo.once('finish', function(status) {
      debug('finish', status);
      xo = null;

      if (status !== 200 && status !== 204) {
        return callback(new Error('http status ' + status));
      }
      callback();
    });
    return function() {
      debug('abort');
      xo.close();
      xo = null;

      var err = new Error('Aborted');
      err.code = 1000;
      callback(err);
    };
  };
}
function AjaxBasedTransport(transUrl, urlSuffix, Receiver, AjaxObject) {
  SenderReceiver.call(this, transUrl, urlSuffix, createAjaxSender(AjaxObject), Receiver, AjaxObject);
}

// lib/transport/lib/sender-receiver.js 的 SenderReceiver
function SenderReceiver(transUrl, urlSuffix, senderFunc, Receiver, AjaxObject) {
  var pollUrl = urlUtils.addPath(transUrl, urlSuffix);
  debug(pollUrl);
  var self = this;
  BufferedSender.call(this, transUrl, senderFunc);

  this.poll = new Polling(Receiver, pollUrl, AjaxObject);
  this.poll.on('message', function(msg) {
    debug('poll message', msg);
    self.emit('message', msg);
  });
  this.poll.once('close', function(code, reason) {
    debug('poll close', code, reason);
    self.poll = null;
    self.emit('close', code, reason);
    self.close();
  });
}

inherits(SenderReceiver, BufferedSender);

// lib/transport/lib/buffered-sender.js 的 BufferedSender
function BufferedSender(url, sender) {
  debug(url);
  EventEmitter.call(this);
  this.sendBuffer = [];
  this.sender = sender;
  this.url = url;
}

inherits(BufferedSender, EventEmitter);

BufferedSender.prototype.send = function(message) {
  debug('send', message);
  this.sendBuffer.push(message);
  if (!this.sendStop) {
    this.sendSchedule();
  }
};

// lib/transport/lib/polling.js 的 Polling
function Polling(Receiver, receiveUrl, AjaxObject) {
  debug(receiveUrl);
  EventEmitter.call(this);
  this.Receiver = Receiver;
  this.receiveUrl = receiveUrl;
  this.AjaxObject = AjaxObject;
  this._scheduleReceiver();
}
```

整个链路特别长，但是只要你理解这个链路，所有其他的降级方案都是大同小异，先来个图来说明这些类的继承关系：

![流程图](https://raw.githubusercontent.com/theniceangel/webpack-learning/master/images/xhr-streaming-transport.png)

继承关系是这样的: XhrStreamingTransport -> AjaxBasedTransport -> SenderReceiver -> BufferedSender，SenderReceiver 依赖于 Receiver，BufferedSender 依赖于 createAjaxSender(AjaxObject)。

由此来看 XhrStreamingTransport 是继承了 BufferedSender 来向服务端发送信息，依赖于 Receiver 来向服务端保持long polling的链接。

BufferedSender 第二个参数是 senderFn，也就是 createAjaxSender(AjaxObject) 返回的闭包函数。代码如下：

```js
function createAjaxSender(AjaxObject) {
  return function(url, payload, callback) {
    debug('create ajax sender', url, payload);
    var opt = {};
    if (typeof payload === 'string') {
      opt.headers = {'Content-type': 'text/plain'};
    }
    var ajaxUrl = urlUtils.addPath(url, '/xhr_send');
    var xo = new AjaxObject('POST', ajaxUrl, payload, opt);
    xo.once('finish', function(status) {
      debug('finish', status);
      xo = null;

      if (status !== 200 && status !== 204) {
        return callback(new Error('http status ' + status));
      }
      callback();
    });
    return function() {
      debug('abort');
      xo.close();
      xo = null;

      var err = new Error('Aborted');
      err.code = 1000;
      callback(err);
    };
  };
}
```

其中 `var ajaxUrl = urlUtils.addPath(url, '/xhr_send');` 这一段代码的 url，就是真实向服务端 "write" 数据的接口地址。也就可以看出 BufferedSender 的作用，是客户端向服务端发送数据，而 `Receiver` 就是客户端用来服务端保持 long polling 的功能类，XhrStreamingTransport 用到的是 XhrReceiver，它位于 `lib/transport/receiver/xhr.js`，代码如下：

```js
function XhrReceiver(url, AjaxObject) {
  debug(url);
  EventEmitter.call(this);
  var self = this;

  this.bufferPosition = 0;

  this.xo = new AjaxObject('POST', url, null);
  this.xo.on('chunk', this._chunkHandler.bind(this));
  this.xo.once('finish', function(status, text) {
    debug('finish', status, text);
    self._chunkHandler(status, text);
    self.xo = null;
    var reason = status === 200 ? 'network' : 'permanent';
    debug('close', reason);
    self.emit('close', null, reason);
    self._cleanup();
  });
}

inherits(XhrReceiver, EventEmitter);
```

XhrReceiver 内部通过 AjaxObject，也就是 XHRCorsObject 来与服务端通信，而这个 url 就是以 `/xhr_streaming` 结尾的请求地址，等后面分析 `sockjs-node` 的时候就会知道 `/xhr_streaming` 用来保持长轮训， `/xhr_send` 是客户端用来向服务端发送信息，这样一来一回，就实现了双向通信。

关于 XhrStreamingTransport 再总结一下：

- AjaxBasedTransport 就是为了执行 createAjaxSender(AjaxObject)，生成 senderFn 给 BufferedSender 使用的，因为客户端真正给服务端 send message 是 BufferedSender 类提供的能力，请求地址类似于 `*/xhr_send`。
- SenderReceiver 是用来管理 Sender 以及 Receiver 类，它继承了 BufferedSender，并且 `this.poll` 属性是 `Polling` 类，这个类里面使用到了 `XhrReceiver`，这个类的作用为了客户端与服务端保持 long polling，请求地址类似于 `*/xhr_streaming`。服务端会 hang on 这次请求，直到服务端的 buffer 数据溢出的时候，会给客户端发送 buffer 数据，这样客户端 close 这次请求之后再发起下次请求。

经过这些精心的架构设计以及类的拆分，XhrStreamingTransport 就实现了客户端与服务端的双向通信通道。这些实现，都是基于 [sockjs-protocol](https://sockjs.github.io/sockjs-protocol)。

分析了两种 transport 之后，有点累，如果你自己对其他的 transport 的实现感兴趣，可以自己去钻研下。比如 `xdr-streaming`、`eventsource` 等等，其实原理都是类似的，只不过是为了兼容不同的浏览器。