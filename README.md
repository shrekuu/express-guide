> 在页面上阅读小字版：[http://sofish.github.com/express-guide/](http://sofish.github.com/express-guide/)
>
> 在页面上阅读大字版：[http://shrekuu.github.com/express-guide/](http://shrekuu.github.com/express-guide/)
>
> 大字版也换上了更舒服的字体, 保护眼睛. 英文为 monaco 字体, 中文 Hiragino Sans GB.  💁 🙅 🙆 🙋


# Express 指南

这是对 [express.js guide](http://expressjs.com/guide.html) 的一个翻译。翻译人：[Sofish Lin](http://sofish.de)（[twitter](http://twitter.com/sofish)）。

翻译此文档的目的是学习 Express，因为翻译意味着一种责任，因此边看 Express 源代码边翻。在容易混淆的地方都加上了译注。给自己做一个记录保存下来，也给有需要的人一份参考。当然，由于个人水平的原因，并不能最恰当地传达作者的幽默，也少不了错误，还请各位有看到的同学多多指正。

如有任何问题，请直接给我提 issue、pull request 或者在 twitter at-at 我。

----------------------------------------------------------

### 安装

```bash
$ npm install express
```

或者在任何地方使用可执行的 `express(1)` 安装： 

```bash
\# 译注：强烈建议这种方式
$ npm install -g express
```

## 快速上手

最快上手 express 的方法是利用可执行的 `express(1)` 来生成一个应用，如下所示：

创建一个 app：

```bash
$ npm install -g express
$ express /tmp/foo && cd /tmp/foo
```

安装依赖包：

```bash
$ npm install -d
```

启动服务器:

```bash
$ node app.js
```

### 创建一个服务器

要创建一个 `express.HTTPServer` 实例，只需调用 `createServer()` 方法。 通用这个应用实例，我们可以定义基于 HTTP 动作（HTTP Verbs）的路由，以 `app.get()` 为例：

```js
var app = require('express').createServer();

app.get('/', function(req, res){
  res.send('hello world');
});

app.listen(3000);
```

### 创建一个 HTTPS 服务器

如上述初始化一个 `express.HTTPSServer` 实例。然后我们给它传一个配置对象，接受 `key`、`cert` 和其他在 [https 文档](http://nodejs.org/docs/v0.3.7/api/https.html#https.createServer) 所提到的（属性/方法）。

```js
 var app = require('express').createServer({ key: ... });
```

### 配置

Express 支持任意环境，如产品阶段（production）和开发阶段（development）。开发者可以使用 `configure()` 方法来设置当前所需环境。如果 `configure()` 的调用不包含任何环境名，它将运行于所有环境中所指定的回调。

<mark>译注：</mark> 像 _production_ / _development_ / _stage_ 这些别名都是可以自已取的，如 [application.js](https://github.com/visionmedia/express/blob/master/lib/application.js) 中的 `app.configure` 所示。实际用法看下面例子。

下面这个例子仅在开发阶段 `dumpExceptions` (抛错)，并返回堆栈异常。不过在两个环境中我们都使用 `methodOverride ` 和 `bodyParser`。注意一下 `app.router` 的使用，它可以（可选）用来加载（mount）程序的路由，另外首次调用 `app.get()`、`app.post()` 等也将会加载路由。

```js
app.configure(function(){
	app.use(express.methodOverride());
	app.use(express.bodyParser());
	app.use(app.router);
});

app.configure('development', function(){
	app.use(express.static(__dirname + '/public'));
	app.use(express.errorHandler({ dumpExceptions: true, showStack: true }));
});

app.configure('production', function(){
  var oneYear = 31557600000;
  app.use(express.static(__dirname + '/public', { maxAge: oneYear }));
  app.use(express.errorHandler());
});
```

对于相似的环境你可以传递多个环境字符串：

```js
app.configure('stage', 'prod', function(){
  // config
});
```

对于任何内部设置（[#](https://twitter.com/sofish/status/219430499725217794))，Express 提供了 `set(key[, val])`、 `enable(key)` 和 `disable(key)` 方法:

<mark>译注：</mark>设置详见：[application.js](https://github.com/visionmedia/express/blob/master/lib/application.js) 的 `app.set`。

```js
 app.configure(function(){
    app.set('views', __dirname + '/views');
    app.set('views');
    // => "/absolute/path/to/views"
    
    app.enable('some feature');
    // 等价于：app.set('some feature', true);
    
    app.disable('some feature');
    // 等价于：app.set('some feature', false);
 
    app.enabled('some feature')
    // => false
 });
```
变更环境我们可以设置 `NODE_ENV` 环境变量，如：

```bash
$ NODE_ENV=production node app.js
```

这非常重要，因为多数缓存机制只在产品阶段是被打开的。


### 设置

Express 支持下列快捷（out of the box）设置:

* `basepath` 用于 `res.redirect()` 的应用程序基本路径（base path），显式地处理绑定的应用程序（transparently handling mounted apps.）
* `view` View 默认的根目录为 **CWD/views**
* `view engine` 默认 View 引擎处理（View 文件）并不需要使用后缀
* `view cache` 启用 View 缓存 (在产品阶段被启用)
* `charet` 改变编码，默认为 utf-8
* `case sensitive routes` 路由中区分大小写
* `strit routing` 启用后（路由中的）结尾 `/` 将不会被忽略（译注：即 `app.get('/sofish')` 和 `app.get('/sofish/')` 将是不一样的）
* `json callback` 启用 `res.send()` / `res.json()` 显式的 jsonp 支持（transparent jsonp support）

### 路由

Express 利用 HTTP 动作提供一套提示性强、有表现力的路由 API。打个比方，如果想要处理某个路径为 _/user/12_ 的账号，我们能像下面这样来定义路由。关联到命名占位符（named placeholders）的值可用 `req.params` 来访问。

```js
app.get('/user/:id', function(req, res){
    res.send('user ' + req.params.id);
});
```

路由是一个在内部被编译为正则的字符串。譬如，当 _/user/:id_ 被编译，一个简化版本的正则表达弄大概如下：

```js
// 修改一下官方的这个字符串
/\/user\/([^\/]+)\/?/
```

正则表达式可以传入应用于复杂的场景。由于通过字面量正则表达式捕获的内容组是匿名的，我们可能直接通过 `req.params` 来访问它们。因此，我们捕获的第一组内容将是 `req.params[0]`，同时第二组是紧接着的 `req.params[1]`。 

```js
app.get(/^\/users?(?:\/(\d+)(?:\.\.(\d+))?)?/, function(req, res){
    res.send(req.params);
});
```

Curl 针对上述定义路由的请求：

```bash
$ curl http://dev:3000/user
[null,null]
$ curl http://dev:3000/users
[null,null]
$ curl http://dev:3000/users/1
["1",null]
$ curl http://dev:3000/users/1..15
["1","15"]
```
下面是一些路由的实例，关联到他们可能使用到的路径：

```bash
"/user/:id"
/user/12

"/users/:id?"
/users/5
/users

"/files/*"
/files/jquery.js
/files/javascripts/jquery.js

"/file/*.*"
/files/jquery.js
/files/javascripts/jquery.js

"/user/:id/:operation?"
/user/1
/user/1/edit

"/products.:format"
/products.json
/products.xml

"/products.:format?"
/products.json
/products.xml
/products

"/user/:id.:format?"
/user/12
/user/12.json
```

举个例子，我们可以使用 `POST` 发送 json 数据，通过 `bodyParser` 这个可以解析 json 请求内容（或者其他内容）的中间件来返回数据，并将返回结果存于 `req.body` 中：

```js
var express = require('express')
  , app = express.createServer();

app.use(express.bodyParser());

app.post('/', function(req, res){
  res.send(req.body);
});

app.listen(3000);
```

通常我们可以使用一个像 `user/:id` 这样，没有（命名）限制的“傻瓜”式的占位符。然而比方说，我们要限制用户 id 只能是数字，那么我们可能使用 `/user/:id([0-9]+)`，这个将仅当占位符是包含至少一位数字时才生效（适配，match）。

### 进路控制（Passing Route Control）

我们可以通过调用第三个参数，`next()` 函数，来控制下一个适配的路由。如果找不到适配，控制权将会传回给 Connect，同时中间件将会按在 `use()` 中添加的顺序被依次调用。道理同样适应于多个定义到同一路径的路由，他们将会依次被调用直到其中某个不调用 `next()` 而决定做出请求响应。

```js
app.get('/users/:id?', function(req, res, next){
	var id = req.params.id;
	if (id) {
		// do something
	} else {
		next();
	}
});

app.get('/users', function(req, res){
	// do something else
});
```

`app.all()` 方法只调用一次就可以方便地把同样的逻辑到所有 HTTP 动作。下面我们使用它来从伪数据中提取一个用户，将其赋给 `req.user`。

```js
var express = require('express')
  , app = express.createServer();

var users = [{ name: 'tj' }];

app.all('/user/:id/:op?', function(req, res, next){
  req.user = users[req.params.id];
  if (req.user) {
    next();
  } else {
    next(new Error('cannot find user ' + req.params.id));
  }
});

app.get('/user/:id', function(req, res){
  res.send('viewing ' + req.user.name);
});

app.get('/user/:id/edit', function(req, res){
  res.send('editing ' + req.user.name);
});

app.put('/user/:id', function(req, res){
  res.send('updating ' + req.user.name);
});

app.get('*', function(req, res){
  res.send(404, 'what???');
});

app.listen(3000); 
```

### 中间件

使用的 [Connect](http://github.com/senchalabs/connect) 中间件（属性）通常伴随着你的一个常规 Connect 服务器，被传到 `express.createServer()` 。如：

```js
var express = require('express');

var app = express.createServer(
  	  express.logger()
  	, express.bodyParser()
  );
```

另外，在 `configure()` 块内 —— 这个渐进式的宫殿（译注：笑^^，in a progressive manner），我们还可以方便地使用 `use()` 来添加中间件。

```js
app.use(express.logger({ format: ':method :url' }));
```

通常，使用 connect 中间件你可能会用到 `require('connect')`，像这样：

```js
var connect = require('connect');
app.use(connect.logger());
app.use(connect.bodyParser());
```

这在某种程度上来说有点不爽，所以 express 重导出（re-exports）了这些中间件属性，尽管他们是一样的:

```js
app.use(express.logger());
app.use(express.bodyParser());
```

中间件的顺序非常重要，当 Connect 收到一个请求，我们传到 `createServer()` 或者 `use()` 执行的第一个中间件将附带三个参数，request、response，以及一个回调函数（通常是 `next`）。当 `next()` 被调用，将轮到第二个中间件，依此类推。之所以说这是值得注意的，是因为很多中间件彼此依赖，例如 `methodOverride()` 查询 `req.body` 方法来检测 HTTP 方法重载，另一方面 `bodyParser()` 解析请求内容并将其于寄存于 `req.body`。另一个例子是 cookie 解析和 session 支持，我们必须先 `use()` `cookieParser()` 紧接着 `session()`。

很多 Express 应用都包含这样的一行 `app.use(app.router)`，这看起来可能有点奇怪，其实它仅仅是一个包含所有定义路由规则，并执行基于现有 URL 请求和 HTTP 方法路由查找的一个中间件功能。Express 允许你决定其位置（to position），不过默认情况下它被放置于底部。通过改变路由的位置，我们可以改变中间件的优先级，譬如我们想把错误报告做为最后的中间件，以便任何传给 `next()` 的异常都可以通过它来处理；又或者我们希望静态文件服务优先级更低，以允许我们的路由可以监听单个静态文件请求的下载次数，等等。这看起来差不多是这样的：

```js
app.use(express.logger(...));
app.use(express.bodyParser(...));
app.use(express.cookieParser(...));
app.use(express.session(...));
app.use(app.router);
app.use(express.static(...));
app.use(express.errorHandler(...));
```

首先我们添加 `logger()`，它可能包含 node 的 `req.end()` 方法，提供我们响应时间的数据。接下来请求的内容将会被解析（如果有数据的话），紧接着的是 cookie 解析和 session 支持，同时 `req.session` 将会在触发 `app.router` 中的路由时被定义，这时我们并不调用 `next()`，因此 `static()` 中间件将不会知道这个请求，如若已经定义了如下一个路由，我们则可以记录各种状态、拒绝下载和消耗下载点数等。

```js
var downloads = {};

app.use(app.router);
app.use(express.static(__dirname + '/public'));

app.get('/*', function(req, res, next){
  var file = req.params[0];
  downloads[file] = downloads[file] || 0;
  downloads[file]++;
  next();
});
```

### 路由中间件

路由可以利用路由器中间件，传递一个以上的回调函数（或者数组）到其方法中。这个特性非常有利于限制访问、通过路由下载数据，等等。

通常异步数据检索看起来可能像下例，我们使用 `:id` 参数，尝试加载一个用户： 

```js
app.get('/user/:id', function(req, res, next){
  loadUser(req.params.id, function(err, user){
    if (err) return next(err);
    res.send('Viewing user ' + user.name);
  });
});
```

为保证 DRY 原则和提升可读，我们可以把这个逻辑应用于一个中间件内。如下所示，抽象这个逻辑到中间件内将允许你重用它，同时保证了我们路由的简洁。

```js
function loadUser(req, res, next) {
  // You would fetch your user from the db
  var user = users[req.params.id];
  if (user) {
    req.user = user;
    next();
  } else {
    next(new Error('Failed to load user ' + req.params.id));
  }
}

app.get('/user/:id', loadUser, function(req, res){
  res.send('Viewing user ' + req.user.name);
});
```

多重路由可以，并按顺序应用到更深一层的逻辑，如限制一个用户账号的访问。下面的例子只允许通过鉴定的用户才可以编辑他（她）的账号。

```js
function andRestrictToSelf(req, res, next) {
  req.authenticatedUser.id == req.user.id
    ? next()
    : next(new Error('Unauthorized'));
}

app.get('/user/:id/edit', loadUser, andRestrictToSelf, function(req, res){
  res.send('Editing user ' + req.user.name);
});
```

时刻铭记路由只是简单的函数，如下所示，我们可以定义返回中间件的函数以创建一个更具表现力，更灵活的方案。

```js
function andRestrictTo(role) {
  return function(req, res, next) {
    req.authenticatedUser.role == role
      ? next()
      : next(new Error('Unauthorized'));
  }
}

app.del('/user/:id', loadUser, andRestrictTo('admin'), function(req, res){
  res.send('Deleted user ' + req.user.name);
});
```

常用的中间件“堆栈”可以通过一个数组来传递（会被递归应用），这些中间件可以混着、匹配到任何层次（which can be mixed and matched to any degree）。

```js
var a = [middleware1, middleware2]
  , b = [middleware3, middleware4]
  , all = [a, b];

app.get('/foo', a, function(){});
app.get('/bar', a, function(){});

app.get('/', a, middleware3, middleware4, function(){});
app.get('/', a, b, function(){});
app.get('/', all, function(){});
```

对于这个实例的完整代码，请看 [route middleware example](http://github.com/visionmedia/express/blob/master/examples/route-middleware/app.js) 这个仓库。

我们可能会有多次想要“跳过”剩余的路由中间件，继续匹配后续的路由。做到这点，我们只需调用 `next()` 时带上 `'route'` 字符串 —— `next('route')`。如果没有余下的路由匹配到请求的 URL，Express 将会返回 `404 Not Found`。


### HTTP 方法

至此已接触了好几次 `app.get()`，除此这外 Express 还提供了其他常见的 HTTP 动作，如 `app.post()` 、`app.del()` 等等。

POST 用法的一个常用例子是提交一个表单。下面我们简单地在 html 中把表单的 method 属性设置为 post，控制权将会指派给它下面所定义的路由。

```html
<form method="post" action="/">
     <input type="text" name="user[name]" />
     <input type="text" name="user[email]" />
     <input type="submit" value="Submit" />
</form>
```

默认上 Express 并不知道如何处理这个请求的内容，因此我们必须添加 `bodyParser` 中间件，它将解析 `application/x-www-form-urlencoded` 和 `application/json` 请求的内容，并把变量存放于 `req.body` 中。我们可以像下述示例一样来使用这个中间件：

```js
app.use(express.bodyParser());
```

如下，我们的路由将有权访问 `req.body.user` 对象，当有 name 和 email 被定义时它将包含这两个属性（译注：如果表单发送的内容不为空的话）。

```js
app.post('/', function(req, res){
  console.log(req.body.user);
  res.redirect('back');
});
```

当想在一个表单中使用像 PUT 这样的方法，我们可以使用一个命名为 `_method` 的 hidden `input`，它可以用以修改 HTTP 方法。为了做这个，我们首先需要 `methodOverride` 中间件，它必须出现于 `bodyParser` 后面，以便使用它的 `req.body`中所包含的表单值。

```js
app.use(express.bodyParser());
app.use(express.methodOverride());
```

对于这些方法为何不是默认拥有，简单来说只是因为它并不是 Express 所要求完整功能所必须。方法的使用依赖于你的应用，你可能并不需要它们，客户端依然能使用像 PUT 和 DELETE 这样的方法，你可以直接使用它们，因为 `methodOverride` 为 form 提供了一个非常不错的解决方案。下面将示范如何使用 PUT 这个方法，看起来可能像：

```html
<form method="post" action="/">
  <input type="hidden" name="_method" value="put" />
  <input type="text" name="user[name]" />
  <input type="text" name="user[email]" />
  <input type="submit" value="Submit" />    
</form>

app.put('/', function(){
    console.log(req.body.user);
    res.redirect('back');
});
```

### 错误处理

Express 提供了 `app.error()` 方法以便接收到的异常在一个路由里抛出，或者传到 `next(err)` 中。下面这个例子将基于特定的 NotFound 异常处理不同的页面：

```js
function NotFound(msg){
  this.name = 'NotFound';
  Error.call(this, msg);
  Error.captureStackTrace(this, arguments.callee);
}

NotFound.prototype.__proto__ = Error.prototype;

app.get('/404', function(req, res){
  throw new NotFound;
});

app.get('/500', function(req, res){
  throw new Error('keyboard cat!');
});
```

如下述，我们可以多次调用 `app.error()`。这里我们检测 NotFound 的实例，并显示 404 页面，或者传到 `next` 错误处理器。值得注意的是这些处理器可以在任何地方定义，因为他们将会在 `listen()` 的时候被放置于路由处理器下面。它允许在 `configure()` 块内有定义，以便我们能基于环境用不同的异常处理方式。

```js
app.error(function(err, req, res, next){
    if (err instanceof NotFound) {
        res.render('404.jade');
    } else {
        next(err);
    }
});
```

为求简洁（for the simplicity），这里我们假定这个 demo 的所有错误为 500，当然你可以可以选择自己喜欢的。像 node 执行文件系统的系统调用时，你可能会接收到一个带有 ENOENT 的 `error.code`，意思为 “不存在这样的文件或目录” 的错误，我们可以在错误处理器中使用，或者当有需要时可显示一个指定的页面。

```js
app.error(function(err, req, res){
  res.render('500.jade', {
     error: err
  });
});
```

我们的 app 同样可以利用 Connect 的 `errorHandler` 中间件来汇报异常。譬如当我们希望在 “开发” 环境输出 `stderr` 异常时，我们可以使用：

```js
app.use(express.errorHandler({ dumpExceptions: true }));
```

同时在开发阶段我们可能需要在花哨的 HTML 页面显示我们传递和抛出的异常，对此我们可以把 `showStack` 设置为 `true`。

```js
app.use(express.errorHandler({ showStack: true, dumpExceptions: true }));
```

`errorHandler` 中间件还可以在 `Accept: application/json` 存在的时候返回 json，这对于开发重度依赖客户端 Javascript 的应用非常有用。

### Route 参数预处理

路由参数预处理，通过隐式数据加载和请求验证，可以大大提升你程序的可读性。打个比方，你通常需要持续地从多个路由获取基本数据。像用 `/user/:id` 加载一个用户，通常来说我们可能会这样干：

```js
app.get('/user/:userId', function(req, res, next){
  User.get(req.params.userId, function(err, user){
    if (err) return next(err);
    res.send('user ' + user.name);
  });
}); 
```

通过预处理，我们的参数可以映射到执行验证、控制(coercion)，甚至从数据库加载数据的回调。如下我们带着参数名调用 `app.param()` 希望将其映射于某些中间件。如你所见，我们接受代表占位符值的 `id` 参数。使用这个，我们如常加载用户并处理错误，以及简单地调用 `next()` 来把控制权交由下一个预处理或者路由处理器。

```js
app.param('userId', function(req, res, next, id){
  User.get(id, function(err, user){
    if (err) return next(err);
    if (!user) return next(new Error('failed to find user'));
    req.user = user;
    next();
  });
});
```

一旦这样做，上所述将会大大地提升路由的可读性，并且允许我们轻松地在整个程序中共享逻辑：

```js
app.get('/user/:userId', function(req, res){
  res.send('user ' + req.user.name);
});
```

### View 处理

View 文件件使用 &lt;name&gt;.&lt;engine&gt; 这样的格式，其中 &lt;engine&gt; 是被 `require` 进来模块的名。例如 `layout.ejs` 将告诉 view 系统去 `require('ejs')`，被加载的模块必须(导出) `exports.compile(str, options)` 方法，并返回一个 Function 来适应 Express。`app.register()` 可用以改变这种默认行为，将文件扩展名映射到特定的引擎。譬如 “foo.html” 可以由 ejs 来处理。

下面这个例子使用 [Jade](http://github.com/visionmedia/jade) 来处理 index.html。因为我们并未使用 `layout: false`，index.jade 处理后的内容将会被传入到 layout.jade 中一个名为 body 的本地变量。

```js
app.get('/', function(req, res){
	res.render('index.jade', { title: 'My Site' });
});
```

新的 `view engine` 设置允许我们指定默认的模板引擎，例如当我们使用 jade 时可以这样设置：

```js
app.set('view engine', 'jade');
```

允许我们这样处理：

```
res.render('index');
```

对应于：

```js
res.render('index.jade');
```

当 `view engine` 被设定，扩展名实属可选，但我们依然可以混着匹配模板引擎：

```js
res.render('another-page.ejs');
```

Express 同时还提供了 `view options` 设置，这将应用于一个 view 每次被渲染的时候，譬如你不希望使用 layouts  的时候可能会这样做：

```js
app.set('view options', {
	layout: false
});
```

在需要的时候，这可以在 `res.render()` 调用的内部进行重载：

```js
res.render('myview.ejs', { layout: true });
```

当有需要变更一个 layout，我们通常需要再指定一个路径。譬如当我们已经把 `view engine` 设置为 jade，并且这个文件命名为 ./views/mylayout.jade，我们可以这样简单地进行传参：

```js
res.render('page', { layout: 'mylayout' });
```

否则（译注：没有把 `view engine` 设置为 jade 或者其他的引擎时），我们必须指定一个扩展名：

```js
res.render('page', { layout: 'mylayout.jade' });
```

它们同样可以是绝对路径：

```js
res.render('page', { layout: __dirname + '/../../mylayout.jade' });
```

对于这点有一个不错的例子 —— 自定义 ejs 的起始和闭合标签：

```js
app.set('view options', {
	open: '{{',
	close: '}}'
})
```

### View 部件

Express 的 view 系统内置了部件（partials） 和集合器（collections）的支持，相当于用一个 “迷你” 的 view 替换一个文档碎片（document fragment）。示例，在一个 view 中重复渲染来显示评论，我们可以使用部件集：

```js
partial('comment', { collection: comments });
```

如果并不需要其他选项或者本地变量，我们可以省略整个对象，简单地传进一个数组，这与上述是等价的：

```js
partial('comment', comments);
```

在使用中，部件集无偿地提供了一些 “神奇” 本地变量的支持：

* _firstInCollection_  `true`，当它是第一个对象的时候
* _indexInCollection_  在集合器对象中的索引
* _lastInCollection_  `true`，当它是最后一个对象的时候
* _collectionLength_  集合器对象的长度

本地变量的传递（生成）具备更高的优先级，同时，传到父级 view 的本地变量对于子级 view 同样适应。例如当我们用 `partial('blog/post', post)` 来渲染一个博客文章，它将会生成一个 `post` 本地变量，在调用这个函数的 view 中存在本地变量 `user`，它将同样对 `blog/post` 有效。（译注：这里 partial 比较像 php 中的 include 方法）。

__注意:__ 请谨慎使用部件集合器，渲染一个长度为 100 的部件集合数组相当于我们需要处理 100 个 view。对于简单的集合，最好重复内置，而非使用部件集合器以避免开销过大。

### View 查找

View 查找相对于父级 view （路径）执行，如我们有一个 view 页面叫作 views/user/list.jade，并且在其内部写有 `partial('edit')` 则它会尝试加载 views/user/edit.jade，同理 `partial('../messages')` 将会加载 views/messages.jade。

View 系统还支持模板索引，允许你使用一个与 view 同名的目录。例如在一个路由中，`res.render('users')` 得到的非 views/users.jade 即 views/users/index.jade。（译注：先处理 &lt;path&gt;.&lt;engine&gt; 的情况，再处理 &lt;path&gt;/&lt;index.&lt;engine&gt; 的情况，详情可见 [view.js](https://github.com/visionmedia/express/blob/master/lib/view.js)。）

当使用上述 view 索引，我们在与 view 同一个目录下，使用 `partial('users')` 中引用 views/users/index.jade，与此同时 view 系统会尝试索引 `../users/index`，而无须我们调用 `partial('users')`。

### Template Engines

下列为 Express 最常用的模板引擎:

* [Haml](http://github.com/visionmedia/haml.js) haml 实现
* [Jade](http://jade-lang.com) haml.js 继位者
* [EJS](http://github.com/visionmedia/ejs) 嵌入式 JavaScript
* [CoffeeKup](http://github.com/mauricemach/coffeekup) 基于 CoffeeScript 的模板
* [jQuery Templates](https://github.com/kof/node-jqtpl) 

### Session 支持

Session 支持可以通过使用 Connect 的 session 中间件来获得，为此通常我们同时需要在其前加上 `cookieParser` 中间件，它将解析和存储 cookie 数据于 `req.cookies` 中。

```js
app.use(express.cookieParser());
app.use(express.session({ secret: "keyboard cat" }));
```

默认情况下 session 中间件使用 Connect 内置的内存存储，然而还有其他多种实现方式。如 [connect-redis](http://github.com/visionmedia/connect-redis) 提供了一种 [Redis](http://code.google.com/p/redis/) 的 session 存储，它这可像下面这样被使用：

```js
var RedisStore = require('connect-redis')(express);
app.use(express.cookieParser());
app.use(express.session({ secret: "keyboard cat", store: new RedisStore }));
```

至此，`req.session` 和 `req.sessionStore` 属性将可以被所有路由和后继的中间件使用。在 `req.session` 上的所有属性都会在一个响应中被自动保存下来，譬如当我们想要添加数据到购物车： 

```js
var RedisStore = require('connect-redis')(express);
app.use(express.bodyParser());
app.use(express.cookieParser());
app.use(express.session({ secret: "keyboard cat", store: new RedisStore }));

app.post('/add-to-cart', function(req, res){
  // 我们可能通过一个表单 POST 出多个 item
  // (在些使用 bodyParser() 中间件)
  var items = req.body.items;
  req.session.items = items;
  res.redirect('back');
});

app.get('/add-to-cart', function(req, res){
  // 当返回时，页面 GET /add-to-cart
  // 我们可以检查 req.session.items && req.session.items.length
  // 来打印出提示
  if (req.session.items && req.session.items.length) {
    req.notify('info', 'You have %s items in your cart', req.session.items.length);
  }
  res.render('shopping-cart');
});
```

对于 `req.session` 对旬，它还有像 `Session#touch()`、`Session#destroy()`、 `Session#regenerate()` 等用以维护和操作 session 的方法。更多的详情请看 [Connect Session](http://senchalabs.github.com/connect/middleware-session.html) 的文档。

### 升级指南

对于使用 Express 1.x 的同学，如果你有很重要的程序需要升级到 2.x 以获得更好的支持，请看官方非常详细的 [迁移指南](http://expressjs.com/guide.html#migration-guide)

----------------------------------------------------------

最后，感谢一下自己，又长进了。读到这里的你，如果感觉它对你有帮助，请同时分享给你有需要的朋友吧～
