# Zepto.js
这里记录了zepto的核心--Zepto.js的源码阅读

## Zepto.qsa
```javascript
// `$.zepto.qsa` is Zepto's CSS selector implementation which
// uses `document.querySelectorAll` and optimizes for some special cases, like `#id`.
// This method can be overridden in plugins.
var simpleSelectorRE = /^[\w-]*$/;
zepto.qsa = function (element, selector) {
  var found,
    maybeID = selector[0] == '#',
    maybeClass = !maybeID && selector[0] == '.',
    nameOnly = maybeID || maybeClass ? selector.slice(1) : selector, // Ensure that a 1 char tag name still gets checked
    isSimple = simpleSelectorRE.test(nameOnly)
  return (element.getElementById && isSimple && maybeID) ? // Safari DocumentFragment doesn't have getElementById
    ( (found = element.getElementById(nameOnly)) ? [found] : [] ) :
    (element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11) ? [] :
      slice.call(
        isSimple && !maybeID && element.getElementsByClassName ? // DocumentFragment doesn't have getElementsByClassName/TagName
          maybeClass ? element.getElementsByClassName(nameOnly) : // If it's simple, it could be a class
            element.getElementsByTagName(selector) : // Or a tag
          element.querySelectorAll(selector) // Or it's not simple, and we need to query all
      )
}
```
变量`maybeID`用来检查`selector`是否符合`css`的`id`选择器写法。

变量`maybeClass`用来检查`selector`是否符合`css`的`class`选择器写法。

变量`nameOnly`截取`idName`或者`className`的部分。

变量`isSimple`用来测试`nameOnly`是否是简单选择器（它测试`nameOnly`是否只包含数字字母下划线短横线，如果`selector`的写法是属性选择器之类的，就通不过这个正则）。

`isSimple && maybeID`首先检测了`selector`必为`#id-name_1`这种类似形式，然后对于`id`使用`getElementById`的api来获取元素，放到返回的数组里。

如果上面那一步走不通，则判断`element`的类型是否为`Element`或者`Document`或者`Document Fragment`，如果不是，返回空数组；如果是，而且`isSimple && !maybeID && maybeClass`，
那么可能是`className`，调用`getElementsByClassName`；如果是`isSimple && !maybeID && !maybeClass`，那么`selector`还有可能是`tagName`，调用`getElementsByTagName`；
如果以上情况都不是，最后调用`querySelectorAll`。

总之`Zepto.qsa`的作用是，输入`element和selector`，输出选到的元素组成的数组。在查询中，它尽量判断情况并选择使用`getElementByxxx`方法，最后才会选择使用
`querySelectorAll`方法。这是因为尽管`querySelectorAll`方法比较便捷，但是[性能上差的太多](https://www.zhihu.com/question/24702250/answer/29162146)。

对于`slice.call`的用法不清楚的，请移步[Array.prototype.slice及其他Array方法](https://segmentfault.com/a/1190000008940666)，简单说，`slice.call`就是把一个类数组
对象转化成一个真正的数组对象。

对于上面三元运算符的`??::`形式不清楚的，请移步[三目（三元）运算符??::的形式](https://segmentfault.com/a/1190000008939524)。

以上的两篇文章也是我的原创。
## Zepto.init
`Zepto.init`是`zepto`代码中的核心，为什么这样说呢？请看：
```javascript
// `$` will be the base `Zepto` object. When calling this
// function just call `$.zepto.init, which makes the implementation
// details of selecting nodes and creating Zepto collections
// patchable in plugins.
$ = function(selector, context){
  return zepto.init(selector, context)
}
```
这是我们熟悉的`$函数`，常见的比如`$("#div1")`或者`$(function(){})`就是调用的`$.zepto.init`：
```javascript
// `$.zepto.init` is Zepto's counterpart to jQuery's `$.fn.init` and
// takes a CSS selector and an optional context (and handles various
// special cases).
// This method can be overridden in plugins.
zepto.init = function (selector, context) {
  var dom
  // If nothing given, return an empty Zepto collection
  if (!selector) return zepto.Z()
  // Optimize for string selectors
  else if (typeof selector == 'string') {
    selector = selector.trim()
    // If it's a html fragment, create nodes from it
    // Note: In both Chrome 21 and Firefox 15, DOM error 12
    // is thrown if the fragment doesn't begin with <
    if (selector[0] == '<' && fragmentRE.test(selector))
      dom = zepto.fragment(selector, RegExp.$1, context), selector = null
    // If there's a context, create a collection on that context first, and select
    // nodes from there
    else if (context !== undefined) return $(context).find(selector)
    // If it's a CSS selector, use it to select nodes.
    else dom = zepto.qsa(document, selector)
  }
  // If a function is given, call it when the DOM is ready
  else if (isFunction(selector)) return $(document).ready(selector)
  // If a Zepto collection is given, just return it
  else if (zepto.isZ(selector)) return selector
  else {
    // normalize array if an array of nodes is given
    if (isArray(selector)) dom = compact(selector)
    // Wrap DOM nodes.
    else if (isObject(selector))
      dom = [selector], selector = null
    // If it's a html fragment, create nodes from it
    else if (fragmentRE.test(selector))
      dom = zepto.fragment(selector.trim(), RegExp.$1, context), selector = null
    // If there's a context, create a collection on that context first, and select
    // nodes from there
    else if (context !== undefined) return $(context).find(selector)
    // And last but no least, if it's a CSS selector, use it to select nodes.
    else dom = zepto.qsa(document, selector)
  }
  // create a new Zepto collection from the nodes found
  return zepto.Z(dom, selector)
}
```
`zepto.init`接收两个参数：`selector`一般当作选择器，和`context`，调用选择器的上下文。`context`的理解一般是作为root的节点。

首先，如果没传`selector`，那么通过`zepto.Z()`返回一个空的zepto类。然后，如果`selector`是一个字符串的话，检查这个字符串是不是`html`片段。这里`fragmentRE = /^\s*<(\w+|!)[^>]*>/`
这个正则需要理解一下:
```javascript
/**
 * ^\s*    任意空白符开头
 * <       接着一个<
 * (\w+|!) 接着一个以上的字母数字下划线，或者一个感叹号,并且匹配储存在RegExp.$1，换句话说，把tagName拿了出来.
 * [^>]*   任意不是>的字符
 * >       一个>
 * @type {RegExp}
 */
var fragmentRE = /^\s*<(\w+|!)[^>]*>/
```
如果是`html`片段，则调用`zepto.fragment`创建一个节点，并且把`selector`置为`null`。

接着如果`selector`是`string`而且传入了`context`，调用`$(context).find(selector)`去`context`下找`selector`，如果没有传`context`，则在`document`下调用`zepto.qsa`。
**这里有一个问题：为什么传了`context`调用的是`find`方法，而对`document`则是用`zepto.qsa`，难道不可以`$(document).find(selector)`吗**

接着，如果`selector`不是`string`而是`function`，那么在文档ready的时候调用这个`function`。

如果`selector`是一个Zepto的对象集，那么直接返回它。比如`var zObj = $("#myDiv");console.log(zObj == $(zObj));// true`

如果`selector`既不是对象集也不是`string`也不是`function`: 如果是类数组，那么`dom`变量存一个数组，被`compact`处理过之后，里面的元素都不为null；如果是一个`Object`，那么dom变量存一个数组，把
selector存进去；如果是一个`new String()`创建的selector，那么测试字符串是否符合html片段；再往后面的就和前面一样了。最后，使用`zepto.Z()`创建一个zepto对象集并返回。
## Zepto.matches
```javascript
zepto.matches = function(element, selector) {
    if (!selector || !element || element.nodeType !== 1) return false
    var matchesSelector = element.matches || element.webkitMatchesSelector ||
                          element.mozMatchesSelector || element.oMatchesSelector ||
                          element.matchesSelector
    if (matchesSelector) return matchesSelector.call(element, selector)
    // fall back to performing a selector:
    var match, parent = element.parentNode, temp = !parent
    if (temp) (parent = tempParent).appendChild(element)
    match = ~zepto.qsa(parent, selector).indexOf(element)
    temp && tempParent.removeChild(element)
    return match
  }
```
