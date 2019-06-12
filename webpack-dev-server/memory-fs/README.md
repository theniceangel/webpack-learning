# memory-fs

简单的内存文件系统，在 javascript 对象中保存数据。

## 示例

memory-fs 上手非常简单，示例如下：

```js
var fs = new MemoryFileSystem();

fs.mkdirpSync("/a/test/dir");
fs.writeFileSync("/a/test/dir/file.txt", "Hello World");

const buf = fs.readFileSync("/a/test/dir/file.txt"); // returns Buffer("Hello World")
```

保持着与原生 `fs` 模块类似的 `API`，看了这些语法之后，我的内心十分好奇，内部到底是怎么实现的，所以决定深挖一下内部实现。

## 构造函数

```js
class MemoryFileSystem {
	constructor(data) {
		this.data = data || {};
		this.join = join;
		this.pathToArray = pathToArray;
		this.normalize = normalize;
  }
}
```

- **`data`**

保存着所有目录、文件、内容等信息。

- **`join`**

用来拼接路径。

- **`normalize`**

兼容不同 platform 下的文件路径格式。

- **`pathToArray`**

文件路径处理之后组装成的数组。

先从 `mkdirpSync` 这个函数入手

### mkdirpSync

```js
mkdirpSync(_path) {
  const path = pathToArray(_path);
  if(path.length === 0) return;
  let current = this.data;
  for(let i = 0; i < path.length; i++) {
    if(isFile(current[path[i]]))
      throw new MemoryFileSystemError(errors.code.ENOTDIR, _path, "mkdirp");
    else if(!isDir(current[path[i]]))
      current[path[i]] = {"":true};
    current = current[path[i]];
  }
  return;
}
```

接收文件路径字符串作为入参，先调用 `pathToArray` 函数将 `_path` 处理成数组。`pathToArray` 的代码相对比较复杂，因为它是兼容不同平台的格式。

```js
// patchToArray
function pathToArray(path) {
	path = normalize(path);
	const nix = /^\//.test(path);
	if(!nix) {
		if(!/^[A-Za-z]:/.test(path)) {
			throw new MemoryFileSystemError(errors.code.EINVAL, path);
		}
		path = path.replace(/[\\\/]+/g, "\\"); // multi slashs
		path = path.split(/[\\\/]/);
		path[0] = path[0].toUpperCase();
	} else {
		path = path.replace(/\/+/g, "/"); // multi slashs
		path = path.substr(1).split("/");
	}
	if(!path[path.length-1]) path.pop();
	return path;
}

//  normalize
odule.exports = function normalize(path) {
	var parts = path.split(/(\\+|\/+)/);
	if(parts.length === 1)
		return path;
	var result = [];
	var absolutePathStart = 0;
	for(var i = 0, sep = false; i < parts.length; i += 1, sep = !sep) {
		var part = parts[i];
		if(i === 0 && /^([A-Z]:)?$/i.test(part)) {
			result.push(part);
			absolutePathStart = 2;
		} else if(sep) {
			if (i === 1 && parts[0].length === 0 && part === "\\\\") {
				result.push(part);
			} else {
				result.push(part[0]);
			}
		} else if(part === "..") {
			switch(result.length) {
				case 0:
					result.push(part);
					break;
				case 2:
					i += 1;
					sep = !sep;
					result.length = absolutePathStart;
					break;
				case 4:
					if(absolutePathStart === 0) {
						result.length -= 3;
					} else {
						i += 1;
						sep = !sep;
						result.length = 2;
					}
					break;
				default:
					result.length -= 3;
					break;
			}
		} else if(part === ".") {
			switch(result.length) {
				case 0:
					result.push(part);
					break;
				case 2:
					if(absolutePathStart === 0) {
						result.length -= 1;
					} else {
						i += 1;
						sep = !sep;
					}
					break;
				default:
					result.length -= 1;
					break;
			}
		} else if(part) {
			result.push(part);
		}
	}
	if(result.length === 1 && /^[A-Za-z]:$/.test(result))
		return result[0] + "\\";
	return result.join("");
};
```

`pathToArray` 内部先调用 `normalize` 函数来解决不同系统下文件路径格式不一致的问题。接下来根据文件的格式，输出对应的路径组成的数组。举个例子：

```js
normalize("/a/b/c") => "/a/b/c" => ['a', 'b', 'c']
normalize("C:\\a\\b\\..") => "C:\\a" => ['C:', 'a']
```

拿到这些路径节点的名称之后，用作 `this.data` 的 key 名称，并且构成树形结构。

```js
if(path.length === 0) return;
  let current = this.data;
  for(let i = 0; i < path.length; i++) {
    if(isFile(current[path[i]]))
      throw new MemoryFileSystemError(errors.code.ENOTDIR, _path, "mkdirp");
    else if(!isDir(current[path[i]]))
      current[path[i]] = {"":true};
    current = current[path[i]];
  }
  return;
```

从 `mkdirpSync` 内部接下来的代码来看。`path` 是一个路径节点的数组。每次循环 path，拿到对应的节点名称作为 this.data 的多层级的嵌套属性名称。就拿路径 `/a/b/c` 来说，最后 `this.data` 就变成了如下的树状结构。

```js
{
  a: {
    "": true,
    b: {
      "": true,
      c: {
        "": true
      }
    }
  }
}
```

看了 `mkdirpSync` 之后，那么看下对应的 `readdirSync` 函数

### readdirSync

通过 `mkdirpSync` 创建了目录之后，接下来可以通过 `readdirSync` 这个 API 来获取目录的信息。

```js
readdirSync(_path) {
  if(_path === "/") return Object.keys(this.data).filter(Boolean);
  const path = pathToArray(_path);
  let current = this.data;
  let i = 0;
  for(; i < path.length - 1; i++) {
    if(!isDir(current[path[i]]))
      throw new MemoryFileSystemError(errors.code.ENOENT, _path, "readdir");
    current = current[path[i]];
  }
  if(!isDir(current[path[i]])) {
    if(isFile(current[path[i]]))
      throw new MemoryFileSystemError(errors.code.ENOTDIR, _path, "readdir");
    else
      throw new MemoryFileSystemError(errors.code.ENOENT, _path, "readdir");
  }
  return Object.keys(current[path[i]]).filter(Boolean);
}
```

函数首先判断，如果 `_path = /` 的时候，直接取 `this.data` 的根属性拿到目录信息。接着就是通过路径数组一层层找到嵌套的文件或者文件夹的信息。

其余的 API 都是异曲同工之妙，主要是了解 `normalize` 的细节以及 `this.data` 的数据结构形式，你就能完全掌握这个库。

再说一下跟 stream 有关的两个 API。

### createReadStream

可读流：

```js
createReadStream(path, options) {
  let stream = new ReadableStream();
  let done = false;
  let data;
  try {
    data = this.readFileSync(path);
  } catch (e) {
    stream._read = function() {
      if (done) {
        return;
      }
      done = true;
      this.emit('error', e);
      this.push(null);
    };
    return stream;
  }
  options = options || { };
  options.start = options.start || 0;
  options.end = options.end || data.length;
  stream._read = function() {
    if (done) {
      return;
    }
    done = true;
    this.push(data.slice(options.start, options.end));
    this.push(null);
  };
  return stream;
}
```

可读流的实现是重写了 `_read` 方法。并且这个方法支持 `start` 与 `end` 的配置项来决定将文件的哪些内容读入可读流。可读流所有的数据都还是存在在 `this.data` 的属性上，因为 `this.readFileSync(path)` 就是从内存中拿到 Buffer，然后读入到可读流中。

### createWriteStream

可写流：

```js
createWriteStream(path) {
  let stream = new WritableStream();
  try {
    // Zero the file and make sure it is writable
    this.writeFileSync(path, new Buffer(0));
  } catch(e) {
    // This or setImmediate?
    stream.once('prefinish', function() {
      stream.emit('error', e);
    });
    return stream;
  }
  let bl = [ ], len = 0;
  stream._write = (chunk, encoding, callback) => {
    bl.push(chunk);
    len += chunk.length;
    this.writeFile(path, Buffer.concat(bl, len), callback);
  }
  return stream;
}
```

可读流的实现是重写了 `_write` 方法。createWriteStream 只需要传入路径字符串，内部会调用 writeFileSync 生成拥有 0 Buffer 的文件。只要调用了 `write` 方法并且传入 chunk，那么就会走到 `_write` 的逻辑，会把传入的 chunk 内容拼接到之前的 Buffer上。

# 总结

`webpackDevServer` 就是依赖这个库，将所有的数据都存在内存当中，这样在开发环境，不用通过文件系统打包出所有的文件，大大提高了开发效率。