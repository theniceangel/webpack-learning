## watchpack

watchpack 是 webpack 用来 watch 文件或者目录变化的一个类库，与 node 提供的 `fs.watch` 相比较，它有着更高的抽象。

1. 对于每一个目录，都只有一个 `DirectoryWatcher` 实例，并且同一个目录下的不同文件是享用同一个 `DirectoryWatcher` 实例。
2. 对于同一个应用程序来说，是共享一个 `WatcherManager` 实例，所有的 `DirectoryWatcher` 实例都是通过 Map 存储在 `WatcherManager` 实例上。

> 当前版本是 2.0.0-beta.4

## 使用

```js
var Watchpack = require("watchpack");

var wp = new Watchpack({
  // 派发 aggregated 事件的间隔
	aggregateTimeout: 1000

  // 是否开启轮询
	poll: true

  // 忽略的目录或者文件
	ignored: "**/.git",
});

// Watchpack.prototype.watch(files: string[], directories: string[], startTime?: number)
wp.watch(listOfFiles, listOfDirectories, Date.now() - 10000);
// starts watching these files and directories
// calling this again will override the files and directories

wp.on("change", function(filePath, mtime) {
	// filePath: the changed file
	// mtime: last modified time for the changed file (null if file was removed)
	// for folders it's a time before that all changes in the directory happened
});

wp.on("aggregated", function(changes, removals) {
	// changes: a Set of all changed files
	// removals: a Set of all removed files
});

// Watchpack.prototype.pause()
wp.pause();
// stops emitting events, but keeps watchers open
// next "watch" call can reuse the watchers

// Watchpack.prototype.close()
wp.close();
// stops emitting events and closes all watchers

// Watchpack.prototype.getTimeInfoEntries()
var fileTimes = wp.getTimeInfoEntries();
```

## 源码

源码由三个文件组成，`watchpack`、`watcherManager`、`DirectoryWatcher`。

### watchpack

```js
class Watchpack extends EventEmitter {
	constructor(options) {
		super();
		if (!options) options = {};
		if (!options.aggregateTimeout) options.aggregateTimeout = 200;
		this.options = options;
		this.watcherOptions = {
			ignored: ignoredToRegexp(options.ignored),
			poll: options.poll
		};
		this.fileWatchers = [];
		this.dirWatchers = [];
		this.paused = false;
		this.aggregatedChanges = new Set();
    this.aggregatedRemovals = new Set();
		this.aggregateTimeout = 0;
		this._onTimeout = this._onTimeout.bind(this);
    }
}
```

`Watchpack` 继承了 `EventEmitter` 类，所以他的原型方法上就有 `on` 与 `emit` 方法，来注册和派发事件。先看下它的实例属性：

1. **options**

    传入的配置项

2. **watcherOptions**

    关于 watch 功能的配置项。`ignored` 表示忽略某些文件或者目录。`poll` 是否开启轮询来 watch 文件的变化。

3. **fileWatchers**

    存放监听文件的 watcher。

4. **dirWatchers**

    存放监听目录的 watcher。

5. **paused**

    如果为 `true`，就不派发事件

6. **aggregatedChanges、aggregatedRemovals、aggregateTimeout、_onTimeout**

    这四个都是跟 `aggregated` 事件有关，对于 watchpack 来说，只要监听到文件的变化或者目录的变化就会执行 `change` 事件。而 `aggregated` 则是更低频率的 `change` 事件，只要 `change` 事件派发了，就会派发 `aggregated`。只不过是在 `aggregateTimeout` 事件之后执行的，并且如果上一次 `aggregated` 没执行，则会被清除。代码如下：

    ```js
    // 用来派发 aggregated 事件，aggregatedChanges 与 aggregatedRemovals 就是监听器的入餐。
    _onTimeout() {
      this.aggregateTimeout = 0;
      const changes = this.aggregatedChanges;
      const removals = this.aggregatedRemovals;
      this.aggregatedChanges = new Set();
      this.aggregatedRemovals = new Set();
      this.emit("aggregated", changes, removals);
    }

    _onChange(item, mtime, file, type) {
      file = file || item;
      if (this.paused) return;
      this.emit("change", file, mtime, type);
      // 如果上一次的 aggregateTimeout 还没执行，就会被清除。
      if (this.aggregateTimeout) clearTimeout(this.aggregateTimeout);
      this.aggregatedChanges.add(item);
      this.aggregateTimeout = setTimeout(
        this._onTimeout,
        this.options.aggregateTimeout
      );
    }
    ```

- **getTimeInfoEntries**

  ```js
  getTimeInfoEntries() {
    if (EXISTANCE_ONLY_TIME_ENTRY === undefined) {
      EXISTANCE_ONLY_TIME_ENTRY = require("./DirectoryWatcher")
        .EXISTANCE_ONLY_TIME_ENTRY;
    }
    const directoryWatchers = new Set();
    addWatchersToSet(this.fileWatchers, directoryWatchers);
    addWatchersToSet(this.dirWatchers, directoryWatchers);
    const map = new Map();
    for (const w of directoryWatchers) {
      const times = w.getTimeInfoEntries();
      for (const [path, entry] of times) {
        if (map.has(path)) {
          if (entry === EXISTANCE_ONLY_TIME_ENTRY) continue;
          const value = map.get(path);
          if (value === entry) continue;
          if (value !== EXISTANCE_ONLY_TIME_ENTRY) {
            map.set(path, Object.assign({}, value, entry));
            continue;
          }
        }
        map.set(path, entry);
      }
    }
    return map;
  }
  ```

  返回一个带有所有文件与目录的事件信息的 map。它的数据结构如下：

  ```markup
  Map<string, { safeTime: number, timestamp: number }>
  ```

- **watch**

  `watch` 是对外暴露的 API，让调用方能够 watch 自己想要的文件或者目录。

  ```js
  watch(files, directories, startTime) {
    this.paused = false;
    const oldFileWatchers = this.fileWatchers;
    const oldDirWatchers = this.dirWatchers;
    const filter = this.watcherOptions.ignored
      ? path => !this.watcherOptions.ignored.test(path.replace(/\\/g, "/"))
      : () => true;
    this.fileWatchers = files
      .filter(filter)
      .map(file =>
        this._fileWatcher(
          file,
          watcherManager.watchFile(file, this.watcherOptions, startTime)
        )
      );
    this.dirWatchers = directories
      .filter(filter)
      .map(dir =>
        this._dirWatcher(
          dir,
          watcherManager.watchDirectory(dir, this.watcherOptions, startTime)
        )
      );
    oldFileWatchers.forEach(w => w.close());
    oldDirWatchers.forEach(w => w.close());
  }
  ```

  函数内部先保存 `oldFileWatchers` 与 `oldDirWatchers`，等待后面的销毁。然后过滤出匹配的 fileWatcher 与 dirWatcher。它们是由 `_fileWatcher` 与 `_dirWatcher` 方法生成的。

  ```js
  _fileWatcher(file, watcher) {
    watcher.on("change", (mtime, type) => {
      this._onChange(file, mtime, file, type);
    });
    watcher.on("remove", type => {
      this._onRemove(file, file, type);
    });
    return watcher;
  }

  _dirWatcher(item, watcher) {
    watcher.on("change", (file, mtime, type) => {
      this._onChange(item, mtime, file, type);
    });
    return watcher;
  }
  ```

  它们都接收绝对路径作为第一个参数，watcher 作为第二个参数。并且在监听了 `watcher` 的 `change` 事件，进而执行 watchpack 的 `_onChange` 方法，在 `_onChange` 的逻辑里面，就 `emit` 了 `change` 与 `aggregated` 事件，如果用户监听了这两个事件，那么相对应的回调函数就会执行。

  ```js
  _onChange(item, mtime, file, type) {
    file = file || item;
    if (this.paused) return;
    this.emit("change", file, mtime, type);
    if (this.aggregateTimeout) clearTimeout(this.aggregateTimeout);
    this.aggregatedChanges.add(item);
    this.aggregateTimeout = setTimeout(
      this._onTimeout,
      this.options.aggregateTimeout
    );
  }
  ```

  刚才说到 `_fileWatcher` 与 `_dirWatcher` 的第二个 `watcher` 参数。这个是调用 `watcherManager.watchFile` 与 `watcherManager.watchDirectory` 返回的。那么目光转移到 `watcherManager` 看看。

### watcherManager

`watcherManager.js` 文件默认导出了 `WatcherManager` 实例，意味着所有引入这个模块的，都共享了相同的一份实例。

```js
class WatcherManager {
	constructor() {
		this.directoryWatchers = new Map();
	}

	getDirectoryWatcher(directory, options) {
		if (DirectoryWatcher === undefined) {
			DirectoryWatcher = require("./DirectoryWatcher");
		}
		options = options || {};
		const key = directory + " " + JSON.stringify(options);
		const watcher = this.directoryWatchers.get(key);
		if (watcher === undefined) {
			const newWatcher = new DirectoryWatcher(directory, options);
			this.directoryWatchers.set(key, newWatcher);
			newWatcher.on("closed", () => {
				this.directoryWatchers.delete(key);
			});
			return newWatcher;
		}
		return watcher;
	}

	watchFile(p, options, startTime) {
		const directory = path.dirname(p);
		return this.getDirectoryWatcher(directory, options).watch(p, startTime);
	}

	watchDirectory(directory, options, startTime) {
		return this.getDirectoryWatcher(directory, options).watch(
			directory,
			startTime
		);
	}
}

module.exports = new WatcherManager();
```

`directoryWatchers` 是一个 Map。它的数据结构如下：

```js
// key 是绝对路径
// value 是 DirectoryWatcher 实例
Map<string, directoryWatcher>
```

`watchFile` 与 `watchDirectory` 都调用了 `getDirectoryWatcher` 返回一个 directoryWatcher，并且调用了 `directoryWatcher.watch` 方法来返回一个 watcher。

`getDirectoryWatcher` 方法很简单，就是获取或者生成一个 directoryWatcher。并且将它存放在 `directoryWatchers` 属性上，它相对应的键名是绝对路径拼接序列化后的 options 而生成的字符串。

从上面来看，`watcherManager` 就类似于一个管理器，它存放了所有的 directoryWatcher。而 directoryWatcher 才是 watcherpack 的核心逻辑所在。

### DirectoryWatcher

```js
class DirectoryWatcher extends EventEmitter {
	constructor(directoryPath, options) {
		super();
		if (FORCE_POLLING) {
			options.poll = FORCE_POLLING;
		}
		this.options = options;
		this.path = directoryPath;
		// safeTime is the point in time after which reading is safe to be unchanged
		// timestamp is a value that should be compared with another timestamp (mtime)
		/** @type {Map<string, { safeTime: number, timestamp: number }} */
		this.files = new Map();
		this.directories = new Map();
		this.lastWatchEvent = 0;
		this.initialScan = true;
		this.ignored = options.ignored;
		this.nestedWatching = false;
		this.polledWatching =
			typeof options.poll === "number"
				? options.poll
				: options.poll
				? 5007
				: false;
		this.initialScanRemoved = new Set();
		this.watchers = new Map();
		this.parentWatcher = null;
		this.refs = 0;
		this._activeEvents = new Map();
		this.closed = false;
		this.scanning = false;
		this.scanAgain = false;
		this.scanAgainInitial = false;

		this.createWatcher();
		this.doScan(true);
  }
}
```

`DirectoryWatcher` 也是继承了 `EventEmitter`。实例上有几个比较重要的属性。

1. **path**

  当前 directoryWatcher 的绝对路径。

2. **files**

  保存当前目录下的所有文件的 timeInfo(`{ safeTime: number, timestamp: number }`) 信息。

3. **watchers**

  文件或者目录与 `watcher` 构成的 Map

4. **directories**

  子目录绝对路径与 `watcher` 构成的 Map

再来看下 `createWatcher`。

```js
createWatcher() {
  try {
    if (this.polledWatching) {
      // Poll for changes
      const interval = setInterval(() => {
        this.doScan(false);
      }, this.polledWatching);
      this.watcher = {
        close: () => {
          clearInterval(interval);
        }
      };
    } else {
      if (IS_OSX) {
        this.watchInParentDirectory();
      }
      this.watcher = fs.watch(this.path);
      this.watcher.on("change", this.onWatchEvent.bind(this));
      this.watcher.on("error", this.onWatcherError.bind(this));
    }
  } catch (err) {
    this.onWatcherError(err);
  }
}
```

一般走到 `else` 逻辑，也就是 `this.watcher = fs.watch(this.path)`。可以看到 `watcher` 属性就是 node 的 `fs.watch` 返回的值。并且还注册了 `change` 和 `error` 监听器。

之后走到了 `this.doScan(true)` 逻辑。`doScan` 的作用就是扫描某一目录下面的所有文件或者子目录，填充 `this.watchers`、`this.directories` 等信息。

主要还是看下 `DirectoryWatcher.watch` 方法。

```js
watch(filePath, startTime) {
  const key = withoutCase(filePath);
  let watchers = this.watchers.get(key);
  if (watchers === undefined) {
    watchers = new Set();
    this.watchers.set(key, watchers);
  }
  this.refs++;
  const watcher = new Watcher(this, filePath, startTime);
  watcher.on("closed", () => {
    watchers.delete(watcher);
    if (watchers.size === 0) {
      this.watchers.delete(key);
      if (this.path === filePath) this.setNestedWatching(false);
    }
    if (--this.refs <= 0) this.close();
  });
  watchers.add(watcher);
  let safeTime;
  if (filePath === this.path) {
    this.setNestedWatching(true);
    safeTime = this.lastWatchEvent;
    for (const entry of this.files.values()) {
      fixupEntryAccuracy(entry);
      safeTime = Math.max(safeTime, entry.safeTime);
    }
  } else {
    const entry = this.files.get(filePath);
    if (entry) {
      fixupEntryAccuracy(entry);
      safeTime = entry.safeTime;
    } else {
      safeTime = 0;
    }
  }
  process.nextTick(() => {
    if (this.closed) return;
    if (safeTime) {
      if (safeTime >= startTime) watcher.emit("change", safeTime);
    } else if (this.initialScan && this.initialScanRemoved.has(filePath)) {
      watcher.emit("remove");
    }
  });
  return watcher;
}
```

该方法很简单，就是生成 `Watcher` 实例，并且填充 `files` 这个属性所需要的字段值。而 `Watcher` 的定义如下：

```js
class Watcher extends EventEmitter {
	constructor(directoryWatcher, filePath, startTime) {
		super();
		this.directoryWatcher = directoryWatcher;
		this.path = filePath;
		this.startTime = startTime && +startTime;
	}

	checkStartTime(mtime, initial) {
		const startTime = this.startTime;
		if (typeof startTime !== "number") return !initial;
		return startTime <= mtime;
	}

	close() {
		this.emit("closed");
	}
}
```

它也继承了 `EventEmitter`。`directoryWatcher` 是保持对当前目录 watcher 的引用。`path` 是当前 watcher 对象的文件或者目录的绝对路径。`startTime` 则是开始事件。

从 `directoryWatcher.watch` 方法来看，`watchpack.fileWatchers` 属性或者 `watchpack.dirWatchers` 保存的就是这些 `watcher` 实例。