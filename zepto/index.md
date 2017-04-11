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
变量maybeID用来检查selector是否符合css的id选择器写法。
变量maybeClass用来检查selector是否符合css的class选择器写法。
变量nameOnly截取idName或者className的部分。
变量isSimple用来测试nameOnly是否是简单选择器（它测试nameOnly是否只包含数字字母下划线短横线，如果selector的写法是属性选择器之类的，就通不过这个正则）。
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
