# SockJs

webpackDevServer 3.7.1 版本是依赖 SockJs 来实现 hotReload、liveLoad 等能力。SockJs 提供了一套完整的客户端与服务端双向通信的实现，分别为 [sockjs-client](https://github.com/sockjs/sockjs-client)、[sockjs-node](https://github.com/sockjs/sockjs-node)，而这两个库的实现，都是基于自己提供的 [sockjs-protocol](https://github.com/sockjs/sockjs-protocol) 的协议之上，换句话说，如果想要知道客户端 sockjs 与 服务端 sockjs 的实现细节，必须知道这个协议的到底约定了什么？

这个协议的约定，会伴随着阅读两个库的源码而具体分析。

第一篇：[sockjs-client 的源码分析](https://github.com/theniceangel/webpack-learning/issues/7)
第二篇：[sockjs-node 的源码分析](https://github.com/theniceangel/webpack-learning/issues/8)
