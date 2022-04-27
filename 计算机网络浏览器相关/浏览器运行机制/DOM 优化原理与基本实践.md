# DOM 优化原理与基本实践

DOM 优化这块划分为三个小专题：“DOM 优化思路”、“异步更新策略”及“回流与重绘”

DOM 为什么这么慢？
1. JS 引擎和渲染引擎（浏览器内核）是独立实现的。当我们用 JS 去操作 DOM 时，本质上是 JS 引擎和渲染引擎之间进行了“跨界交流”。这个过程的开销不可忽略，次数多了，性能必然受影响。
2. 除了访问DOM节点，可能还会做DOM的修改，而修改会引发**回流**和**重绘**
    +  回流： 对 DOM 的修改引发了 DOM 几何尺寸的变化（比如修改元素的宽、高或隐藏元素等）时，浏览器需要重新计算元素的几何属性（其他元素的几何属性和位置也会因此受到影响），然后再将计算的结果绘制出来。这个过程就是回流（也叫重排）。
    +  重绘：当我们对 DOM 的修改导致了样式的变化、却并未影响其几何属性（比如修改了颜色或背景色）时，浏览器不需重新计算元素的几何属性、直接为该元素绘制新的样式（跳过了上图所示的回流环节）。这个过程叫做重绘。

由此我们可以看出，重绘不一定导致回流，回流一定会导致重绘。回流比重绘做的事情更多，带来的开销也更大。

在开发中，要从代码层面出发，尽可能把回流和重绘的次数最小化。

## 哪些实际操作会导致回流与重绘
+ 最“贵”的操作：改变 DOM 元素的几何属性：当一个DOM元素的几何属性发生变化时，所有和它相关的节点（比如父子节点、兄弟节点等）的几何属性都需要进行重新计算，它会带来巨大的计算量。
+ “价格适中”的操作：改变 DOM 树的结构：这里主要指的是节点的增减、移动等操作。
+ 最容易被忽略的操作：获取一些特定属性的值：当你要用到像这样的属性：offsetTop、offsetLeft、 offsetWidth、offsetHeight、scrollTop、scrollLeft、scrollWidth、scrollHeight、clientTop、clientLeft、clientWidth、clientHeight 时，浏览器为了**即时计算**而获取到的这些值，也会进行回流。
+ 除此之外，当我们调用了 getComputedStyle 方法，或者 IE 里的 currentStyle 时，也会触发回流。原理是一样的，都为求一个“即时性”和“准确性”。

注意：display：none会发生reflow；而visibility：hidden只会触发repaint

### 如何规避回流与重绘
1. 减少频繁的DOM操作，使用JS操作来替换不必要的DOM的更新修改，做最少量的DOM操作
2. 避免逐条改变样式，使用类名去合并样式
```
// 每次单独操作，都去触发一次渲染树更改，从而导致相应的回流与重绘过程。
const container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'

// 加入样式
const basic_style = {
  width: '100px',
  height: '200px',
  border: '10px solid red'
  color: 'red'
}
const container = document.getElementById('container')
container.classList.add('basic_style')
// 合并之后，等于我们将所有的更改一次性发出，用一个 style 请求解决掉了。
```
3. 将 DOM “离线”：一旦给元素设置 display: none，将其从页面上“拿掉”，那么我们的后续操作，将无法触发回流与重绘——这个将元素“拿掉”的操作，就叫做 DOM 离线化。

```
const container = document.getElementById('container')
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
// ...code

// 离线化之后
let container = document.getElementById('container')
container.style.display = 'none'
container.style.width = '100px'
container.style.height = '200px'
container.style.border = '10px solid red'
container.style.color = 'red'
// ...code
container.style.display = 'block'
```




再看一个案例：
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>DOM操作测试</title>
</head>
<body>
  <div id="container"></div>
</body>
<script>
  for(var count=0;count<10000;count++){ 
    document.getElementById('container').innerHTML+='<span> 我是一个小测试</span>'
  }
</script>
</html>
```

这段代码有两个明显的可优化点。

1. 过路费交太多了。我们每一次循环都调用 DOM 接口重新获取了一次 container 元素，DOM访问次数太多。
2. 不必要的 DOM 更改太多了。在10000 次循环里，修改了 10000 次 DOM 树。对 DOM 的修改会引发渲染树的改变、进而去走一个（可能的）回流或重绘的过程，而这个过程的开销是很“贵”的。

修复：
```
let container = document.getElementById('container')
let content = ''
for(let count=0;count<10000;count++){ 
  // 先对内容进行操作
  content += '<span>我是一个小测试</span>'
} 
// 内容处理好了,最后再触发DOM的更改
container.innerHTML = content
```

事实上，考虑JS 的运行速度，比 DOM 快得多这个特性。我们减少 DOM 操作的核心思路，就是让 JS 去给 DOM 分压。

## Event Loop 与异步更新策略

基于 Event Loop 机制，对 Vue 的异步更新策略作探讨.

Event Loop 过程解析:
+ 初始状态：调用栈空。micro 队列空，macro 队列里有且只有一个 script 脚本（整体代码）。
+ 全局上下文（script 标签）被推入调用栈，同步代码执行。在执行的过程中，通过对一些接口的调用，可以产生新的 macro-task 与 micro-task，它们会分别被推入各自的任务队列里。
+ 同步代码执行完了，script 脚本会被移出 macro 队列，**这个过程本质上是队列的 macro-task 的执行和出队的过程**
+ 上一步我们出队的是一个 macro-task，这一步我们处理的是 micro-task。(需要注意的是：当 macro-task 出队时，任务是一个一个执行的；而 micro-task 出队时，任务是一队一队执行的。因此，我们处理 micro 队列这一步，会逐个执行队列中的任务并把它出队，直到队列被清空)
+ **执行渲染操作，更新界面**（重点）。
+ 上述过程循环往复，直到两个队列都清空

### 渲染的时机

假如我想要在异步任务里进行DOM更新，我该把它包装成 micro 还是 macro 呢？

```
// task是一个用于修改DOM的回调
setTimeout(task, 0)
```

+ 现在 task 被推入的 macro 队列。但因为 script 脚本本身是一个 macro 任务，所以本次执行完 script 脚本之后，下一个步骤就要去处理 micro 队列了，再往下就去执行了一次 render。
+ 但本次render我的目标task其实并没有执行，想要修改的DOM也没有修改，因此这一次的render其实是一次无效的render。
+ macro 不 ok，我们转向 micro 试试看。我用 Promise 来把 task 包装成是一个 micro 任务：`Promise.resolve().then(task)`
+ 当结束了对 script 脚本的执行，紧接着就去处理 micro-task 队列了，micro-task 处理完，DOM 修改好了，紧接着就可以走 render 流程了。
+ 不需要再消耗多余的一次渲染，不需要再等待一轮事件循环，直接为用户呈现最即时的更新结果。

所以：**当我们需要在异步任务中实现 DOM 修改时，把它包装成 micro 任务是相对明智的选择**


### 异步更新策略——以 Vue 为例：

当我们使用 Vue 或 React 提供的接口去更新数据时，这个更新并不会立即生效，而是会被推入到一个队列里。待到适当的时机，队列中的更新任务会被**批量触发**。这就是异步更新。

异步更新可以帮助我们避免过度渲染，是“让 JS 为 DOM 分压”的典范之一。
异步更新的特性在于它只看结果，因此渲染引擎不需要为过程买单。
最典型的例子，比如有时我们会遇到这样的情况：

```
// 任务一
this.content = '第一次测试'
// 任务二
this.content = '第二次测试'
// 任务三
this.content = '第三次测试'
```

+ 我们在三个更新任务中对同一个状态修改了三次，如果我们采取传统的同步更新策略，那么就要操作三次 DOM。但本质上需要呈现给用户的目标内容其实只是第三次的结果，也就是说只有第三次的操作是有意义的——我们白白浪费了两次计算。

+ 但如果我们把这三个任务塞进异步更新队列里，它们会先**在 JS 的层面上被批量执行完毕**。当流程走到渲染这一步时，它仅仅需要针对有意义的计算结果操作一次 DOM——这就是异步更新的妙处。

### Vue状态更新手法：nextTick


```
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 检查上一个异步任务队列（即名为callbacks的任务数组）是否派发和执行完毕了。pending此处相当于一个锁
  if (!pending) {
    // 若上一个异步任务队列已经执行完毕，则将pending设定为true（把锁锁上）
    pending = true
    // 是否要求一定要派发为macro任务
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      // 如果不说明一定要macro 你们就全都是micro
      microTimerFunc()
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

我们看到，Vue 的异步任务默认情况下都是用 Promise 来包装的，也就是是说它们都是 micro-task。

继续细化解析一下 macroTimeFunc() 和 microTimeFunc() 两个方法：
```
// macro首选setImmediate 这个兼容性最差
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
    isNative(MessageChannel) ||
    // PhantomJS
    MessageChannel.toString() === '[object MessageChannelConstructor]'
  )) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  // 兼容性最好的派发方式是setTimeout
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

```
// 简单粗暴 不是ios全都给我去Promise 如果不兼容promise 那么你只能将就一下变成macro了
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
} else {
  // 如果无法派发micro，就退而求其次派发为macro
  microTimerFunc = macroTimerFunc
}
```

我们注意到，无论是派发 macro 任务还是派发 micro 任务，派发的任务对象都是一个叫做 flushCallbacks 的东西，这个东西做了什么呢？

```
function flushCallbacks () {
  pending = false
  // callbacks在nextick中出现过 它是任务数组（队列）
  const copies = callbacks.slice(0)
  callbacks.length = 0
  // 将callbacks中的任务逐个取出执行
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

总结：Vue 中每产生一个状态更新任务，它就会被塞进一个叫 callbacks 的数组（此处是任务队列的实现形式）中。这个任务队列在被丢进 micro 或 macro 队列之前，会先去检查当前是否有异步更新任务正在执行（即检查 pending 锁）。如果确认 pending 锁是开着的（false），就把它设置为锁上（true），然后对当前 callbacks 数组的任务进行派发（丢进 micro 或 macro 队列）和执行。设置 pending 锁的意义在于保证状态更新任务的有序进行，避免发生混乱。

