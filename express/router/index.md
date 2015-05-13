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