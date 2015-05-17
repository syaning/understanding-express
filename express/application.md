# application.js

### 1. 模块依赖

该模块主要依赖如源码所示：

```javascript
var finalhandler = require('finalhandler');
var flatten = require('./utils').flatten;
var Router = require('./router');
var methods = require('methods');
var middleware = require('./middleware/init');
var query = require('./middleware/query');
var debug = require('debug')('express:application');
var View = require('./view');
var http = require('http');
var compileETag = require('./utils').compileETag;
var compileQueryParser = require('./utils').compileQueryParser;
var compileTrust = require('./utils').compileTrust;
var deprecate = require('depd')('express');
var merge = require('utils-merge');
var resolve = require('path').resolve;
var slice = Array.prototype.slice;
```

### 2. `app`及`init`

```javascript
var app = exports = module.exports = {};
```

因此，该模块的导出对象为一个对象字面量，后面会陆续在该对象上增加属性。其`init`方法如下：

```javascript
/**
 * Initialize the server.
 *
 *   - setup default configuration
 *   - setup default middleware
 *   - setup route reflection methods
 *
 * @api private
 */

app.init = function(){
  this.cache = {};
  this.settings = {};
  this.engines = {};
  this.defaultConfiguration();
};
```

该方法为初始化函数，会在express.js的`createApplication`方法中调用，因此在执行如下代码的时候，会调用该初始化函数：

```javascript
var express = require('express');
var app = express();
```

通过源码，可以得知，初始化的时候，主要是初始化`cache`，`settings`，`engines`，并调用`defaultConfiguration`函数进行默认配置。

### 3. `app.listen`

在不使用框架的时候，我们可以通过如下方式创建一个服务器：

```javascript
var http = require('http');

http.createServer(function(req, res) {
        res.write('hello world');
        res.end();
    })
    .listen(8000);
```

下面来看`app.listen`方法：

```javascript
/**
 * Listen for connections.
 *
 * A node `http.Server` is returned, with this
 * application (which is a `Function`) as its
 * callback. If you wish to create both an HTTP
 * and HTTPS server you may do so with the "http"
 * and "https" modules as shown here:
 *
 *    var http = require('http')
 *      , https = require('https')
 *      , express = require('express')
 *      , app = express();
 *
 *    http.createServer(app).listen(80);
 *    https.createServer({ ... }, app).listen(443);
 *
 * @return {http.Server}
 * @api public
 */

app.listen = function(){
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```

注意，在该模块中，`app`只是一个对象字面量，而在使用express进行开发的时候，`app`事实上是一个函数，该函数在express.js的`createApplication`方法中定义。如下所示：

```javascript
var app = function(req, res, next) {
  app.handle(req, res, next);
};
```

正因为如此，在源码中会有`http.createServer(this)`。所以，所有的HTTP请求，事实上都是交给`app`函数去处理了。而`app`函数的执行则是又调用了它的`handle`方法。

### 4. `settings`相关

`app`的`settings`属性为一个对象字面量，因此以键值对的形式保存着一系列设置信息。首先来看`set`方法：

```javascript
/**
 * Assign `setting` to `val`, or return `setting`'s value.
 *
 *    app.set('foo', 'bar');
 *    app.get('foo');
 *    // => "bar"
 *
 * Mounted servers inherit their parent server's settings.
 *
 * @param {String} setting
 * @param {*} [val]
 * @return {Server} for chaining
 * @api public
 */

app.set = function(setting, val){
  if (arguments.length === 1) {
    // app.get(setting)
    return this.settings[setting];
  }

  // set value
  this.settings[setting] = val;

  // trigger matched settings
  switch (setting) {
    case 'etag':
      debug('compile etag %s', val);
      this.set('etag fn', compileETag(val));
      break;
    case 'query parser':
      debug('compile query parser %s', val);
      this.set('query parser fn', compileQueryParser(val));
      break;
    case 'trust proxy':
      debug('compile trust proxy %s', val);
      this.set('trust proxy fn', compileTrust(val));

      // trust proxy inherit back-compat
      Object.defineProperty(this.settings, trustProxyDefaultSymbol, {
        configurable: true,
        value: false
      });

      break;
  }

  return this;
};
```

该方法既可以用来设置选项，也可以用来获取某个设置选项的值。首先判断当参数长度为1的时候，用作getter，直接返回设置的值。否则的话进行设置。

当设置的选项是`etag`，`query parser`或`trust proxy`，会触发相关的额外设置。

在`set`方法的基础上，有如下四个方法，该四个方法都是接受一个`setting`参数：

- `enabled`：判断setting是否启用
- `disabled`：判断setting是否禁用
- `enable`：启用setting
- `disable`：禁用setting

### 5. 默认配置`defaultConfiguration`

在`app`执行`init`的时候，会调用`defaultConfiguration`方法，该方法主要是初始化一些配置选项。源码如下：

```javascript
/**
 * Initialize application configuration.
 *
 * @api private
 */

app.defaultConfiguration = function(){
  // default settings
  this.enable('x-powered-by');
  this.set('etag', 'weak');
  var env = process.env.NODE_ENV || 'development';
  this.set('env', env);
  this.set('query parser', 'extended');
  this.set('subdomain offset', 2);
  this.set('trust proxy', false);

  // trust proxy inherit back-compat
  Object.defineProperty(this.settings, trustProxyDefaultSymbol, {
    configurable: true,
    value: true
  });

  debug('booting in %s mode', env);

  this.on('mount', function onmount(parent) {
    // inherit trust proxy
    if (this.settings[trustProxyDefaultSymbol] === true
      && typeof parent.settings['trust proxy fn'] === 'function') {
      delete this.settings['trust proxy'];
      delete this.settings['trust proxy fn'];
    }

    // inherit protos
    this.request.__proto__ = parent.request;
    this.response.__proto__ = parent.response;
    this.engines.__proto__ = parent.engines;
    this.settings.__proto__ = parent.settings;
  });

  // setup locals
  this.locals = Object.create(null);

  // top-most app is mounted at /
  this.mountpath = '/';

  // default locals
  this.locals.settings = this.settings;

  // default configuration
  this.set('view', View);
  this.set('views', resolve('views'));
  this.set('jsonp callback name', 'callback');

  if (env === 'production') {
    this.enable('view cache');
  }

  Object.defineProperty(this, 'router', {
    get: function() {
      throw new Error('\'app.router\' is deprecated!\nPlease see the 3.x to 4.x migration guide for details on how to update your app.');
    }
  });
};
```

这段代码主要做了以下几件事情：

1. 设置了一些选项，主要包括：
    - `x-powered-by`
    - `etag`
    - `env`
    - `query parser`
    - `subdomain offset`
    - `trust proxy`
    - `@@symbol:trust_proxy_default`
    - `view`
    - `views`
    - `jsonp callback name`
    - `view cache`
2. 注册`mount`事件的回调函数
3. 为`app`增加一些属性，主要包括：
    - `locals`
    - `mountpath`

### 6. 初始化Router`lazyrouter`

下面来看`lazyrouter`方法，该方法主要是初始化`Router`对象。该方法的源码如下：

```javascript
/**
 * lazily adds the base router if it has not yet been added.
 *
 * We cannot add the base router in the defaultConfiguration because
 * it reads app settings which might be set after that has run.
 *
 * @api private
 */
app.lazyrouter = function() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```

通过查看模块依赖的相关代码，可以发现`Router`其实就是`./router/index.js`模块的导出对象。

该方法之所叫做`lazyrouter`，是因为只有当需要`Router`的时候才初始化该对象。注释里也解释了，之所以不在`defaultConfiguration`里就把这一部分做掉，是因为该部分代码的执行需要依赖于一些设置选项，如`case sensitive routing`，`strict routing`和`query parser fn`，而这些设置可能在`defaultConfiguration`后更改。

该方法执行后，`app`就有了一个`_router`属性，其值是一个`Router`对象。

### 7. `app.route`

该方法代码如下：

```javascript
/**
 * Proxy to the app `Router#route()`
 * Returns a new `Route` instance for the _path_.
 *
 * Routes are isolated middleware stacks for specific paths.
 * See the Route api docs for details.
 *
 * @api public
 */

app.route = function(path){
  this.lazyrouter();
  return this._router.route(path);
};
```

该方法只是`Router.route`的一个代理方法，详细实现可以参考`Router`的`route`方法。

### 8. VERB方法

```javascript
/**
 * Delegate `.VERB(...)` calls to `router.VERB(...)`.
 */

methods.forEach(function(method){
  app[method] = function(path){
    if ('get' == method && 1 == arguments.length) return this.set(path);

    this.lazyrouter();

    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

这段代码主要是实现`app.get`、`app.post`等HTTP动作相关的方法。`methods`为一个依赖模块，其本身是一个数组，包含所有的HTTP方法。该方法的思路如下：

- 如果`method`为`get`且只有一个参数，即`app.get(path)`，则此时该方法当作`getter`来使用，通过调用`this.set(path)`来获取相关配置
- 执行`this.lazyrouter`
- 创建`route`对象，并注册处理函数。举例来说，例如`app.get('/users', fn)`，则首先使用`/users`创建一个`route`对象，然后调用`route.get(fn)`
- 返回`this`从而可以链式调用

### 9. `app.all`

该方法源码如下：

```javascript
/**
 * Special-cased "all" method, applying the given route `path`,
 * middleware, and callback to _every_ HTTP method.
 *
 * @param {String} path
 * @param {Function} ...
 * @return {app} for chaining
 * @api public
 */

app.all = function(path){
  this.lazyrouter();

  var route = this._router.route(path);
  var args = slice.call(arguments, 1);
  methods.forEach(function(method){
    route[method].apply(route, args);
  });

  return this;
};
```

该方法会创建一个`route`，然后依次调用其相关的HTTP方法，举例来说，加入使用了`app.all('/users', fn)`，则首先使用`/users`来创建一个`route`对象，然后调用`route.get(fn)`、`route.post(fn)`……

### 10. `app.engine`

该方法主要是注册模板引擎，源码如下：

```javascript
/**
 * Register the given template engine callback `fn`
 * as `ext`.
 *
 * By default will `require()` the engine based on the
 * file extension. For example if you try to render
 * a "foo.jade" file Express will invoke the following internally:
 *
 *     app.engine('jade', require('jade').__express);
 *
 * For engines that do not provide `.__express` out of the box,
 * or if you wish to "map" a different extension to the template engine
 * you may use this method. For example mapping the EJS template engine to
 * ".html" files:
 *
 *     app.engine('html', require('ejs').renderFile);
 *
 * In this case EJS provides a `.renderFile()` method with
 * the same signature that Express expects: `(path, options, callback)`,
 * though note that it aliases this method as `ejs.__express` internally
 * so if you're using ".ejs" extensions you dont need to do anything.
 *
 * Some template engines do not follow this convention, the
 * [Consolidate.js](https://github.com/tj/consolidate.js)
 * library was created to map all of node's popular template
 * engines to follow this convention, thus allowing them to
 * work seamlessly within Express.
 *
 * @param {String} ext
 * @param {Function} fn
 * @return {app} for chaining
 * @api public
 */

app.engine = function(ext, fn){
  if ('function' != typeof fn) throw new Error('callback function required');
  if ('.' != ext[0]) ext = '.' + ext;
  this.engines[ext] = fn;
  return this;
};
```

`ext`是文件后缀，`fn`是相应的模板引擎函数。如果`fn`不是函数，则报错。如果`ext`不是以点开始，则在其前面加上一个点。然后在`this.engines`中进行设置。

### 11. `app.param`

该方法是`Router.param`的代理方法，源码如下：

```javascript
/**
 * Proxy to `Router#param()` with one added api feature. The _name_ parameter
 * can be an array of names.
 *
 * See the Router#param() docs for more details.
 *
 * @param {String|Array} name
 * @param {Function} fn
 * @return {app} for chaining
 * @api public
 */

app.param = function(name, fn){
  this.lazyrouter();

  if (Array.isArray(name)) {
    name.forEach(function(key) {
      this.param(key, fn);
    }, this);
    return this;
  }

  this._router.param(name, fn);
  return this;
};
```

首先是执行`lazyrouter`。如果`name`是一个数组，则对于数组中的每一项，调用该方法。如果`name`是字符串，则调用`this._router.param`方法。

### 12. `app.path`

该方法用来获取应用的绝对路径，代码如下：

```javascript
/**
 * Return the app's absolute pathname
 * based on the parent(s) that have
 * mounted it.
 *
 * For example if the application was
 * mounted as "/admin", which itself
 * was mounted as "/blog" then the
 * return value would be "/blog/admin".
 *
 * @return {String}
 * @api private
 */

app.path = function(){
  return this.parent
    ? this.parent.path() + this.mountpath
    : '';
};
```

如果该应用是作为一个子应用，则返回父应用的路径加上挂载路径，否则返回空字符串。

### 13. `app.use`

该方法是`Router.use`的代理方法。源码如下：

```javascript
/**
 * Proxy `Router#use()` to add middleware to the app router.
 * See Router#use() documentation for details.
 *
 * If the _fn_ parameter is an express app, then it will be
 * mounted at the _route_ specified.
 *
 * @api public
 */

app.use = function use(fn) {
  var offset = 0;
  var path = '/';

  // default path to '/'
  // disambiguate app.use([fn])
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

  var fns = flatten(slice.call(arguments, offset));

  if (fns.length === 0) {
    throw new TypeError('app.use() requires middleware functions');
  }

  // setup router
  this.lazyrouter();
  var router = this._router;

  fns.forEach(function (fn) {
    // non-express app
    if (!fn || !fn.handle || !fn.set) {
      return router.use(path, fn);
    }

    debug('.use app under %s', path);
    fn.mountpath = path;
    fn.parent = this;

    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        req.__proto__ = orig.request;
        res.__proto__ = orig.response;
        next(err);
      });
    });

    // mounted an app
    fn.emit('mount', this);
  }, this);

  return this;
};
```

该方法的思路如下：

- 首先定义局部变量`offset`和`path`
    - `offset`：表示从第几个函数开始为中间件函数，默认为0
    - `path`：路径，默认为`/`
- 当第一个参数不为函数的时候
    - `while`循环中是针对第一个参数为数组的情况，即`[fn]`，`[[fn]]等情况，此时让局部变量`arg`为数组或多层数组的第一项
    - 如果`arg`不为函数，则认为第一个参数为字符串，并将其赋值给`path`，同时设置`offset`为`1`
- 获取参数中的`offset`下标开始的所有参数，并进行扁平化操作，赋值给`fns`，即代表所有的中间件函数
- 如果没有中间件函数，则抛出错误
- 对`fns`进行`forEach`操作，即对于每个中间件函数：
    - 如果该函数没有`handle`或`set`属性，则认为其是一个普通的中间件，直接调用`router.use`即可
    - 否则认为它是一个app，这里使用的是[duck typing](http://en.wikipedia.org/wiki/Duck_typing)。于是设置子应用的`mountpath`和`parent`，然后封装`fn`为`mounted_app`并调用`router.use`，然后发出`mount`事件。

### 14. `app.handle`

该方法主要是用于处理请求。首先`app`是一个函数，在`app.listen`中，可以注意到`app`事实上是`http.createServer`的回调函数，而`app`函数的内容，可以在`express.js`的`createApplication`方法中看到，就是调用了`app.handle`方法。因此，当一个请求到来的时候，就会调用`app.handle`。该方法的源码如下：

```javascript
/**
 * Dispatch a req, res pair into the application. Starts pipeline processing.
 *
 * If no _done_ callback is provided, then default error handlers will respond
 * in the event of an error bubbling through the stack.
 *
 * @api private
 */

app.handle = function(req, res, done) {
  var router = this._router;

  // final handler
  done = done || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  // no routes
  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }

  router.handle(req, res, done);
};
```

首先，在使用`http.createServer`的时候，其回调函数接收的是两个参数，即`req`和`res`。

因此，当`app`不是作为子应用的时候，参数中的`done`为`undefined`，因此需要调用`finalhandler`。而当`app`作为子应用存在时，参数`done`不为空，因此直接使用。

> `finalhandler`是一个依赖模块。具体可以参考[https://github.com/pillarjs/finalhandler](https://github.com/pillarjs/finalhandler).

当`this._router`不存在的时候，直接调用`done`，否则调用`router.handle`方法。

### 15. `app.render`

TBD
