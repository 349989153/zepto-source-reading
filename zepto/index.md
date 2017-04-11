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
