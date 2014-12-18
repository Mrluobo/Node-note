##Promise/Deferred模式
###Promise/Deferred模式包含两部分：
Promise状态：未完成态、完成态、失败态。
**方向**：从未完成态转换为完成态或失败态，方向不可逆。
Promise对象需要具有`.then()`方法即可。
`.then()`示例：

    .then(fulfilledHandler,errorHandler,progressHandler);

Promise实现示例：
```javascript
var Promise = function(){
	EventEmitter.call(this);
}
util.inherits(Promise,EventEmitter);
//通过继承events模块来实现。

Promise.prototype.then = function(fulfilledHandler,errorHandler,progressHandler){
	if(typeof fulfilledHandler == "function"){
		this.once("success",fulfilledHandler);
	}
	if(typeof errorHandler == "function"){
		this.once("error",errorHandler);
	}
	if(typeof progressHandler == "function"){
		this.once("progress",progressHandler);
	}
	return this;
}
```
then方法将回调函数存储起来。
Deferred对象来触发回调:
```
var Deferred = function(){
	this.state = "unfulfilled";
	this.promise = new Promise();
}
Deferred.prototype.resolve = function(obj){
	this.state = "fulfilled";
	this.promise.emit("success",obj);
}
Deferred.prototype.reject = function(err){
	this.state = "failed";
	this.promise.emit("error",err);
}
Deferred.prototpye.progress = function(data){
	this.promise.emit("progress",data);
}
```
实例响应代码：
```javascript
res.setEncoding("utf-8");
res.on("data",function(chunk){
	console.log(chunk);
});
res.on("end",function(){
	//dosomething
});
res.on("error",function(err){
	//deal with error
});
```
可转换为：
```javascript
res.then(function(){
	//dosomething
},function(err){
	//deal with error
},function(chunk){
	console.log(chunk);
});
```
需要进行的代码改造:
```javascript
var promisify = function(res){
	var deferred = new Deferred();
	var result = "";
	res.on("data",function(chunk){
		deferred.progress(chunk);
		result += chunk;
	});
	res.on("error",function(err){
		deferred.reject(err);
	});
	res.on("end",function(){
		deferred.resolve(result);
	})
	return deferred.promise;
}
```
最终调用方法:
```javascript
promisify(res).then(function(){
	//dosomething
},function(err){
	//deal with error
},function(chunk){
	console.log(chunk);
});
```
###Q模块
实例：
```javascript
var readFile = function(file,encoding){
	var deferred = Q.defer();
	fs.readFile(file,encoding,deferred.makeNodesolver());
	return deferred.promise;
}
调用方法
readFile("foo.txt","utf-8").then(function(){
	//do something
},function(err){
	//deal with error
})
```


###多异步协作

类似于EventProxy的实现，只有当所有异步都成功之后，整个异步操作才成功。
```javascript
Deferred.prototype.all = function(promises){
	var count = promises.length;
	var that = this;
	var result = [];
	promises.forEach(function(promise,i){
		promise.then(function(data){
			count--;
			result[i] = data;
			if(count == 0){
				that.resolve(result);
			}
		},function(err){
			that.reject(err);
		})
	})
	return this.promise;
};
```
调用方法:
```javascript
var promise1 = readFile("test1.txt","utf-8");
var promise2 = readFile("test2.txt","utf-8");
var deferred = new Deferred();
deferred.all([promise1,promise2]).then(function(result){
	//dosomething
},function(err){
	//deal with error
});
```
支持序列执行的Promise
```javascript
var Promise = function(){
	this.queue = [];
	this.isPromise = true;
}
Promise.prototype.then = function(fulfilledHandler,errorHandler,progressHandler){
	var handler = {};
	if(typeof fulfilledHandler == "function"){
		handler.fulfilled = fulfilledHandler;
	}
	if(typeof errorHandler == "function"){
		handler.error = errorHandler;
	}
	this.queue.push(handler);
	return this;
}
var Deferred = function(){
	this.promise = new Promise();
}
Deferred.prototype.callback = function(){
	var that = this;
	return function(err,file){
		if(err){
			return that.reject(err);
		}
		that.resolve(file);
	}
}
Deferred.prototype.resolve = function(obj){
	var handler;
	var promise = this.promise;
	while((handler = promise.queue.shift())){

		console.log(promise.name);

		if(handler && handler.fulfilled){
			var ret = handler.fulfilled(obj);
			if(ret && ret.isPromise){
				ret.queue = promise.queue;
				this.promise = ret;
				return;
			}
		}
	}
}
var fs = require("fs");
var readFile1 = function(file,encoding){
	var deferred = new Deferred();
	fs.readFile(file,encoding,deferred.callback());
	deferred.promise.name = 1;
	return deferred.promise;
}
var readFile2 = function(file,encoding){
	var deferred = new Deferred();
	fs.readFile(file,encoding,deferred.callback());
	deferred.promise.name = 2;
	return deferred.promise;
}
readFile1("file1.txt","utf-8").then(function(file1){
	return readFile2(file1.trim(),"utf-8");
}).then(function(file2){
	console.log(file2);
})
```
APIpromise化
方法：
```javascript
var smooth = function(methed){

	return function(){
		var args = Array.prototype.slice.call(arguments,0);
		var deferred = new Deferred();
		args.push(deferred.callback());
		methed.apply(null,args);
		return deferred.promise;
	}
}
```
目标实现：
```javascript
var fs = require("fs");
var readFile1 = function(file,encoding){
	var deferred = new Deferred();
	fs.readFile(file,encoding,deferred.callback());
	deferred.promise.name = 1;
	return deferred.promise;
}
var readFile2 = function(file,encoding){
	var deferred = new Deferred();
	fs.readFile(file,encoding,deferred.callback());
	deferred.promise.name = 2;
	return deferred.promise;
}
readFile1("file1.txt","utf-8").then(function(file1){
	return readFile2(file1.trim(),"utf-8");
}).then(function(file2){
	console.log(file2);
})
```
可以简化为：
```javascript
var fs = require("fs");
var readFile = smooth(fs.readFile);
readFile("file1.txt","utf-8").then(function(file1){
	return readFile(file1.trim(),"utf-8");
}).then(function(file2){
	console.log(file2);
})
```




