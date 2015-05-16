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