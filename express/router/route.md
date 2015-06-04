# route.js

### 1. 模块依赖

```javascript
/**
 * Module dependencies.
 */

var debug = require('debug')('express:router:route');
var Layer = require('./layer');
var methods = require('methods');
var utils = require('../utils');
```

### 2. 导出对象

该模块的导出对象即为`Route`类，源码如下：

```javascript
/**
 * Expose `Route`.
 */

module.exports = Route;

/**
 * Initialize `Route` with the given `path`,
 *
 * @param {String} path
 * @api private
 */

function Route(path) {
  debug('new %s', path);
  this.path = path;
  this.stack = [];

  // route handlers for various http methods
  this.methods = {};
}
```

一个`Router`实例主要有如下属性：

- `path`
- `stack`
- `methods`

### 3. `Route.prototype.all`

该方法为所有的HTTP请求方法添加一条路由，源码如下：

```javascript
/**
 * Add a handler for all HTTP verbs to this route.
 *
 * Behaves just like middleware and can respond or call `next`
 * to continue processing.
 *
 * You can use multiple `.all` call to add multiple handlers.
 *
 *   function check_something(req, res, next){
 *     next();
 *   };
 *
 *   function validate_user(req, res, next){
 *     next();
 *   };
 *
 *   route
 *   .all(validate_user)
 *   .all(check_something)
 *   .get(function(req, res, next){
 *     res.send('hello world');
 *   });
 *
 * @param {function} handler
 * @return {Route} for chaining
 * @api public
 */

Route.prototype.all = function(){
  var callbacks = utils.flatten([].slice.call(arguments));
  callbacks.forEach(function(fn) {
    if (typeof fn !== 'function') {
      var type = {}.toString.call(fn);
      var msg = 'Route.all() requires callback functions but got a ' + type;
      throw new Error(msg);
    }

    var layer = Layer('/', {}, fn);
    layer.method = undefined;

    this.methods._all = true;
    this.stack.push(layer);
  }, this);

  return this;
};
```

该方法主要思路：获取所有参数并进行扁平化处理。对于每个参数，如果不是函数，则报错，如果是函数，则构造一个`Layer`对象，并将构造的`Layer`对象添加到`this.stack`数组中。同时将`this.methods._all`设置为`true`。

### 4. VERB方法

```javascript
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var callbacks = utils.flatten([].slice.call(arguments));

    callbacks.forEach(function(fn) {
      if (typeof fn !== 'function') {
        var type = {}.toString.call(fn);
        var msg = 'Route.' + method + '() requires callback functions but got a ' + type;
        throw new Error(msg);
      }

      debug('%s %s', method, this.path);

      var layer = Layer('/', {}, fn);
      layer.method = method;

      this.methods[method] = true;
      this.stack.push(layer);
    }, this);
    return this;
  };
});
```

该方法与`Route.prototype.all`方法非常类似。首先`methods`是一个依赖模块，其导出对象就是一个数组，包括所有的HTTP方法。因此这段代码就是为`Route.prototype`添加`get`、`post`等方法，且整体逻辑与`Route.prototype.all`基本类似，不同点为：

- `layer.method`设置为`method`，而不是`undefined`
- 在`this.methods`中，设置相关的`method`属性为`true`

### 5. `Route.prototype._handles_method`

该方法用来判断该路由能否处理给定的HTTP请求方法，源码如下：

```javascript
Route.prototype._handles_method = function _handles_method(method) {
  if (this.methods._all) {
    return true;
  }

  method = method.toLowerCase();

  if (method === 'head' && !this.methods['head']) {
    method = 'get';
  }

  return Boolean(this.methods[method]);
};
```

如果`this.methods._all`为`true`，则可以处理一切HTTP方法，因此直接返回`true`。

如果`method`为`head`且`this.methods`中未对`head`方法进行设置，则改变`method`的值为`get`，即判断是否可以处理GET方法。

最后，判断`this.methods`中是否设置了相应的HTTP方法，并返回。

### 6. `Route.prototype._options`

该方法返回该路由所支持的所有的HTTP请求方法，源码如下：

```javascript
/**
 * @return {Array} supported HTTP methods
 * @api private
 */

Route.prototype._options = function _options() {
  var methods = Object.keys(this.methods);

  // append automatic head
  if (this.methods.get && !this.methods.head) {
    methods.push('head');
  }

  for (var i = 0; i < methods.length; i++) {
    // make upper case
    methods[i] = methods[i].toUpperCase();
  }

  return methods;
};
```

这里需要注意的是，如果有`GET`而没有`HEAD`方法，则把`HEAD`方法也加进去。

### 7. `Route.prototype.dispatch`

该方法主要是对HTTP请求进行处理分发，源码如下：

```javascript
/**
 * dispatch req, res into this route
 *
 * @api private
 */

Route.prototype.dispatch = function(req, res, done){
  var idx = 0;
  var stack = this.stack;
  if (stack.length === 0) {
    return done();
  }

  var method = req.method.toLowerCase();
  if (method === 'head' && !this.methods['head']) {
    method = 'get';
  }

  req.route = this;

  next();

  function next(err) {
    if (err && err === 'route') {
      return done();
    }

    var layer = stack[idx++];
    if (!layer) {
      return done(err);
    }

    if (layer.method && layer.method !== method) {
      return next(err);
    }

    if (err) {
      layer.handle_error(err, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

首先定义了两个变量，`idx`表示当前下标，`stack`表示该路由的处理函数列表。然后在进行了一些基本的处理后，会调用`next()`。`next`函数的思路如下：

- 如果有错误，则直接调用`done()`
- 获取`layer`，如果`layer`不存在，则调用`done`
- 如果`layer.method`与当前的HTTP请求方法不匹配，则跳过，执行`next()`，去获取下一个处理函数
- 如果`layer.method`与当前的HTTP请求方法匹配，则根据有无错误选择执行`layer.handle_error`或`layer.handle_request`

例如下面的例子：

```javascript
app.route('/users')
  .post(function foo(req, res, next) {
    // do something
    next();
  }).get(function bar(req, res, next) {
    // do something
    next();
  });
```

当通过`GET`方法访问`/users`的时候，会先判断`handle`属性为`foo`的Layer，发现HTTP请求方法不匹配，因此跳过。然后判断`handle`属性为`bar`的Layer，发现匹配，于是调用该Layer的`handle_request`方法，从而调用`bar()`。