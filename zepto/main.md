## Zepto的基础架构
源码里把主要结构提炼出来的话：
```javascript
var Zepto = (function() {
  var $, zepto = {};
  zepto.Z = function(dom, selector) {
    return new Z(dom, selector)
  }
  zepto.init = function(selector, context) {...}// zepto.init()返回的是一个Zepto对象
  
  $ = function(selector, context){
    return zepto.init(selector, context)
  }
  $.extend = function(target){...}
  $.each = function(elements, callback){...}
  $.fn = {
    constructor: zepto.Z,
    length: 0
    ....
    ....
  }
  ....
  ....
  zepto.Z.prototype = Z.prototype = $.fn
  $.zepto = zepto
  
  return $
})()

window.Zepto = Zepto
window.$ === undefined && (window.$ = Zepto)

```

首先定义一个全局变量`Zepto`，它是匿名函数的值。接着来到匿名函数里，发现声明了变量`$`并且`return $`，现在明白了，前面的`Zepto`其实就是这个`$`。
```javascript
var Zepto = (function() {
  var $;
  return $
})()
```
好来看看这个`$`到底是什么，发现这个`$`其实是个函数，它接收两个参数，返回`zepto.init()`的值。并且对于这个`$函数`，在后面为它添加了许多的属性，比如`$.extend`, `$.each`, `$.fn`等。我想，**返回一个
函数，并且往这个函数上加很多自定义属性**的做法，是由`zepto.js`设定的使用方法决定的。`$("#myDiv")`是`zepto.js`最基础的用法，可以看到这其实就是个函数调用语句，类似于`foo("#myDiv")`，所以返回的
主体（也就是`$`），必须是一个`function`。
```javascript
$ = function(selector, context){
  return zepto.init(selector, context)
}
$.extend = function(target){...}
$.each = function(elements, callback){...}
$.fn = {
  constructor: zepto.Z,
  length: 0
  ....
  ....
}
```
然后进阶地，`zepto.js`得实现链式调用：`$("myDiv").text("hello")`，这就要求`$()`调用的时候返回一个实例对象（`zepto.js`里命名为`z`），并且`text`等方法定义在`Z类`的`prototype`上，这样一阶链就能够实现
；如果需要实现多阶链如：`$("myDiv").text("hello").show()`，就要求自定义在`Z类`的`prototype`上的方法，每个都需要返回一个新的`Z类`型。本来，这些方法直接定义在`Z类`的`prototype`上就好，但是考虑到可扩展
性，`zepto.js`先把它们定义在`$.fn`这个`{}`里，然后再让`Z.prototype`指向`$.fn`，这样，随着`$`被return，`$.fn`也会暴露出来，如果你对`$.fn`进行修改，就能对`zepto.js`所定义的方法进行增删改，极大地加强了可扩展性。
```javascript
zepto.Z.prototype = Z.prototype = $.fn
```

并且可以看到，随着`$.zepto = zepto`,在`zepto`这个`{}`上定义的方法，也随之暴露了出来。所以这里也可以看到闭包的有用之处：闭包之内都可以理解为私有变量，而这些私有变量如果需要暴露出来，依附着`return`
的共有变量，就可以实现。
```javascript
$.zepto = zepto
```
最后，把这个`Zepto`变量挂载到`Window`上，并且如果`$`没有定义的，把`Zepto`挂载到`Window.$`上，以便能在页面里直接调用`$()`
```javascript
window.Zepto = Zepto
window.$ === undefined && (window.$ = Zepto)
```

