
## frame
- events: 点击事件、键盘事件、滚动事件等
- macro: 宏任务，如 setTimeout
- micro: 微任务，如 Promise
- rAF: requestAnimationFrame
- Layout: CSS 计算，页面布局
- Paint: 页面绘制
- rIC: requestIdleCallback

## why fiber
根本原因，是大量的同步计算任务阻塞了浏览器的 UI 渲染。默认情况下，JS 运算、页面布局和页面绘制都是运行在浏览器的主线程当中，他们之间是互斥的关系。如果 JS 运算持续占用主线程，页面就没法得到及时的更新。当我们调用setState更新页面的时候，React 会遍历应用的所有节点，计算出差异，然后再更新 UI。整个过程是一气呵成，不能被打断的。如果页面元素很多，整个过程占用的时机就可能超过 16 毫秒，就容易出现掉帧的现象。

针对这一问题，React 团队从框架层面对 web 页面的运行机制做了优化，得到很好的效果。

## fiber 这个词

也称协程、或者纤程。（ Ruby ）是一种控制流程的让出机制。要理解协程，你得和普通函数一起来看, 以Generator为例:
普通函数执行的过程中无法被中断和恢复：
```js
const tasks = []
function run() {
  let task = []
  // 
  while (task = tasks.shift()) {
    execute(task)
  }
}
```
而 Generator 可以:
```js
const tasks = []
function * run() {
  let task = []

  while (task = tasks.shift()) {
    // 🔴 判断是否有高优先级事件需要处理, 有的话让出控制权
    if (hasHighPriorityEvent()) {
      yield 
    }
    // 处理完高优先级事件后，恢复函数调用栈，继续执行...
    execute(task)
    yield 12
    // 
    yield 12
  }
}
```
```js
next()
next()
```
## fiber

解决主线程长时间被 JS 运算占用这一问题的基本思路，是将运算切割为多个步骤，分批完成。也就是说在完成一部分任务之后，将控制权交回给浏览器，让浏览器有时间进行页面的渲染。等浏览器忙完之后，再继续之前未完成的任务。

旧版 React 通过递归的方式进行渲染，使用的是 JS 引擎自身的函数调用栈，它会一直执行到栈空为止。而Fiber实现了自己的组件调用栈，它以链表的形式遍历组件树，可以灵活的暂停、继续和丢弃执行的任务。实现方式是使用了浏览器的requestIdleCallback这一 API。官方的解释是这样的：

window.requestIdleCallback()会在浏览器空闲时期依次调用函数，这就可以让开发者在主事件循环中执行后台或低优先级的任务，而且不会对像动画和用户交互这些延迟触发但关键的事件产生影响。函数一般会按先进先调用的顺序执行，除非函数在浏览器调用它之前就到了它的超时时间。
有了解题思路后，我们再来看看 React 具体是怎么做的。

## react
React 框架内部的运作可以分为 3 层：

- Virtual DOM 层，描述页面长什么样。
- Reconciler 层，负责调用组件生命周期方法，进行 Diff 运算等。
- Renderer 层，根据不同的平台，渲染出相应的页面，比较常见的是 ReactDOM 和 ReactNative。
这次改动最大的当属 Reconciler 层了，React 团队也给它起了个新的名字，叫Fiber Reconciler。这就引入另一个关键词：Fiber。
Fiber 其实指的是一种数据结构，它可以用一个纯 JS 对象来表示：
```js
const fiber = {
    stateNode,    // 节点实例
    child,        // 子节点
    sibling,      // 兄弟节点
    return,       // 父节点
}
// ReactFiber.js
```
Fiber Reconciler 在执行过程中，会分为 2 个阶段:
- 阶段一，生成 Fiber 树，得出需要更新的节点信息。这一步是一个渐进的过程，可以被打断。
- 阶段二，将需要更新的节点一次过批量更新，这个过程不能被打断。

## Fiber 树

fiber对象 -> 链表  -> 树

需要对React现有的数据结构进行调整，模拟函数调用栈, 将之前需要递归进行处理的事情分解成增量的执行单元，将递归转换成迭代.
一个个 fiber 节点，构成整个组件树。

递归转换成迭代
(循环)


## 任务优先级

上面说了，为了避免任务被饿死，可以设置一个超时时间. 这个超时时间不是死的，低优先级的可以慢慢等待, 高优先级的任务应该率先被执行.

- Immediate(-1) - 这个优先级的任务会同步执行, 或者说要马上执行且不能中断
- UserBlocking(250ms) 这些任务一般是用户交互的结果, 需要即时得到反馈
- Normal (5s) 应对哪些不需要立即感受到的任务，例如网络请求
- Low (10s) 这些任务可以放后，但是最终应该得到执行. 例如分析通知，上报日志（辅助我们监控应用）
- Idle (没有超时时间) 一些没有必要做的任务 (e.g. 比如隐藏的内容), 可能会被饿死

位于：`Scheduler.js`

## 最后
框架能够帮助我们保证应用的性能下限，react 框架层面的优化

[requestidlecallback](https://www.w3.org/TR/requestidlecallback/)