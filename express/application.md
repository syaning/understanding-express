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

### 2. app及init

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

### 3. app.listen

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

### 4. settings相关

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
