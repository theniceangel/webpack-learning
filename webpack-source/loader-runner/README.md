# LoaderRunner

绝大多数人都在使用 webpack 作为构建工具。那么 `loader` 作为处理各种资源的工具，大家肯定也不会陌生。很多人没写过 loader，但是都对 loader 的具体怎么写，怎样执行的一无所知。那么本文就对 `3.0.0` 版本做一个全方位的揭秘。

## loader

所谓 loader 只是一个导出为函数的 JavaScript 模块。它接收上一个 loader 产生的结果或者资源文件(resource file)作为入参。也可以用多个 loader 函数组成 loader chain。compiler 需要得到最后一个 loader 产生的处理结果。这个处理结果应该是 String 或者 Buffer（被转换为一个 string）。具体的用法，可以看 [Loader](https://webpack.js.org/api/loaders#thisadddependency) 官网的描述。接下来我们从源码的角度去分析，为什么可以这样做，为什么可以实现同异步钩子，内部到底是怎么实现的。那么这就是 [loader-runner](https://github.com/webpack/loader-runner) 的作用所在。

## 入口

loader-runner 是一个独立出去的 npm 包，它的入口在 `lib/LoaderRunner.js`。

```js
exports.runLoaders = function runLoaders(options, callback) {
	// read options
	var resource = options.resource || ""; // loaders 处理的资源
	var loaders = options.loaders || []; // loaders 配置
	var loaderContext = options.context || {}; // 所有 loaders 共享的数据
	var readResource = options.readResource || readFile; // 文件输入系统

	//
	var splittedResource = resource && splitQuery(resource);
	var resourcePath = splittedResource ? splittedResource[0] : undefined; // 资源路径
	var resourceQuery = splittedResource ? splittedResource[1] : undefined; // 资源的 query
	var contextDirectory = resourcePath ? dirname(resourcePath) : null; // 资源的目录

	// execution state
	var requestCacheable = true; // 缓存的标识位
	var fileDependencies = []; // 文件依赖的缓存
	var contextDependencies = []; // 目录依赖的缓存

	// prepare loader objects
	loaders = loaders.map(createLoaderObject); // 处理 loaders 的若干属性
  // loaderContext 是在所有 loaders 处理资源时候共享的一份数据
  // loaderIndex 是一个指针，它控制了所有 loaders 的 pitch 与 normal 函数的执行
	loaderContext.context = contextDirectory;
	loaderContext.loaderIndex = 0;
	loaderContext.loaders = loaders;
	loaderContext.resourcePath = resourcePath;
	loaderContext.resourceQuery = resourceQuery;
	loaderContext.async = null; // 为了实现异步 loader 的闭包函数
	loaderContext.callback = null; // 为了实现同步或者异步 loader 的闭包函数
	loaderContext.cacheable = function cacheable(flag) {
		if(flag === false) {
			requestCacheable = false;
		}
	};
	loaderContext.dependency = loaderContext.addDependency = function addDependency(file) {
		fileDependencies.push(file);
	};
	loaderContext.addContextDependency = function addContextDependency(context) {
		contextDependencies.push(context);
	};
	loaderContext.getDependencies = function getDependencies() {
		return fileDependencies.slice();
  };
	loaderContext.getContextDependencies = function getContextDependencies() {
		return contextDependencies.slice();
  };
  // 清除所有缓存
	loaderContext.clearDependencies = function clearDependencies() {
		fileDependencies.length = 0;
		contextDependencies.length = 0;
		requestCacheable = true;
  };

  // 这些 getter/setter 都是为了在 loader 函数里面通过 this 求值能动态得到对应的值
	Object.defineProperty(loaderContext, "resource", {
		enumerable: true,
		get: function() {
			if(loaderContext.resourcePath === undefined)
				return undefined;
			return loaderContext.resourcePath + loaderContext.resourceQuery;
		},
		set: function(value) {
			var splittedResource = value && splitQuery(value);
			loaderContext.resourcePath = splittedResource ? splittedResource[0] : undefined;
			loaderContext.resourceQuery = splittedResource ? splittedResource[1] : undefined;
		}
	});
	Object.defineProperty(loaderContext, "request", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	Object.defineProperty(loaderContext, "remainingRequest", {
		enumerable: true,
		get: function() {
			if(loaderContext.loaderIndex >= loaderContext.loaders.length - 1 && !loaderContext.resource)
				return "";
			return loaderContext.loaders.slice(loaderContext.loaderIndex + 1).map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	Object.defineProperty(loaderContext, "currentRequest", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.slice(loaderContext.loaderIndex).map(function(o) {
				return o.request;
			}).concat(loaderContext.resource || "").join("!");
		}
	});
	Object.defineProperty(loaderContext, "previousRequest", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders.slice(0, loaderContext.loaderIndex).map(function(o) {
				return o.request;
			}).join("!");
		}
	});
	Object.defineProperty(loaderContext, "query", {
		enumerable: true,
		get: function() {
			var entry = loaderContext.loaders[loaderContext.loaderIndex];
			return entry.options && typeof entry.options === "object" ? entry.options : entry.query;
		}
	});
	Object.defineProperty(loaderContext, "data", {
		enumerable: true,
		get: function() {
			return loaderContext.loaders[loaderContext.loaderIndex].data;
		}
	});

	// 防止 loaderContext 加入新属性
	if(Object.preventExtensions) {
		Object.preventExtensions(loaderContext);
	}

  // resourceBuffer 属性是资源文件对应的 Buffer
	var processOptions = {
		resourceBuffer: null,
		readResource: readResource
  };
  // 整个 loaders 的 pitch 与 normal 函数的执行全流程
	iteratePitchingLoaders(processOptions, loaderContext, function(err, result) {
		// ...... 先省略
	});
};
```

`runLoaders` 函数接收 options 与 callback 作为入参。

1. **options的处理**

  这一步是处理传入的 options，并且生成一些属性。其中 `createLoaderObject` 里面是对每个 `loaders` 的处理。

  ```js
  function createLoaderObject(loader) {
    var obj = {
      path: null,
      query: null,
      options: null,
      ident: null,
      normal: null,
      pitch: null,
      raw: null,
      data: null,
      pitchExecuted: false,
      normalExecuted: false
    };
    Object.defineProperty(obj, "request", {
      enumerable: true,
      get: function() {
        return obj.path + obj.query;
      },
      set: function(value) {
        if(typeof value === "string") {
          var splittedRequest = splitQuery(value);
          obj.path = splittedRequest[0];
          obj.query = splittedRequest[1];
          obj.options = undefined;
          obj.ident = undefined;
        } else {
          if(!value.loader)
            throw new Error("request should be a string or object with loader and object (" + JSON.stringify(value) + ")");
          obj.path = value.loader;
          obj.options = value.options;
          obj.ident = value.ident;
          if(obj.options === null)
            obj.query = "";
          else if(obj.options === undefined)
            obj.query = "";
          else if(typeof obj.options === "string")
            obj.query = "?" + obj.options;
          else if(obj.ident)
            obj.query = "??" + obj.ident;
          else if(typeof obj.options === "object" && obj.options.ident)
            obj.query = "??" + obj.options.ident;
          else
            obj.query = "?" + JSON.stringify(obj.options);
        }
      }
    });
    obj.request = loader;
    if(Object.preventExtensions) {
      Object.preventExtensions(obj);
    }
    return obj;
  }
  ```

  loader 对象上会新增很多属性，比如 `normal` 是对应 loader 暴露出来的函数，`pitch` 是 loader 的 pitch 函数。`raw` 也是 loader 暴露出来的属性用来控制是否将资源转成 Buffer 格式。`pitchExecuted` 与 `normalExecuted` 用来标记当前 loader 的 pitch 与 normal 是否执行，在 `iteratePitchingLoaders` 内部是用来控制 `loaderContext.loaderindex` 的变化。

2. **loaderContext的处理**

  这个变量非常重要，它的 `loaderIndex` 属性控制了所有 loaders 的 pitch 与 normal 执行的所有流程， `async` 是一个闭包函数，它是实现异步 loader 的关键，这是后话。 `callback` 也是一个闭包函数，不过它不仅能实现同步 loader，还能实行异步 loader，同时还支持在 loader 的函数里面返回一个 promise。

3. **getter/setter**

  接下来就是通过 Object.defineProperty 将 loaderContext 的一些属性变成 `getter/setter`，这样做的目的是为了形成闭包的环境，每个 loader 函数里面通过 `this` 语法，能够动态的取到一些属性，比如 `data`、`query`、`remainingRequest` 等等。因为它们都会随着 `loaderIndex` 变化而改变。

4. **iteratePitchingLoaders**

  最后调用 iteratePitchingLoaders 开始所有 loaders 的 pitch 与 normal 函数的执行。

## iteratePitchingLoaders

```js
function iteratePitchingLoaders(options, loaderContext, callback) {
	// 如果所有 loaders 的 pitch 都执行完成
	if(loaderContext.loaderIndex >= loaderContext.loaders.length)
		return processResource(options, loaderContext, callback);

	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];

	// 进入下一个 loader 的 pitch 函数
	if(currentLoaderObject.pitchExecuted) {
		loaderContext.loaderIndex++;
		return iteratePitchingLoaders(options, loaderContext, callback);
	}

	// 加载当前 loader 的 pitch 与 normal 函数
	loadLoader(currentLoaderObject, function(err) {
		if(err) {
			loaderContext.cacheable(false);
			return callback(err);
		}
    var fn = currentLoaderObject.pitch;
    // 标识当前 loader 的 pitch 执行完成，就会走到上面的 loaderContext.loaderIndex++ 逻辑。
		currentLoaderObject.pitchExecuted = true;
		if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);

    // 调用当前 loader 的 pitch，决定是否进入下个 pitch，
    // 还是跳过后面所有的 loader 的 pitch 与 normal(包括当前 normal)。
		runSyncOrAsync(
			fn,
			loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
			function(err) {
				if(err) return callback(err);
				var args = Array.prototype.slice.call(arguments, 1);

        // 如果当前 pitch 返回了一个不含有 `undefined` 的值
        // 那么就放弃之后的 loader 的 pitch 与 normal 的执行。
				var hasArg = args.some(function(value) {
					return value !== undefined;
				});
				if(hasArg) {
					loaderContext.loaderIndex--;
					iterateNormalLoaders(options, loaderContext, args, callback);
				} else {
					iteratePitchingLoaders(options, loaderContext, callback);
				}
			}
		);
	});
}
```

可以看出函数的内部，有多处调用 iteratePitchingLoaders 自身的逻辑，`loaderContext.loaderIndex` 与 `currentLoaderObject.pitchExecuted` 在递归的逻辑当中发挥了决定性的作用。

在 webpack 官网当中，作者对 [pitch loader](https://webpack.js.org/api/loaders#thisadddependency) 做了一些描述。

> loaders 是从右向左开始执行，但是有些时候 loaders 只关心 `metadata`，也就是 `loaderContext` 上面的一些共享的属性，这也就是 `pitch` 方法存在的意义。它的调用是从左向右执行，并且还能跳过某些 loader 的执行。

举个例子：

```js
module.exports = {
  //...
  module: {
    rules: [
      {
        //...
        use: [
          'a-loader',
          'b-loader',
          'c-loader'
        ]
      }
    ]
  }
};
```

那么 `iteratePitchingLoaders` 方法的执行流程如下：

```markup
|- a-loader `pitch`
  |- b-loader `pitch`
    |- c-loader `pitch`
      |- requested module is picked up as a dependency
    |- c-loader normal execution
  |- b-loader normal execution
|- a-loader normal execution
```

就具体的原来分析来看。首先，`loaderContext.loaderIndex` 是从 0 开始递增，说明 pitch 的执行就是从左向右的顺序。接着就会走到 `loadLoader` 的逻辑，这个逻辑就是加载 loader 函数。

```js
module.exports = function loadLoader(loader, callback) {
	try {
		var module = require(loader.path);
	} catch(e) {
		// it is possible for node to choke on a require if the FD descriptor
		// limit has been reached. give it a chance to recover.
		if(e instanceof Error && e.code === "EMFILE") {
			var retry = loadLoader.bind(null, loader, callback);
			if(typeof setImmediate === "function") {
				// node >= 0.9.0
				return setImmediate(retry);
			} else {
				// node < 0.9.0
				return process.nextTick(retry);
			}
		}
		return callback(e);
	}
	if(typeof module !== "function" && typeof module !== "object") {
		return callback(new LoaderLoadingError(
			"Module '" + loader.path + "' is not a loader (export function or es6 module)"
		));
	}
	loader.normal = typeof module === "function" ? module : module.default;
	loader.pitch = module.pitch;
	loader.raw = module.raw;
	if(typeof loader.normal !== "function" && typeof loader.pitch !== "function") {
		return callback(new LoaderLoadingError(
			"Module '" + loader.path + "' is not a loader (must have normal or pitch function)"
		));
	}
	callback();
};
```

loadLoader 的执行很简单，就是加载 loader 这个 module，并且挂载 `normal`、`pitch`、`raw` 三个属性，`raw` 属性用来控制读入资源文件的是不是 Buffer 类型。

既然 loadLoader 加载完 loader 之后，那么就走到它的回调函数里面。

```js
loadLoader(currentLoaderObject, function(err) {
  if(err) {
    loaderContext.cacheable(false);
    return callback(err);
  }
  var fn = currentLoaderObject.pitch;
  // 标识当前 loader 的 pitch 执行完成，就会走到上面的 loaderContext.loaderIndex++ 逻辑。
  currentLoaderObject.pitchExecuted = true;
  if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);

  // 调用当前 loader 的 pitch，决定是否进入下个 pitch，
  // 还是跳过后面所有的 loader，执行当前 loader 的 normal 函数。
  runSyncOrAsync(
    fn,
    loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
    function(err) {
      if(err) return callback(err);
      var args = Array.prototype.slice.call(arguments, 1);

      // 如果当前 pitch 返回了一个不含有 `undefined` 的值
      // 那么就放弃之后的 loader 的 pitch 与 normal 的执行。
      var hasArg = args.some(function(value) {
        return value !== undefined;
      });
      if(hasArg) {
        loaderContext.loaderIndex--;
        iterateNormalLoaders(options, loaderContext, args, callback);
      } else {
        iteratePitchingLoaders(options, loaderContext, callback);
      }
    }
  );
});
```

如果当前 fn（即 loader.pitch） 不存在，那么就走进下一个 loader 的 pitch 的逻辑。这样一步一步的递归 `iteratePitchingLoaders`，直到所有 loader 的 pitch 函数都执行完，那么就走到下面处理资源文件的逻辑。

```js
if(loaderContext.loaderIndex >= loaderContext.loaders.length)
    return processResource(options, loaderContext, callback);

function processResource(options, loaderContext, callback) {
	// set loader index to last loader
	loaderContext.loaderIndex = loaderContext.loaders.length - 1;

	var resourcePath = loaderContext.resourcePath;
	if(resourcePath) {
		loaderContext.addDependency(resourcePath);
		options.readResource(resourcePath, function(err, buffer) {
			if(err) return callback(err);
			options.resourceBuffer = buffer;
			iterateNormalLoaders(options, loaderContext, [buffer], callback);
		});
	} else {
		iterateNormalLoaders(options, loaderContext, [null], callback);
	}
}
```

processResource 的执行，代表着 pitch 的执行都完成了，开始读入资源文件的 buffer。

```js
loaderContext.loaderIndex = loaderContext.loaders.length - 1;
```
先将 loaderIndex 设置为最后一个，即 normal 的执行是逆向的。接着调用 `addDependency` 将当前资源文件的路径推入 `fileDependencies` 数组，也就是在整个资源文件被 loaders 处理的过程当中，都能拿到这个 `fileDependencies` 数组的数据，进而开始调用 `iterateNormalLoaders` 来执行 loaders 的 normal 函数。我们来看下 `iterateNormalLoaders` 函数的执行。

## iterateNormalLoaders

```js
function iterateNormalLoaders(options, loaderContext, args, callback) {
	if(loaderContext.loaderIndex < 0)
		return callback(null, args);

	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];

	// iterate
	if(currentLoaderObject.normalExecuted) {
		loaderContext.loaderIndex--;
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}

	var fn = currentLoaderObject.normal;
	currentLoaderObject.normalExecuted = true;
	if(!fn) {
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}

	convertArgs(args, currentLoaderObject.raw);

	runSyncOrAsync(fn, loaderContext, args, function(err) {
		if(err) return callback(err);

		var args = Array.prototype.slice.call(arguments, 1);
		iterateNormalLoaders(options, loaderContext, args, callback);
	});
}
```

如果 `loaderContext.loaderIndex` 已经小于 0，那么就执行 `iteratePitchingLoaders` 的回调函数，进而退出递归。如果当前 loader 的 normal 函数执行完了，也就是当前 `loader.normalExecuted` 的值为 true，就开始递减 `loaderContext.loaderIndex`，接着执行下一个 loader，否则就执行当前 loader 的 normal 函数，并且将 normalExecuted 属性设置为 true，这样下次递归 iterateNormalLoaders 的时候，就能进入下一个 loader 的执行了。执行当前 loader 会先调用 `convertArgs` 来决定是否将上一个 loader 传入的 result 转化为 buffer。看下 `convertArgs`的逻辑：

```js
function convertArgs(args, raw) {
	if(!raw && Buffer.isBuffer(args[0]))
		args[0] = utf8BufferToString(args[0]);
	else if(raw && typeof args[0] === "string")
		args[0] = Buffer.from(args[0], "utf-8");
}
```

逻辑很简单，就是判断如果当前 loader 导出的函数的 raw 属性为 true，那么就转化上一个 loader 传入的 result 为 buffer。

最后走入 `runSyncOrAsync` 函数，这个函数是 loader-runner 最核心的一步，它决定了当前 loader 的走向，支持异步 loader，同步 loader，也支持返回 promise 的 loader。

```js
function runSyncOrAsync(fn, context, args, callback) {
	var isSync = true;
	var isDone = false;
	var isError = false; // internal error
	var reportedError = false;
	context.async = function async() {
		if(isDone) {
			if(reportedError) return; // ignore
			throw new Error("async(): The callback was already called.");
		}
		isSync = false;
		return innerCallback;
	};
	var innerCallback = context.callback = function() {
		if(isDone) {
			if(reportedError) return; // ignore
			throw new Error("callback(): The callback was already called.");
		}
		isDone = true;
		isSync = false;
		try {
			callback.apply(null, arguments);
		} catch(e) {
			isError = true;
			throw e;
		}
	};
	try {
		var result = (function LOADER_EXECUTION() {
			return fn.apply(context, args);
		}());
		if(isSync) {
			isDone = true;
			if(result === undefined)
				return callback();
			if(result && typeof result === "object" && typeof result.then === "function") {
				return result.then(function(r) {
					callback(null, r);
				}, callback);
			}
			return callback(null, result);
		}
	} catch(e) {
		if(isError) throw e;
		if(isDone) {
			// loader is already "done", so we cannot use the callback function
			// for better debugging we print the error on the console
			if(typeof e === "object" && e.stack) console.error(e.stack);
			else console.error(e);
			return;
		}
		isDone = true;
		reportedError = true;
		callback(e);
	}
}
```

函数内部的 `isSync` 和 `isDone` 很重要，`isSync` 是来控制同步还是异步 loader的，`isDone` 是防止 `callback` 被触发多次。`context.async` 是一个闭包函数，它返回的是 innerCallback，而 innerCallback 内部才是真正执行 `runSyncOrAsync` 的 callback 函数，这个 callback 会进入下一次的 iterateNormalLoaders 逻辑。同时 innerCallback 也是 `context.callback` 的一个引用。真正执行 loader 的 normal 的函数语句在下面的这个立即执行函数里面。

```js
try {
  var result = (function LOADER_EXECUTION() {
    return fn.apply(context, args);
  }());
  if(isSync) {
    isDone = true;
    if(result === undefined)
      return callback();
    if(result && typeof result === "object" && typeof result.then === "function") {
      return result.then(function(r) {
        callback(null, r);
      }, callback);
    }
    return callback(null, result);
  }
}
```

LOADER_EXECUTION 函数内部调用了 fn，即 loader 的 normal 函数，并且绑定了上下文 context，context 就是 runLoaders 内部声明的 `loaderContext`。它拥有很多属性和方法，这也就是为啥我们在 loader 里面能够通过 this 获取到它的属性和方法。

```js
module.exports = function(content, map, meta) {
  // 获取到 loaderContext.async
  var callback = this.async();
  // 获取 loaderContext.callback
  var callback = this.callback;
  // 获取当前 loader 索引
  var index = this.loaderIndex
  // 等等。
};
```

然后根据 `result` 与 `isDone` 来决定如何调用 `callback` 来进入下一个 loader 到 normal 函数。这里有三种情况：

1. 同步 loader

```js
module.exports = function(content, map, meta) {
  return someSyncOperation(content);
};
```

如果是同步 loader，那么 `isSync` 为 true，这里判断如果 `result` 是一个 promise，那么等这个 promise 完成之后，调用 callback，否则就调用 callback。

2. 异步 loader

```js
module.exports = function(content, map, meta) {
  this.callback(null, someSyncOperation(content), map, meta);
  return; // always return undefined when calling callback()
};

module.exports = function(content, map, meta) {
  var callback = this.async();
  someAsyncOperation(content, function(err, result) {
    if (err) return callback(err);
    callback(null, result, map, meta);
  });
};
```

如果是异步 loader，那么必须调用 `this.callback(/** arguments */)` 或者 `this.aysnc()`，因为这两个语法其实就是执行 `innerCallback` 函数，内部会将 `isSync` 设置为 false，这样就不会走到同步 loader 的 `if(isSync)` 逻辑。而且 `innerCallback` 函数的内部会调用 callback，进而走到 iterateNormalLoaders 的执行，这样又进入了下一个 loader 的 normal 函数了。

那么 runLoaders 的整体执行流程如下图：

![loader 执行图](https://raw.githubusercontent.com/theniceangel/webpack-learning/master/images/loader.png)

## iteratePitchingLoaders 内的 runSyncOrAsync

刚才讲到的是 loader 的 normal 函数的执行都是在 runSyncOrAsync 内部。其实在我们将 pitch 的时候，也是会执行 runSyncOrAsync，而 pitch 函数的返回，会影响之后所有 loaders 的 pitch 和 normal 阶段。它的逻辑在 loadLoader 的回调里面，代码如下：

```js
loadLoader(currentLoaderObject, function(err) {
  if(err) {
    loaderContext.cacheable(false);
    return callback(err);
  }
  var fn = currentLoaderObject.pitch;
  // 标识当前 loader 的 pitch 执行完成，就会走到上面的 loaderContext.loaderIndex++ 逻辑。
  currentLoaderObject.pitchExecuted = true;
  if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);

  // 调用当前 loader 的 pitch，决定是否进入下个 pitch，
  // 还是跳过后面所有的 loader 的 pitch 与 normal(包括当前 normal)函数。
  runSyncOrAsync(
    fn,
    loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
    function(err) {
      if(err) return callback(err);
      var args = Array.prototype.slice.call(arguments, 1);

      // 如果当前 pitch 返回了一个不含有 `undefined` 的值
      // 那么就放弃之后的 loader 的 pitch 与 normal 的执行。
      var hasArg = args.some(function(value) {
        return value !== undefined;
      });
      if(hasArg) {
        loaderContext.loaderIndex--;
        iterateNormalLoaders(options, loaderContext, args, callback);
      } else {
        iteratePitchingLoaders(options, loaderContext, callback);
      }
    }
  );
});
```

根据上述我们对 runSyncOrAsync 的分析，args 是取决于 `pitch` 函数的返回。如果 `pitch` 只要返回的值都是 undefined，那么直接走到 `iterateNormalLoaders` 的逻辑，也就是跳过 `processResource` 与后面所有 loaders 的 pitch 与 normal 的执行（包括当前 loader 的 normal）。举个列子：

```js
// webpack rules 配置
rules: [
    {
      //...
      use: [
        'a-loader',
        'b-loader',
        'c-loader'
      ]
    }
  ]
}

// a-loader.js
module.exports = function(source) {
	return source + "-simple";
};

// b-loader.js(pitch)
module.exports = function(source) {
	return "I am b-loader.js";
};
module.exports.pitch = function(remainingRequest, previousRequest, data) {
	return 'pitching B'
};

// c-loader.js
module.exports = function(source) {
	return "this loader is ignored?";
};
module.exports.pitch = function(source) {
	return "pitching C won't be excuted";
};

```

由于第二个 `b-loader.js` 含有 `pitch`，并且返回了不为 `undefined` 的值，所以 `b-loader.js` 的 normal 函数不会执行。`c-loader.js` 的 pitch 与 normal 函数也不会被执行。

## 总结

从 LoaderRunner 的源码来看，源码的设计非常灵活，引入了 `pitch` 的概念，并且支持同异步的 loader 和返回 promise 的同步 loader。相信，经历了这篇文章，你对 loader 的概念和执行已经很清晰了，接下来就是看看一些比较有名的 loader，巩固如何去写 loader 了。