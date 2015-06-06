# index.js

### 1. 模块依赖及变量定义

```javascript
/**
 * Module dependencies.
 */

var Route = require('./route');
var Layer = require('./layer');
var methods = require('methods');
var mixin = require('utils-merge');
var debug = require('debug')('express:router');
var deprecate = require('depd')('express');
var parseUrl = require('parseurl');
var utils = require('../utils');

/**
 * Module variables.
 */

var objectRegExp = /^\[object (\S+)\]$/;
var slice = Array.prototype.slice;
var toString = Object.prototype.toString;
```

### 2. 导出对象

该模块为`Router`的代码，导出对象源码如下：

```javascript
/**
 * Initialize a new `Router` with the given `options`.
 *
 * @param {Object} options
 * @return {Router} which is an callable function
 * @api public
 */

var proto = module.exports = function(options) {
  options = options || {};

  function router(req, res, next) {
    router.handle(req, res, next);
  }

  // mixin Router class functions
  router.__proto__ = proto;

  router.params = {};
  router._params = [];
  router.caseSensitive = options.caseSensitive;
  router.mergeParams = options.mergeParams;
  router.strict = options.strict;
  router.stack = [];

  return router;
};
```

该代码的简化版本如下：

```javascript
var Router = function() {
	function router() {
		router.doSomething();
	}
	router.__proto__ = Router;
	router.prop = 'some prop';
	return router;
};

Router.doSomething = function() {
	console.log('do something');
};

var router1 = Router();
var router2 = new Router();
```

在该例子中，`Router`事实上只是一个工厂函数，用来生成`router`，并且作为`router`的原型。在调用`Router`的时候，有没有`new`差别不大，因为最终都是返回了`router`。

回归到源码，该段代码主要做了如下几件事：

1. 生成一个`router`对象。事实上，`router`是一个函数，该函数有一些其它属性。
2. 将`router`的`__proto__`属性设置为`proto`，即让`router`继承自`proto`。
3. 设置`router`的一些属性值，主要包括：
	- `params`
	- `_params`
	- `caseSensitive`
	- `mergeParam`
	- `strict`
	- `stack`

在`application.js`中的`lazyrouter`中，正是通过`new Router({...})`来生成了`app._router`。

### 3. `proto.use`

该方法代码如下：

```javascript
/**
 * Use the given middleware function, with optional path, defaulting to "/".
 *
 * Use (like `.all`) will run for any http METHOD, but it will not add
 * handlers for those methods so OPTIONS requests will not consider `.use`
 * functions even if they could respond.
 *
 * The other difference is that _route_ path is stripped and not visible
 * to the handler function. The main effect of this feature is that mounted
 * handlers can operate without any code changes regardless of the "prefix"
 * pathname.
 *
 * @api public
 */

proto.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // default path to '/'
  // disambiguate router.use([fn])
  if (typeof fn !== 'function') {
    var arg = fn;

    while (Array.isArray(arg) && arg.length !== 0) {
      arg = arg[0];
    }

    // first arg is the path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }

  var callbacks = utils.flatten(slice.call(arguments, offset));

  if (callbacks.length === 0) {
    throw new TypeError('Router.use() requires middleware functions');
  }

  callbacks.forEach(function (fn) {
    if (typeof fn !== 'function') {
      throw new TypeError('Router.use() requires middleware function but got a ' + gettype(fn));
    }

    // add the middleware
    debug('use %s %s', path, fn.name || '<anonymous>');

    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    this.stack.push(layer);
  }, this);

  return this;
};
```

该方法的思路如下：

- 首先定义局部变量`offset`和`path`：
	- `offset`：表示从第几个参数开始为中间件函数，默认为0
	- `path`：路径，默认为`/`
- 当第一个参数不为函数的时候：
	- `while`循环中是针对第一个参数为数组的情况，即`[fn]`，`[[fn]]`等情况，此时让局部变量`arg`为数组或多层数组的第一项
	- 如果`arg`不为函数，则认为第一个参数为字符串，并将其赋值给`path`，同时设置`offset`为`1`
- 获取参数中的`offset`下标开始的所有参数，并进行扁平化操作，赋值给`callbacks`，即代表所有的中间件函数
- 如果没有中间件函数，则抛出错误
- 对`callbacks`进行`forEach`操作，即对于每个中间件函数：
	- 如果它不是一个函数，则报错
	- 如果是一个函数，则使用`path`和该函数构建一个`layer`对象，并加到`this.stack`中
- 返回该对象，从而可以进行链式操作

归结起来，该方法其实就是向`this.stack`数组中添加`layer`对象，`layer`是`router/layer.js`的导出对象的实例。

看如下例子：

```javascript
var express = require('express');
var app = express();

app.use(function foo() {});
app.use('/users', function bar() {}, function test() {});

console.log(app._router.stack);
```

输出结果为：

```javascript
[{
	handle: [Function: query],
	name: 'query',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: expressInit],
	name: 'expressInit',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: foo],
	name: 'foo',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: {
		/^\/?(?=\/|$)/i
		fast_slash: true
	},
	route: undefined
}, {
	handle: [Function: bar],
	name: 'bar',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: /^\/users\/?(?=\/|$)/i,
	route: undefined
}, {
	handle: [Function: test],
	name: 'test',
	params: undefined,
	path: undefined,
	keys: [],
	regexp: /^\/users\/?(?=\/|$)/i,
	route: undefined
}]
```

其中数组中的前两项，是因为在`application.js`的`lazyrouter`方法中，初始化`app._router`的时候，有如下代码：

```javascript
this._router.use(query(this.get('query parser fn')));
this._router.use(middleware.init(this));
```

### 4. `proto.param`

该方法源码如下：

```javascript
/**
 * Map the given param placeholder `name`(s) to the given callback.
 *
 * Parameter mapping is used to provide pre-conditions to routes
 * which use normalized placeholders. For example a _:user_id_ parameter
 * could automatically load a user's information from the database without
 * any additional code,
 *
 * The callback uses the same signature as middleware, the only difference
 * being that the value of the placeholder is passed, in this case the _id_
 * of the user. Once the `next()` function is invoked, just like middleware
 * it will continue on to execute the route, or subsequent parameter functions.
 *
 * Just like in middleware, you must either respond to the request or call next
 * to avoid stalling the request.
 *
 *  app.param('user_id', function(req, res, next, id){
 *    User.find(id, function(err, user){
 *      if (err) {
 *        return next(err);
 *      } else if (!user) {
 *        return next(new Error('failed to load user'));
 *      }
 *      req.user = user;
 *      next();
 *    });
 *  });
 *
 * @param {String} name
 * @param {Function} fn
 * @return {app} for chaining
 * @api public
 */

proto.param = function param(name, fn) {
  // param logic
  if (typeof name === 'function') {
    deprecate('router.param(fn): Refactor to use path params');
    this._params.push(name);
    return;
  }

  // apply param functions
  var params = this._params;
  var len = params.length;
  var ret;

  if (name[0] === ':') {
    deprecate('router.param(' + JSON.stringify(name) + ', fn): Use router.param(' + JSON.stringify(name.substr(1)) + ', fn) instead');
    name = name.substr(1);
  }

  for (var i = 0; i < len; ++i) {
    if (ret = params[i](name, fn)) {
      fn = ret;
    }
  }

  // ensure we end up with a
  // middleware function
  if ('function' != typeof fn) {
    throw new Error('invalid param() call for ' + name + ', got ' + fn);
  }

  (this.params[name] = this.params[name] || []).push(fn);
  return this;
};
```

该方法的主要思路如下：

- 如果第一个参数为函数，即调用方式为`param(fn)`。由于`app.param(callback)`已经在4.11.0版本中废弃，因此处理方式是将参数中的函数放在`this._params`中，然后返回
- 如果`name`以冒号开头，则去掉冒号
- 对于`name`，使用`this._params`中的每个函数对其进行依次处理。加入之前没有调用过`param(fn)`的话，相当于跳过此步
- 如果`fn`参数不是函数，报错
- 向`this.params[name]`数组中添加`fn`
- 返回`this`从而进行链式调用

总的来说，该方法其实是对`this.params`进行设置，而`this.params`则是如下的数据结构：

```javascript
{
	foo: [],
	bar: []
}
```

例如，我们执行如下代码：

```javascript
var express = require('express');
var app = express();

app.param('user_id', function foo() {});
app.param('user_id', function bar() {});
app.param('user_name', function test() {});
```

其中`app.param`是`proto.param`的代理函数。输出结果为：

```javascript
{
	user_id: [
		[Function: foo],
		[Function: bar]
	],
	user_name: [
		[Function: test]
	]
}
```

### 5. `proto.route`

该方法的作用是创建一条新的路由，源码如下：

```javascript
/**
 * Create a new Route for the given path.
 *
 * Each route contains a separate middleware stack and VERB handlers.
 *
 * See the Route api documentation for details on adding handlers
 * and middleware to routes.
 *
 * @param {String} path
 * @return {Route}
 * @api public
 */

proto.route = function(path){
  var route = new Route(path);

  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));

  layer.route = route;

  this.stack.push(layer);
  return route;
};
```

首先使用`path`创建一个`route`对象，然后使用`path`和`route.dispatch`创建一个`layer`对象，将`layer.route`属性值设置为`route`，并把`layer`放在`this.stack`数组中。

### 6. VERB方法

源码如下：

```javascript
// create Router#VERB functions
methods.concat('all').forEach(function(method){
  proto[method] = function(path){
    var route = this.route(path)
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

其实就是使用参数中的`path`进行`this.route(path)`调用，创建一个`route`，然后调用`route`的VERB方法。

### 7. `proto.handle`

该方法主要用来处理接收到的HTTP请求，会被`app.handle`调用。该方法的源码较长，下面是源码的主要结构：

```javascript
proto.handle = function(req, res, done) {
  // ... ...

  next();

  function next(err) {
    // ... ...
  }

  function trim_prefix(layer, layerError, layerPath, path) {
    // ... ...
  }
}
```

因此，该方法主要是调用了`next()`，下面来看`next`函数。由于`next`的源码依然很长且比较繁琐，因此主要看比较关键的代码。在调用`next`之前，首先定义了`idx`和`stack`这两个变量，分别表示当前中间件的下表和中间件列表。

```javascript
// no more matching layers
if (idx >= stack.length) {
  setImmediate(done, layerError);
  return;
}
```

这段代码表明当遍历完了所有的中间件时，执行`done()`。如果遍历未结束，执行`while`循环，代码如下：

```javascript
while (match !== true && idx < stack.length) {
  layer = stack[idx++];
  match = matchLayer(layer, path);
  route = layer.route;

  if (typeof match !== 'boolean') {
    // hold on to layerError
    layerError = layerError || match;
  }

  if (match !== true) {
    continue;
  }

  if (!route) {
    // process non-route handlers normally
    continue;
  }

  if (layerError) {
    // routes do not match with a pending error
    match = false;
    continue;
  }

  var method = req.method;
  var has_method = route._handles_method(method);

  // build up automatic options response
  if (!has_method && method === 'OPTIONS') {
    appendMethods(options, route._options());
  }

  // don't even bother matching route
  if (!has_method && method !== 'HEAD') {
    match = false;
    continue;
  }
}
```

在这里，`layer`表示当前的中间件，`match`表示当前的中间件与当前路径是否匹配。其中`matchLayer(layer, path)`其实是调用了`layer.match(path)`。

`while`循环的逻辑如下：

- 如果`match`不为`true`，即该中间件与路径不匹配，则执行continue，对下一个中间件进行判断
- 如果`match`为`true`但是`route`不存在，则说明是一个非路由中间件，执行continue，但此时由于`match`不再满足循环条件，因此会跳出循环
- 如果`match`为`true`且`route`存在，则说明是一个路由中间件，则将继续对HTTP请求方法做一些处理，首先判断该路由是否能够处理该HTTP方法，即`has_method`
    - 如果`has_method`为`false`且HTTP方法为`OPTIONS`，则执行`appendMethods(options, route._options())`
    - 如果`has_method`为`false`且HTTP方法不为`HEAD`，则设置`match`为`false`，也就是说，该路由无法处理该请求，此时由于`match`依然满足循环条件，因此会对下一个中间件进行判断
    - 如果`has_method`为`true`，则由于`match`不再满足循环条件，因此会跳出循环

总的来说，该循环的主要作用就是从当前下标开始找出第一个能够处理该HTTP请求的中间件。如果是非路由中间件，则只要匹配路径即可；如果是路由中间件，则需要同时匹配路径和HTTP请求方法。

在`while`之后，如果`match`不为`true`，则说明已经遍历完了所有的中间件，因此直接执行`done()`；否则调用`self.process_params`，即进行参数的预处理，并调用回调函数。代码如下：

```javascript
// this should be done for the layer
self.process_params(layer, paramcalled, req, res, function (err) {
  if (err) {
    return next(layerError || err);
  }

  if (route) {
    return layer.handle_request(req, res, next);
  }

  trim_prefix(layer, layerError, layerPath, path);
});
```

在回调函数中，如果`route`存在，即对于路由中间件，调用`layer.handle_request(req, res, next)`，如果不是路由中间件，则会调用`trim_prefix(layer, layerError, layerPath, path)`，对路径进行处理后，才调用`layer.handle_request(req, res, next)`。

### 8. `proto.process_params`

该方法主要是对参数进行预处理，即在接收到HTTP请求后，在对请求处理之前，对请求路径中的参数进行一定的预处理。该方法会在`proto.handle`中被调用。该方法源码主要结构如下：

```javascript
/**
 * Process any parameters for the layer.
 *
 * @api private
 */

proto.process_params = function(layer, called, req, res, done) {
  var params = this.params;

  // captured parameters from the layer, keys and values
  var keys = layer.keys;

  // fast track
  if (!keys || keys.length === 0) {
    return done();
  }

  var i = 0;
  var name;
  var paramIndex = 0;
  var key;
  var paramVal;
  var paramCallbacks;
  var paramCalled;

  // process params in order
  // param callbacks can be async
  function param(err) {
    // ... ...
  }

  // single param callbacks
  function paramCallback(err) {
    // ... ...
  }

  param();
};
```

其中，变量`params`形式示例如下：

```javascript
{
  user_id: [
    [Function: foo],
    [Function: bar]
  ],
  user_name: [
    [Function: test]
  ]
}
```

变量`keys`形式示例如下：

```javascript
[{
  name: 'user_id',
  // ... ...
}, {
  name: 'user_name',
  // ... ...
}]
```

也就是所，`keys`表示的是路径中应当接收的参数，`params`存放的是所有的参数预处理函数，而路径中实际接收到的参数保存在`req.params`中。

在该函数的开始，先对`keys`进行判断，如果它不存在或者为空数组，则直接执行`done()`，否则，会执行`param()`。`param`函数的源码如下：

```javascript
// process params in order
// param callbacks can be async
function param(err) {
  if (err) {
    return done(err);
  }

  if (i >= keys.length ) {
    return done();
  }

  paramIndex = 0;
  key = keys[i++];

  if (!key) {
    return done();
  }

  name = key.name;
  paramVal = req.params[name];
  paramCallbacks = params[name];
  paramCalled = called[name];

  if (paramVal === undefined || !paramCallbacks) {
    return param();
  }

  // param previously called with same value or error occurred
  if (paramCalled && (paramCalled.error || paramCalled.match === paramVal)) {
    // restore value
    req.params[name] = paramCalled.value;

    // next param
    return param(paramCalled.error);
  }

  called[name] = paramCalled = {
    error: null,
    match: paramVal,
    value: paramVal
  };

  paramCallback();
}
```