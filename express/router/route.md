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