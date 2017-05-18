#Zepto是一个轻量级的针对现代高级浏览器的JavaScript库， 它与jquery有着类似的api。
如果你会用jquery，那么你也会用zepto。

##Zepto.js touch模块

### 记录 Zepto.js touch模块

```js

var now, delta, touch = {};
$(document)
    .on('touchstart', startListener)
    .on('touchmove', moveListener)
    .on('touchend', endListener);
```
### 1、是单击还是双击

```js
function startListener(e){
    now = Date.now();
    delta = now - (touch.last || now);

    // 手指连续轻触两次，时间间隔大于0，小于等于.25s，则为双击，反之单击
    if ( delta > 0 && delta <= 250 ) {
        touch.isDoubleTap = true;
    }
    touch.last = now;
}
```
### 2、处理手指长按

```js
var longTapTimeout, longTapDelay = 750;
function longTap() {
    longTapTimeout = null
    if (touch.last) {
        touch.el.trigger('longTap')
        touch = {}
    }
}
function cancelLongTap() {
    if (longTapTimeout) clearTimeout(longTapTimeout)
    longTapTimeout = null
}

function startListener(e){
    // 默认就是长按，如果手指未移动和离开，超过.75s就触发longTap
    longTapTimeout = setTimeout(longTap, longTapDelay)
}
function moveListener(e){
    // 如果手指轻触屏幕后未超过.75s，则取消手指长按监听
    longTapTimeout = setTimeout(longTap, longTapDelay)
}
function endListener(e){
    // 如果手指轻触屏幕后未超过.75s，则取消手指长按监听
    longTapTimeout = setTimeout(longTap, longTapDelay)
}
```

### 3、是滑动(swipe)还是轻触(tap)

```js
// 如果手指移动屏幕超过30像素，则触发相应的滑动事件，swipeLeft, swipeRight, swipeUp, swipeDown
function endListener(e){
    // swipe
    if ((touch.x2 && Math.abs(touch.x1 - touch.x2) > 30) ||
        (touch.y2 && Math.abs(touch.y1 - touch.y2) > 30)) {

        swipeTimeout = setTimeout(function() {
            touch.el.trigger('swipe')
            touch.el.trigger('swipe' + (swipeDirection(touch.x1, touch.x2, touch.y1, touch.y2)))
            touch = {}
        }, 0);
    }
    else {
        // handle tap
        // 关于处理tap事件，请看第四点
    }
}
```

### 4、轻触 tap, singleTap, doubleTap

#### 4.1、何时触发 tap ？

 条件1：手指移动不超过30像素

```js
if ((touch.x2 && Math.abs(touch.x1 - touch.x2) > 30) ||
   (touch.y2 && Math.abs(touch.y1 - touch.y2) > 30)) {
    // swipe
}
else {
    // tap
}
```
 条件2：依据条件1，基本上可以触发tap了，但是还考虑了另一种情况，手指滑动屏幕后又滑动到起始点，那么：
```js
!Math.abs(touch.x1 - touch.x2) > 30) === !Math.abs(touch.y1 - touch.y2) > 30) === true;
```
为了不触发tap事件，这里又加了条件限制，理解这点很重要
```js
if (deltaX < 30 && deltaY < 30) {
    // handle tap
}
```
注意：
```js
deltaX !== Math.abs(touch.x1 - touch.x2);
deltaY !== Math.abs(touch.y1 - touch.y2);
  ```
请看 moveListener 中的代码：

```js
function moveListener(e){
    // ...
    touch.x2 = firstTouch.pageX
    touch.y2 = firstTouch.pageY

    deltaX += Math.abs(touch.x1 - touch.x2)
    deltaY += Math.abs(touch.y1 - touch.y2)
}
```
例: deltaX的计算，你懂得...
```js
if ( Math.abs(touch.x1 - touch.x2) === 10 ) {
    deltaX = 10 + 9 + 8 + ... + 0;
}

```
#### 4.2、处理tap,doubleTap,singleTap三者之间的关系

```js
function cancelAll() {
    if (touchTimeout) clearTimeout(touchTimeout)
    if (tapTimeout) clearTimeout(tapTimeout)
    if (swipeTimeout) clearTimeout(swipeTimeout)
    if (longTapTimeout) clearTimeout(longTapTimeout)
    touchTimeout = tapTimeout = swipeTimeout = longTapTimeout = null
    // 这句很重要，将影响所有需要对touch对象属性判断的语句
    touch = {}
}
function endListener(e){
    tapTimeout = setTimeout(function() {
        var event = $.Event('tap')
        // tap事件对象event可以取消后续绑定的doubleTap, singleTap处理器
        event.cancelTouch = cancelAll
        touch.el.trigger(event)
        // 立即触发双击事件
        if (touch.isDoubleTap) {
            if (touch.el) touch.el.trigger('doubleTap')
            touch = {}
        }
        // 定时.25s后再触发单击事件
        else {
            touchTimeout = setTimeout(function() {
                touchTimeout = null
                if (touch.el) touch.el.trigger('singleTap')
                touch = {}
            }, 250)
        }
    }, 0)
}
```

例如：如何在tap事件处理器中取消 doubleTap或singleTap事件监听器

```js
$('body')
    .on('tap', function(e){
        console.log('tap');
        // 执行下面语句将影响是否触发绑定的 doubleTap或singleTap 处理器
        e.cancelTouch();
    })
    .on('doubleTap', function(e){
        console.log('doubleTap');
    })
    .on('singleTap', function(e){
        console.log('singleTap');
    });

// 'tap'
```

### 5、兼容指针事件系统

```js
// 判断是否是指针事件类型
function isPointerEventType(e, type) {
    return (e.type == 'pointer' + type ||
        e.type.toLowerCase() == 'mspointer' + type)
}

// 判断是否是第一个touch或pointer事件对象
function isPrimaryTouch(event) {
    return (event.pointerType == 'touch' ||
        event.pointerType == event.MSPOINTER_TYPE_TOUCH) && event.isPrimary
}
// 如果是指针类型是 pointerdown 或 pointermove 或 pointerup 且 不是第一个touch 或 pointer 事件对象，返回空，
// 直接屏蔽了第二个、第三...的触摸处理
if ((_isPointerType = isPointerEventType(e, 'down')) && !isPrimaryTouch(e)) return
if ((_isPointerType = isPointerEventType(e, 'move')) && !isPrimaryTouch(e)) return
if ((_isPointerType = isPointerEventType(e, 'up')) && !isPrimaryTouch(e)) return
```
### 6、快捷注册事件

```js
['swipe', 'swipeLeft', 'swipeRight', 'swipeUp', 'swipeDown',
    'doubleTap', 'tap', 'singleTap', 'longTap'
].forEach(function(eventName) {
    $.fn[eventName] = function(callback) {
        return this.on(eventName, callback)
    }
});
```
你可以用 on 方法注册事件，也可以快捷注册，下面两种方式都是一样的，类似jQuery用法
```js
$('body').on('tap', function(){ console.log('body trigger tap event'); });
$('body').tap(function(){ console.log('body trigger tap event'); });
```
### 总结：

touch模块的中所有的事件都支持冒泡，但是不会对原生的touch事件产生影响，另外所有元素绑定的事件都是在文档document元素的touchend处理中触发

如果页面中有一元素在原生touch事件中阻止了冒泡，那么页面中所有元素注册的 zepto touch事件都不会被触发，慎重慎重
