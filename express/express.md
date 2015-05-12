# express.js

### 1. createApplication

在使用express的时候，首先是：

```javascript
var express = require('express');
var app = express();
```

在源码中，有这样一句：

```javascript
exports = module.exports = createApplication;
```

因此，express.js的导出对象实际上就是`createApplication`这个方法，该方法定义如下：

```javascript
/**
 * Create an express application.
 *
 * @return {Function}
 * @api public
 */

function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };

  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);

  app.request = { __proto__: req, app: app };
  app.response = { __proto__: res, app: app };
  app.init();
  return app;
}
```

可以看到，该方法实际上就是一个工厂函数，用来生成一个app的实例。而`app`事实上是一个函数，该函数会有一些其它属性。

其中有两句：

```javascript
mixin(app, EventEmitter.prototype, false);
mixin(app, proto, false);
```

`mixin`是一个依赖模块，它其实就是一个extend操作，这两行代码就是吧`EventEmitter.prototype`和`proto`的属性扩展到`app`上。

### 2. module.exports

下面的代码就是为导出对象添加一些属性，综合起来，`module.exports`，即`express`，有如下属性：

- `application`
- `request`
- `response`
- `Route`
- `Router`
- `query`
- `static`
