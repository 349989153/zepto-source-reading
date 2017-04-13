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
zepto.Z = function(dom, selector) {
  return new Z(dom, selector)
}
zepto.Z.prototype = Z.prototype = $.fn
```
**在这里，我实在无法理解`zepto.Z.prototype = $.fn`这句话的含义，照理说，`Z.prototype = $.fn`已经够用了，`zepto.Z`不过是一个函数，而`Z`才是类，为何要把`zepto.Z.prototype`也设为`$.fn`?**
于是我把源码改成：
```javascript
// zepto.Z.prototype = Z.prototype = $.fn
Z.prototype = $.fn
```
结果zepto.js自带的测试跑不通，测试似乎进入了死循环，一直出不来结果。

于是我开始重新审视我的那句话：**`zepto.Z`不过是一个函数，而`Z`才是类。在js里，其实本没有类的构造函数，只有函数，
用过`new`之后，也就成了类的构造函数。所以，会不会是在哪儿用了类似`new zepto.Z()`的写法了？**
于是我到源码中搜索，果不其然：
```javascript
zepto.isZ = function(object) {
  return object instanceof zepto.Z
}
$.fn = {
  constructor: zepto.Z,
  length: 0
  ....
  ....
}
```
然后我改成：
```javascript
zepto.isZ = function(object) {
  //return object instanceof zepto.Z
  return object instanceof Z
}
$.fn = {
  // constructor: zepto.Z,
  constructor: Z,
  length: 0
  ....
  ....
}
// zepto.Z.prototype = Z.prototype = $.fn
Z.prototype = $.fn
```
并且跑了测试，这回测试跑通。。。。好吧，所以为啥在这两个地方要用zepto.Z而不是Z，又成了一个问题，难道只是作者个人喜好而已？


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
## jQuery的基础架构（版本1.12.5-pre）
和zepto相同，我们也把jquery的主要结构提炼出来：
```javascript
(function( global, factory ) {

  factory( global );

}(typeof window !== "undefined" ? window : this, function( window, noGlobal ) {
	  
}));
```
首先看到最简的形式，jquery就和zepto不一样。

zepto是声明一个全局变量Zepto，它的值是匿名函数的返回值，在匿名函数的函数体内，声明并返回了$函数，可以说，zepto的主体是写在匿名函数里的。

而jquery没有声明全局变量，它的匿名函数也只是执行一句`factory( global );`而已，真正的函数体是匿名函数的第二个参数。

然后是jQuery函数表达式：
```javascript
jQuery = function (selector, context) {

  // The jQuery object is actually just the init constructor 'enhanced'
  // Need init if jQuery is called (just allow error to be thrown if not included)
  return new jQuery.fn.init(selector, context);
}
// 省略
return jQuery;
```
这里和zepto.js是有一点点小区别的，zepto.js生成Z对象是依靠调用zepto.Z，所以$.fn会挂载到Z.prototype上面：
```javascript
zepto.Z = function(dom, selector) {
  return new Z(dom, selector)
}
```
而jQuery则比较简单粗暴，直接把jQuery.fn.init当作构造函数，所以jQuery.fn会挂载到jQuery.fn.init.prototype上面：
```javascript
init = jQuery.fn.init = function( selector, context, root ){...}
init.prototype = jQuery.fn;
```
其他方面基本和zepto.js一样，把jQuery弄成一个函数，这样就能够实现$("myDiv")的调用；并且在jQuery函数上定义各种属性，这样就能够实现如$.each()的调用。