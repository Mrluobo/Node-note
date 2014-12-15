# 深入迁出node学习笔记一————模块

标签（空格分隔）： node javascript

---

##CommonJS规范
###模块引用
```javascript
var math = require('math');
```
###模块定义
在模块定义中，module对象代表模块本身，exports对象用于导出当前模块的方法或者变量，并且是唯一出口。exports对象是module的属性。Node中，一个文件就是一个模块。
```javascript
//math.js
exports.add = function(){
    var sum = 0,
        i = 0,
        args = arugments,
        l = args.length;
    while(i<l){
        sum += args[i++];
    }
    return sum;
}
```
在另外一个文件中：
```javascript
var math = require('math');
exports.increment = function(val){
    return math.add(val,1);
}
```
---
##Node模块
###缓存
Node会对引入过的模块进行缓存，缓存的是编译执行之后的对象。
###模块路径
```javascript
console.log(module.paths);
//执行后结果为：
//['/test/node_mocules','/node_modules']
//从当前目录开始，遍历每个父目录的node_modules目录。直到查找到目标文件。
```
###文件扩展名
Node中在没有扩展名的时候会依次按照```.js```、```.node```、```.json```进行补全尝试。
如果没有查找到文件却得到一个目录，则node会将此目录当做一个包来处理。

```javascript
console.log(require.extensions);
//require.extensions可以直到系统中已有的扩展加载方式。
```
由于exports是以形参的形式传入的，故此当直接给exports赋值会失败，所以，当想要要达到通过require引入一个对象的效果，需要赋值给module.exports。

---
##包和NPM
符合CommonJS规范的包目录应含有以下文件。
 1. package.json  包描述文件。
 2. bin  用于存放可执行的二进制文件。
 3. lib  用于存放javascript二代码目录。
 4. doc  用于存放文档的目录。
 5. test  用于存放单元测试用例的代码。


package.json中的必须字段。

 1. name
 2. description
 3. version
 4. keywords
 5. maintainers
 6. contributors
 7. bugs
 8. licenses
 9. repositories
 10. dependencies
 11. homepage
 12. scripts
 13. author
 14. main
 15. devDependencies
###本地安装方法：
```
npm install 文件目录
```

###从非官方源安装
```
npm install underscore --registry=http://registry.url
```
###更改默认源
```
npm config set registry http://registry.url
```
---

##AMD规范
定义方式：
```javascript
define(id,dependencies,factory);
```
其中id和dependencies可选。
---
##CMD规范
因为AMD需要在声明模块时指定所有的依赖，并且通过形参的形式传递给模块，例如：
```javascript
define(['dep1','dep2'],function(dep1,dep2){
return something
})
```
而CMD则认为应该动态引入依赖，示例如下：
```javascript
define(function(require,exports,module){
    var math = require('math');
})
```


