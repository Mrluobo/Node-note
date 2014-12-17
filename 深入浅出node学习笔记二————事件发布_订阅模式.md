# 事件发布/订阅模式

标签（空格分隔）： node javascript

---

###node自身提供的events模块具有的基本方法：
```addListener/on()```、```once()```、```removeListener()```、```removeAllListeners()```和```emit()``` 语法类似于前端javascript中的自定义事件。

eg:
```javascript
//订阅事件
emitter.on("event1",function(message){
    console.log(message);
});
//发布事件
emitter.emit("event1","I am message!");
```

###事件发布/订阅模式的用途

 1. 异步编程。伴随着事件循环，异步出发。
 2. 解耦业务逻辑。

####若对一个事件添加超过10个侦听器，会得到一条警告。可以通过：
```javascript
emitter.setMaxListeners(0);
```
来去除限制。

###集成events模块
以下为Node中Stream对象继承EventEmitter的示例：
```javascript
var events = require("events");
function Stream(){
    events.EventEmitter.call(this);
}
util.inherits(Stream,events.EventEmitter);
```
util模块中封装了继承的方法。

###利用事件队列解决雪崩问题
**雪崩问题**：在高访问量、大并发量的情况下缓存失效的情景。此时大量的请求同时涌入数据库，数据库无法同时承受如此大的查询量，影响整体响应速度。
示例：
```javascript
var proxy = new events.EventEmitter();
var status = "ready";
var select = function(callback){
    proxy.once("selected",callback);
    if(status == "ready"){
        db.select("SQL",function(results){
            proxy.emit("selected",results);
            status = "ready";
        })
    }
}
```
利用once()将所有callback压入事件队列，且由于once()的特性，每个回调只会被执行依次。对于相同的请求，在同一时间只有第一次的请求会进行查询，后到的请求只需要等待数据即可。一旦查询结束，结果可以被这次请求共同使用。

###多异步之间的协作方案
很多业务逻辑可能依赖多个回调或者事件传递，结果导致嵌套过深。
**解决方案**：
eg:在这里需要读取模板、数据读取、本地化资源读取之后再进行渲染模板。

```javascipt
var count = 0;
var result = ();
var done = function(key,value){
    results[key] = value;
    count++;
    if(count === 3){
        render(results);//渲染页面
    }
}
fs.readFlile(templath_path,"utf-8",function(err,template){
    done("template",template);
});

db.query(sql,function(err,data){
    done("data",data);
});

l10n.get(function(err,resources){
    done("resources",resource);
})
```
同时可以利用偏函数来处理哨兵变量（count）和第三方函数（done）的关系。
```javascript
var after = function(time,callback){
    var count = 0,results = {};
    return function(){
        count++;
        if(count == time){
            callback(results);
        }
    }
}
var done = after(3,render);
```
上面的方法完成了多对一的目的，可以利用发布/订阅模式完成多对多的方案。
```javascript
var emitter = new events.Emitter();
var done = fater(time,render);

emitter.on("done",done);
emitter.on("done",other);


fs.readFlile(templath_path,"utf-8",function(err,template){
    emitter.emit("done","template",template);//第一个参数为事件名，其余为回调参数。
});

db.query(sql,function(err,data){
    emitter.emit("done","data",data);
});

l10n.get(function(err,resources){
    emitter.emit("done","resources",resources);
})
```

###EventProxy模块
```javascript
var proxy = new EventProxy();
``` 
方法：
```javascript
proxy.all("event1","event2","event3",function("各个参数"){
//todo
});
```
.all()方法用来订阅多个事件，当多个事件都被出发后，侦听器执行。只执行一次。
.tail()方法同all一样，不同的是，若订阅的事件都被触发，执行完毕一次之后，事件中的某个再次触发，再次执行侦听器。
```javascript
proxy.after("data",10,function(){
//todo
})
```
在data被触发十次之后执行侦听器。

###EventPorxy异常处理
**之前的处理方式**：
```javascript
var ep = new EventPorxy();
ep.bind("error",function(err){
//卸载处理函数
ep.unbind();
callback(err);
//todo
})
//然后在每次调用时进行判断
//eg:
fs.readFile("template.tpl","utf-8",function(err,content){
    if(err){
        return ep.emit("error",err);
    }
    ep.emit("template",content);
})
```
当逻辑复杂时会有多个处理函数，代码量增多。
EventProxy提供了.fail()和.done()两个方法处理异常。使用方法。
```javascript
var ep = new EventProxy();
ep.all(...);
ep.fail(callback);//订阅错误事件。
fs.readFile("template.tpl","utf-8",ep.done("tpl"));
```
其中：
```javascript
ep.fail(callback);
```
等同于
```javascript
ep.bind("error",function(err){
    ep.unbing();
    callback(err);
})
```
而：
```javascript
ep.done("tpl");
```
相当于：
```javascript
function(err,content){
    if(err){
        return ep.emit("error",err);
    }
    ep.emit("tpl",content);
}
```