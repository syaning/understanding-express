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

console.log(app._router);
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