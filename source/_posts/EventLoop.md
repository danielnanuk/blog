---
title: JavaScript中的EventLoop
date: 2018-06-07 15:51:41
tags:
---
JavaScript是单线程/异步的, 使用Event Loop来实现并发模型.
模型如下:

![EventLoop](/images/EventLoop.png  "JavaScript EventLoop")

术语:

**Stack**: 栈, 函数调用形成了栈

**Heap**: 堆, 用于存放JavaScript对象

**Queue**: 队列, 也称为task queue, 用于存放需要执行的任务

### Event Loop

当Stack为空时, Event Loop取得下一个任务放入栈底进行执行, 若取不到则阻塞

Event Loop的特点有:

* Run to completion
    
    即, 当Queue中的一个任务开始执行, 事件循环将这个任务执行完成, 中途不会中断/暂停(原因思考, 由于JavaScript的单线程, 不需要像很多后端语言那样实现多线程以及上下文切换, 这一特点是JavaScript简单的根源)

* 只有一个Event Loop
    
    有两种Event Loop: Worker Event Loop和Browser Event Loop. 
    W3C原话是这么说的:
    There is also at most one event loop per unit of related similar-origin browsing contexts (though several units of related similar-origin browsing contexts can have a shared event loop).
    可以理解为同源的浏览器上下文只有一个Event Loop, 但多个源的浏览器上下文可以共用一个Event Loop(这就可以理解为什么老版本IE只要有一个页面崩掉, 则整个浏览器崩掉, 因为都公用一个Event Loop)

* 一个Event Loop可能有多个task queue

    一个任务有任务源(task source), 相同任务源的任务，只能放到一个任务队列中; 不同任务源的任务，可以放到不同任务队列中。

#### Task Type
macro-task: 包含script, setTimeout, setInterval, I/O, UI render
micro-task: Promise, Object.observer, MutationObserver

#### 执行模型
1. 找到最早的任务作为task
2. 设置当前执行的任务为task
3. 执行任务task
4. 设置当前执行的任务为null
5. 从对应的task queue移除task
6. microtasks
7. 如果是浏览器Event Loop, 处理渲染
8. 如果是Worker Event Loop 处理对应的逻辑
9. 继续执行步骤1

由以上模型可以知道, 在当前任务中添加的macro-task 和 micro-task, micro-task总是先于 macro-task.

具体参见: [W3C Event Loop的执行模型](https://www.w3.org/TR/html5/webappapis.html#event-loops-processing-model)

#### Never Blocking
   关于JavaScript永不阻塞这一块, 其实主要原因是大量采用event和handler机制, 如果在某个handler中进行大量计算(例如使用while(true)), 
 会导致当前的任务卡住, 整个页面block掉.

猜一下这段代码会输出什么:
```javascript
setTimeout(function() {
  console.log(1)
}, 0);
new Promise(function(resolve) {
  console.log(2);
  resolve();
  console.log(3);
}).then(function() {
  console.log(4);
});
console.log(5);
```
[JSFiddle](https://jsfiddle.net/danielnanuk/hassjgr7/)

注意
    new Promise(fn)构造器中的fn是同步执行的

