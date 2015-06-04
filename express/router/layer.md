# layer.js

### 1. 模块依赖及变量定义

```javascript
/**
 * Module dependencies.
 */

var pathRegexp = require('path-to-regexp');
var debug = require('debug')('express:router:layer');

/**
 * Module variables.
 */

var hasOwnProperty = Object.prototype.hasOwnProperty;
```

其中`path-to-regexp`模块的作用是将一个Express的表示路径的字符串转换成一个正则表达式。例如：

```javascript
var pathToRegexp = require('path-to-regexp');

var keys = [];
var re = pathToRegexp('/foo/:bar', keys);
// re = /^\/foo\/([^\/]+?)\/?$/i
// keys = [{
//  name: 'bar',
//  prefix: '/',
//  delimiter: '/',
//  optional: false,
//  repeat: false,
//  pattern: '[^\\/]+?'
// }]

re.exec('/foo/test');
// => ['/foo/test', 'test']
```

### 2. 导出对象

该模块的导出对象为`Layer`类，源码如下：

```javascript
/**
 * Expose `Layer`.
 */

module.exports = Layer;

function Layer(path, options, fn) {
  if (!(this instanceof Layer)) {
    return new Layer(path, options, fn);
  }

  debug('new %s', path);
  options = options || {};

  this.handle = fn;
  this.name = fn.name || '<anonymous>';
  this.params = undefined;
  this.path = undefined;
  this.regexp = pathRegexp(path, this.keys = [], options);

  if (path === '/' && options.end === false) {
    this.regexp.fast_slash = true;
  }
}
```

其中主要是对一些实例属性的设置，主要包括：

- `handle`
- `name`
- `params`
- `path`
- `regexp`
- `keys`

然后当`path`为`/`且`options.end`为`false`的时候，会设置`this.regexp.fast_slash`的值为`true`。`Layer`的创建主要是在`Route`和`Router`中。

- 在`Route`模块中，当调用`all()`或`METHOD()`方法的时候，会创建`Layer`，但是创建方式为`Layer('/', {}, fn)`，及第二个参数为一个空对象，因此此时的`Layer`实例并未进行`regexp.fast_slash`设置。
- 在`Router`模块中，当调用`use()`或`route()`方法的时候，会创建`Layer()`
    - `use()`：`options.end`为`false`，因此当`path`为`/`时，`fast_slash`为`true`
    - `route()`：`options.end`为`true`，因此不设置`fast_slash`

因此，只有当`path`为`/`且该中间件不是路由中间件的时候，才会设置`this.regexp.fast_slash`的值为`true`。

### 3. `match`

该方法用来判断该`Layer`对象是否与给定的`path`相匹配，并进行后续处理。源码如下：

```javascript
/**
 * Check if this route matches `path`, if so
 * populate `.params`.
 *
 * @param {String} path
 * @return {Boolean}
 * @api private
 */

Layer.prototype.match = function match(path) {
  if (path == null) {
    // no path, nothing matches
    this.params = undefined;
    this.path = undefined;
    return false;
  }

  if (this.regexp.fast_slash) {
    // fast path non-ending match for / (everything matches)
    this.params = {};
    this.path = '';
    return true;
  }

  var m = this.regexp.exec(path);

  if (!m) {
    this.params = undefined;
    this.path = undefined;
    return false;
  }

  // store values
  this.params = {};
  this.path = m[0];

  var keys = this.keys;
  var params = this.params;
  var prop;
  var n = 0;
  var key;
  var val;

  for (var i = 1, len = m.length; i < len; ++i) {
    key = keys[i - 1];
    prop = key
      ? key.name
      : n++;
    val = decode_param(m[i]);

    if (val !== undefined || !(hasOwnProperty.call(params, prop))) {
      params[prop] = val;
    }
  }

  return true;
};
```

思路如下：

- 如果`path`无效，则设置`this.params`和`this.path`为`undefined`，返回`false`
- 如果`this.regexp.fast_slash`为`true`，则设置`this.params`为空对象，设置`this.path`为空字符串，返回`true`
- 执行`this.regexp.exec(path)`判断是否匹配，如果不匹配，则设置`this.params`和`this.path`为`undefined`，返回`false`
- 如果匹配，设置`this.path`为`m[0]`，并对`this.params`进行赋值，然后返回`true`

### 4. `handle_request`和`handle_error`

该方法用来处理HTTP请求，源码如下：

```javascript
/**
 * Handle the request for the layer.
 *
 * @param {Request} req
 * @param {Response} res
 * @param {function} next
 * @api private
 */

Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;

  if (fn.length > 3) {
    // not a standard request handler
    return next();
  }

  try {
    fn(req, res, next);
  } catch (err) {
    next(err);
  }
};
```

如果处理函数的参数个数大于3，则认为不是一个标准的处理函数，因此不执行，而是直接执行`next()`；否则执行`fn`。

例如下面的例子，处理函数就不会执行：

```javascript
app.use('/', function(req, res, next, foo) {
  console.log('hello world');
  next();
});
```

对于`Router`中存放的`Layer`，如果是普通中间件的话，调用的就是使用`app.use()`时注册的处理函数；如果是路由中间件的话，调用的是`route.dispatch()`方法，相关处理函数的注册在`Router`的`route()`方法中，代码如下：

```javascript
var layer = new Layer(path, {
  sensitive: this.caseSensitive,
  strict: this.strict,
  end: true
}, route.dispatch.bind(route));
```

`handle_error`方法与`handle_request`的逻辑基本类似，不做赘述。