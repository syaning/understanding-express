### view.js

### 1. 模块依赖及变量定义

源码如下：

```javascript
/**
 * Module dependencies.
 */

var debug = require('debug')('express:view');
var path = require('path');
var fs = require('fs');
var utils = require('./utils');

/**
 * Module variables.
 * @private
 */

var dirname = path.dirname;
var basename = path.basename;
var extname = path.extname;
var join = path.join;
var resolve = path.resolve;
```

### 2. 导出对象

源码如下：

```javascript
/**
 * Expose `View`.
 */

module.exports = View;

/**
 * Initialize a new `View` with the given `name`.
 *
 * Options:
 *
 *   - `defaultEngine` the default template engine name
 *   - `engines` template engine require() cache
 *   - `root` root path for view lookup
 *
 * @param {String} name
 * @param {Object} options
 * @api private
 */

function View(name, options) {
  options = options || {};
  this.name = name;
  this.root = options.root;
  var engines = options.engines;
  this.defaultEngine = options.defaultEngine;
  var ext = this.ext = extname(name);
  if (!ext && !this.defaultEngine) throw new Error('No default engine was specified and no extension was provided.');
  if (!ext) name += (ext = this.ext = ('.' != this.defaultEngine[0] ? '.' : '') + this.defaultEngine);
  this.engine = engines[ext] || (engines[ext] = require(ext.slice(1)).__express);
  this.path = this.lookup(name);
}
```

构造函数思路如下：

- 设置`name`，`root`和`defaultEngine`
- 获取文件扩展名并设置`ext`
- 如果`ext`为空且`defaultEngine`不存在，则报错
- 如果`ext`为空，则使用`defaultEngine`相对应的扩展名
- 根据`ext`来获取模板引擎并设置`engine`
- 调用`this.lookup(name)`来设置`path`

### 3. `View.prototype.lookup`

该方法主要用于查找模板文件，源码如下：

```javascript
/**
 * Lookup view by the given `name`
 *
 * @param {String} name
 * @return {String}
 * @api private
 */

View.prototype.lookup = function lookup(name) {
  var path;
  var roots = [].concat(this.root);

  debug('lookup "%s"', name);

  for (var i = 0; i < roots.length && !path; i++) {
    var root = roots[i];

    // resolve the path
    var loc = resolve(root, name);
    var dir = dirname(loc);
    var file = basename(loc);

    // resolve the file
    path = this.resolve(dir, file);
  }

  return path;
};
```

由于`this.root`（事实上就是`app.settings`中的`views`）可以是一个字符串或数组，因此首先通过`concat`操作将其统一为一个数组，然后对数组中的每一项，去查找`name`，如果找到了，则不再到下一个目录中查找，并返回结果。

### 4. `View.prototype.resolve`

该方法主要用于解析文件名，源码如下：

```javascript
/**
 * Resolve the file within the given directory.
 *
 * @param {string} dir
 * @param {string} file
 * @private
 */

View.prototype.resolve = function resolve(dir, file) {
  var ext = this.ext;
  var path;
  var stat;

  // <path>.<ext>
  path = join(dir, file);
  stat = tryStat(path);

  if (stat && stat.isFile()) {
    return path;
  }

  // <path>/index.<ext>
  path = join(dir, basename(file, ext), 'index' + ext);
  stat = tryStat(path);

  if (stat && stat.isFile()) {
    return path;
  }
};

/**
 * Return a stat, maybe.
 *
 * @param {string} path
 * @return {fs.Stats}
 * @private
 */

function tryStat(path) {
  debug('stat "%s"', path);

  try {
    return fs.statSync(path);
  } catch (e) {
    return undefined;
  }
}
```

该方法接收两个参数，`dir`和`file`，分别表示目录名和文件名。例如目录名为`foo`，文件名为`bar.html`，则会首先查找`foo/bar.html`是否存在，如果存在，则返回该路径；否则查找`foo/bar/index.html`是否存在，如果存在，返回该路径；否则路径解析失败，默认返回`undefined`。

### 5. `View.prototype.render`

该方法用于渲染模板，其实就是调用了模板引擎函数。源码如下：

```javascript
/**
 * Render with the given `options` and callback `fn(err, str)`.
 *
 * @param {Object} options
 * @param {Function} fn
 * @api private
 */

View.prototype.render = function render(options, fn) {
  debug('render "%s"', this.path);
  this.engine(this.path, options, fn);
};
```