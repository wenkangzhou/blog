跟着《Node.js 开发指南》写MicroBlog实例总结
# 跟着《Node.js 开发指南》写MicroBlog实例总结 

------

《Node.js开发指南》这本书的出版日期虽然比较新，但是作者可能写了挺久的了。Node.js毕竟还年轻，基于之上的Express、mongodb亦如此，因为年轻，因为不成熟，所以版本更新频繁，而且向下不兼容。导致很多代码在新版本的Express、Mongodb下依然不能运行。现将我看这本书，调试这本书中的microblog遇到的问题总结一下，供后人参考。

但是，对于初学者，特别是我这种菜鸟，而且是对英语完全无感的菜鸟，这本书真的很不错，推荐。

##1.       Using layouts with EJS in Express 3.x
在Express2.x中完美的支持了layout和template的概念。但是在express3.x中将这部分内容移除，取而代之，用更好理解的include来代替。

**EJS layouts in Express 2.x**
In Express 2.x my layout.ejs file looked something like this:
```html
<!DOCTYPE html>
<html>
  <head>
    <title><%= title %></title>
    <link rel='stylesheet' href='/stylesheets/style.css' />
  </head>
  <body>
    <h1>Header</h1>
    <p>Welcome to <%= title %></p>
    <%- body %>
    <p>the footer</p>
  </body>
</html>
```
an aIndex.ejs file would look like this:
```html
  <p>the content of the page<p>
```

**EJS layouts in Express 3.x**
为了达到一样的效果，我们将layout.ejs分成两部分layout_top.ejs 和layout_bottom.ejs; 然后使用include包含这两部分。
The contents of layoutTop.ejs looks like this:
```html
<!DOCTYPE html>
<html>
  <head>
    <title><%= title %></title>
    <link rel='stylesheet' href='/stylesheets/style.css' />
  </head>
  <body>
    <h1>Header</h1>
    <p>Welcome to <%= title %></p>
```
where as the contents of layoutBottom.ejs looks like this:
```html
    <p>the footer</p>
  </body>
</html>
```
and finally my index.ejs file now looks like this:
```html
<% include layoutTop %>
<p>the content of the page</p>
<% include layoutBottom %>
```
##2. 用"include" 代替"partial"
 - in Express 2.x : 
```html
<%- partial('say') %>
```
 - in Express 3.x:
```html
<% include say %>
```
##3. 将路由规则从 app.js中分离出来
 - in Express 2.x: 
```html
app.use(express.router(routes))
```
 - in Express 3.x:
```html
var routes = require('./routes');
app.use(app.router);
routes(app);
```
##4. 在express3.x中，移除了对flash的直接支持，用connect-flash这个中间件来实现它。（也可以用req.session.message来实现，详细看第6部分）
```html
var flash = require("connect-flash");
App.use(flash());
```
##5. 用 middleware  res.locals取代app.dynamicHelpers()、用 app.locals取代app.helpers() 
 - in Express 2.x:
```html
app.dynamicHelpers({
     user: function(req, res) {
     return req.session.user;
},
error: function(req, res) {
     var err = req.flash('error');
     if (err.length)
          return err;
     else
          return null;
},
success: function(req, res) {
     var succ = req.flash('success');
     if (succ.length)
          return succ;
     else
          return null;
     },
});
```
 - in Express 3.x:
请注意顺序，这里的定义要在 路由规则之前。每次执行路由规则之前这个app.use都会被调用。不相信啊，不相信，你加个log试试呗；不相信，你写在路由规则时候，看看能work不……..
```html
app.use(function(req, res, next){
  console.log("app.usr local");
  res.locals.user = req.session.user;
  res.locals.post = req.session.post;
  var error = req.flash('error');
  res.locals.error = error.length ? error : null;
  var success = req.flash('success');
  res.locals.success = success.length ? success : null;
  next();
});
app.use(app.router);
routes(app);
```
 
注意：上面的
```html
            var error = req.flash('error');
            res.locals.error = error.length ? error:null;
```
 不能改成这样：
```html
            res.locals.error = req.flash('error').length ? req.flash('error'):null;
```
这是因为：Falsh保存的变量只会在下一次请求中被访问到，所以，上面的三目运
算在赋值时，已经为空了！！！
##6.  Express 2.x migrate to 3.x page says the following: req.flash() (just use sessions: req.session.messages = ['foo'] or similar)
我试验了一下，下面的方法是work的，只是req.session的变量不像flash一样用一次就消失了，req.session的变量需要初始化。
 
 - 在 app.js 中:
```html
app.use(function(req, res, next){
  console.log("app.usr local");
  res.locals.user = req.session.user;
  res.locals.post = req.session.post;
  res.locals.error = req.session.error;
  res.locals.success = req.session.success;
  next();
});
```
 - 在 index.js 中:
对于，每个路由请求，初始化req.session.error  和req.session.success 这两个变量
```html
module.exports = function(app) {
  app.all('*', function(req, res, next) {
    req.session.error = req.session.success = null;
    next();
  });
```
 - 在监听函数中使用
```html
req.session.success = 'register succesfully.';
```
 - 或者
```html
req.session.error = 'User is not exist.';
```
##7. 关于mongodb，在require的时候加上参数express

 - in Express 2.x:

```html
var MongoStore = require('connect-mongo')
```
 - in Express 3.x:
```html
var MongoStore = require('connect-mongo')(express)
```
##8. 关于ensureIndex
 - in Express 2.x :
```html
collection.ensureIndex('name', {unique: true});
```
 - in Express 3.x:
如果还是按照2.x那么些，会报告错误滴：
```html
Error: Cannot use a writeConcern without a provided callback at Db.ensureIndex (/home/rfen/node_js_study/microblog/node_modules/mongodb/lib/mongodb/db.js:1237:11)
```
咋改？随便加个空函数呗。
```html
collection.ensureIndex('name', {unique: true}, function(err, user) {});
```
## 9. 数据库 警告， 这个警告看着烦的慌不？
 -  in Express 2.x :
```html
module.exports = new Db(setting.db, new Server(setting.host, Connection.DEFAULT_PORT));
```
 -  in Express 3.x :
```html
module.exports = new Db(setting.db, new Server(setting.host, Connection.DEFAULT_PORT), {safe:true});
```

    =================================================================
    =  Please ensure that you set the default write concern for the database by setting    =
    =   one of the options                                                                 =
    =                                                                                      =
    =     w: (value of > -1 or the string 'majority'), where < 1 means                     =
    =        no write acknowlegement                                                       =
    =     journal: true/false, wait for flush to journal before acknowlegement             =
    =     fsync: true/false, wait for flush to file system before acknowlegement           =
    =                                                                                      =
    =  For backward compatibility safe is still supported and                              =
    =   allows values of [true | false | {j:true} | {w:n, wtimeout:n} | {fsync:true}]      =
    =   the default value is false which means the driver receives does not                =
    =   return the information of the success/error of the insert/update/remove            =
    =                                                                                      =
    =   ex: new Db(new Server('localhost', 27017), {safe:false})                           =
    =                                                                                      =
    =   http://www.mongodb.org/display/DOCS/getLastError Command                           =
    =                                                                                      =
    =  The default of no acknowlegement will change in the very near future                =
    =                                                                                      =
    =  This message will disappear when the default safe is set on the driver Db           =


##10.  我还遇到了post方法无论如何都不执行的问题，也就是表单提交了没任何反应。折腾了许久，最后发现，是在 ejs中，把method写成了methoed，node.js的调试还是很折磨人的。
##11.       最后，看一下express官方对从2.x到3.x的简介  https://github.com/visionmedia/express/wiki/Migrating-from-2.x-to-3.x

> 文章转载自http://blog.sina.com.cn/s/blog_6591eb240101cmwm.html
