## React架构的演变-从同步到异步

> 为了加深对 React 更新机制的理解，本文转载于：
> 作者：Shenfq
> 链接：https://juejin.im/post/6875681311500025869

React 16 之所以要进行一次大的重构，是因为之前的版本中有一些不可避免的缺陷，一些更新操作需要由同步改为异步。

### React 15 是如何进行一次 setState 更新的

```javascript
import React from 'react'

class App extends React.Component {
  state = {
    val: 0
  }

  componentDidMount() {
    this.setState({
      val: this.state.val + 1
    })
    console.log('first setState', this.state)
    this.setState({
      val: this.state.val + 1
    })
    console.log('second setState', this.state)
    this.setState({
      val: this.state.val + 1
    }, () => {
      console.log('in callback', this.state)
    })
  }

	render() {
    return (
    	<div>val: {this.state.val}</div>
    )
  }
}
```

由于在 `React 生命周期中` 以及 `React提供的事件流中` ，多次 setState 会被合并成一次，这里虽然连续进行了三次 setState，`state.val` 的值实际上只计算了一次。

**结果是**

```javascript
// first setState => { val: 0 }
// second setState => { val: 0 }
// in callback => { val: 1 }
```

很多人称 `setState` 为异步操作，所以导致 `setState` 之后并不能获取到最新值，**其实这个观点是错误的**。

**`setState` 是一次同步操作**，只是每次操作之后并没有立即执行，而是将 `setState` 进行了缓存，mount 事件结束或者事件操作结束，才会拿出所有的 state 进行一次计算。

如果 `setState` 脱离了 **`React 的生命周期`** 或 **`React提供的事件流`**，setState 之后就能立即拿到更新结果

修改上面的代码，将 setState 放入 setTimeout 中，在下一个任务队列进行执行

```javascript
import React from 'react'

class App extends React.Component {
  state = {
    val: 0
  }

  componentDidMount() {
    setTimeout(() => {
      this.setState({
        val: this.state.val + 1
      })
      console.log('first setState', this.state)
      this.setState({
        val: this.state.val + 1
      })
      console.log('second setState', this.state)
    })
  }

	render() {
    return (
    	<div>val: {this.state.val}</div>
    )
  }
}
```

根据执行结果可以看出，setState 之后就能立即看到 `state.val` 的值得变化

```javascript
// first setState => { val: 1 }
// second setState => { val: 2 }
```

为了更加深入理解 setState，下面简单讲解一下React 15 中 setState 的更新逻辑，下面的代码是对源码的一些精简，并非完整逻辑。

### 旧版本 setState 源码简析

setState 的主要逻辑都在 ReactUpdateQueue 中实现，在调用 setState 后，并没有立即修改 state，而是将传入的参数放到了组件内部的 `_pendingStateQueue` 中，之后调用 `enqueueUpdate` 来进行更新操作。

```javascript
// 对外暴露的 React.Component
function ReactComponent() {
  this.updater = ReactUpdateQueue
}

// setState 方法挂载在原型链上
ReactComponent.prototype.setState = function setState(partialState, callback) {
  // 调用 setState 后，会调用内部的 updater.enqueueSetState
  this.updater.enqueueSetState(this, partialState)
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState')
  }
}

var ReactUpdateQueue = {
  enqueueSetState(component, partialState) {
    // 在组件的 _pendingStateQueue 上暂存新的 state
    if (!component._pendingStateQueue) {
      component._pendingStateQueue = []
    }
    var queue = component._pendingStateQueue
    queue.push(partialState)
    enqueueUpdate(component)
  },
  enqueueCallback(component, callback, callerName) {
    // 在组件的 _pendingCallbacks 上暂存 callback
    if (component._pendingCallbacks) {
      component._pendingCallbacks.push(callback)
    } else {
      component._pendingCallbacks = [callback]
    }
    enqueueUpdate(component)
  }
}
```

`enqueueUpdate` 首先会通过 `batchingStrategy.isBatchingUpdates` 判断当前是否在更新流程中，如果不在更新流程中，会调用 `batchingStrategy.batchedUpdates()` 进行更新。如果在流程中，会将待更新的组件放入 `dirtyComponents` 中进行缓存。

```javascript
var dirtyComponents = []
function enqueueUpdate(component) {
  if (!batchingStrategy.isBatchingUpdates) {
    // 开始进行批量更新
    batchingStrategy.batchedUpdates(enqueueUpdate, component)
    return
  }
  // 如果在更新流程，则将组件放入脏组件队列，表示组件待更新
  dirtyComponents.push(component)
}
```

`batchingStrategy` 是 React 进行批处理的一种策略，改策略的实现基于 `Transaction`，虽然名字和数据库事务一样，但是做的事情却不一样

```javascript
class ReactDefaultBatchingStrategyTrancaction extends Transaction {
  constructor() {
    this.reinitializeTransaction()
  }

  getTransactionWrappers() {
    return [
      {
        initialize: () => {},
        close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
      },
      {
        initialize: () => {},
        close: () => {
          ReactDefaultBatchingStrategyTrancaction.isBatchingUpdates = false
        },
      },
    ]
  }
}

var transaction = new ReactDefaultBatchingStrategyTrancaction()

var batchingStrategy = {
  // 判断是否在更新流程中
  isBatchingUpdates: false,
  // 开始进行批量更新
  batchedUpdates(callback, component) {
    // 获取之前的更新状态
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategyTrancaction.isBatchingUpdates
    // 将更新状态改为 true
    ReactDefaultBatchingStrategyTrancaction.isBatchingUpdates = true
    if (alreadyBatchingUpdates) {
      // 如果已经在更新状态中，等待之前的更新结束
      return callback(callback, component)
    } else {
      // 进行更新
      return transaction.perform(callback, null, component)
    }
  },
}
```

`Transaction` 通过 `perform` 方法启动，然后通过扩展的 `getTransactionWrappers` 获取一个数组，该数组内存在多个 wrapper 对象，每个对象包含两个属性：`initilize`、`close`。`perform` 会先调用所有的 `wrapper.initialize` ，然后调用传入的回调，最后调用所有的 `wrapper.close`。

```javascript
class Transaction {
  reinitializeTransaction() {
    this.transactionWrappers = this.getTransactionWrappers()
  }

  perform(method, scope, ...param) {
    this.initializeAll(0)
    var ret = method.call(scope, ...param)
    this.closeAll(0)
    return ret
  }

  initializeAll(startIndex) {
    var transactionWrappers = this.transactionWrappers
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i]
      wrapper.initialize.call(this)
    }
  }

  closeAll(startIndex) {
    var transactionWrappers = this.transactionWrappers
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i]
      wrapper.close.call(this)
    }
  }
}
```

其执行流程如下

```javascript
/*
*                       wrappers (injected at creation time)
*                                      +        +
*                                      |        |
*                    +-----------------|--------|--------------+
*                    |                 v        |              |
*                    |      +---------------+   |              |
*                    |   +--|    wrapper1   |---|----+         |
*                    |   |  +---------------+   v    |         |
*                    |   |          +-------------+  |         |
*                    |   |     +----|   wrapper2  |--------+   |
*                    |   |     |    +-------------+  |     |   |
*                    |   |     |                     |     |   |
*                    |   v     v                     v     v   | wrapper
*                    | +---+ +---+   +---------+   +---+ +---+ | invariants
* perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
* +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
*                    | |   | |   |   |         |   |   | |   | |
*                    | |   | |   |   |         |   |   | |   | |
*                    | |   | |   |   |         |   |   | |   | |
*                    | +---+ +---+   +---------+   +---+ +---+ |
*                    |  initialize                    close    |
*                    +-----------------------------------------+
*/
```



我们简化一个代码，再重新看一下 setState 的流程

```javascript
// 1. 调用 Component.setState
ReactComponent.prototype.setState = function (partialState) {
  this.updater.enqueueSetState(this, partialState);
};

// 2. 调用 ReactUpdateQueue.enqueueSetState，将 state 值放到 _pendingStateQueue 进行缓存
var ReactUpdateQueue = {
  enqueueSetState(component, partialState) {
    var queue = component._pendingStateQueue || (component._pendingStateQueue = []);
    queue.push(partialState);
    enqueueUpdate(component);
  }
}

// 3. 判断是否在更新过程中，如果不在就进行更新
var dirtyComponents = [];
function enqueueUpdate(component) {
  // 如果之前没有更新，此时的 isBatchingUpdates 肯定是 false
  if (!batchingStrategy.isBatchingUpdates) {
    // 调用 batchingStrategy.batchedUpdates 进行更新
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }
  dirtyComponents.push(component);
}

// 4. 进行更新，更新逻辑放入事务中进行处理
var batchingStrategy = {
  isBatchingUpdates: false,
  // 注意：此时的 callback 为 enqueueUpdate 
  batchedUpdates: function (callback, component) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;
    ReactDefaultBatchingStrategy.isBatchingUpdates = true;
    if (alreadyBatchingUpdates) {
      // 如果已经在更新状态中，重新调用 enqueueUpdate，将 component 放入 dirtyComponents
      return callback(callback, component);
    } else {
      // 进行事务操作
      return transaction.perform(callback, null, component);
    }
  }
};
```

启动事务可以拆分成三步来看：

1. 先执行 wrapper 的initialize，此时的 initialize 都是一些空函数，可以直接跳过；
2. 然后执行 callback （也就是 `enqueueUpdate`），执行 enqueueUpdate 时，由于已经进入了更新状态，`batchingStrategy.isBatchingUpdates` 被修改成了 `true`，所以最后还是会把 component 放入脏组件队列，等待更新；
3. 后面执行两个 close 方法，第一个方法的 `flushBatchedUpdates` 是用来进行组件更新的，第二个方法用来修改更新状态，表示更新已经结束。

```javascript
getTransactionWrappers () {
  return [
    {
      initialize: () => {},
      close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
    },
    {
      initialize: () => {},
      close: () => {
        ReactDefaultBatchingStrategy.isBatchingUpdates = false;
      }
    }
  ]
}
```

`flushBatchedUpdates` 里面会取出所有的脏组件队列进行 diff，最后更新到 DOM。

```javascript
function flushBatchedUpdates() {
  if (dirtyComponents.length) {
    runBatchedUpdates()
  }
}

function runBatchedUpdates() {
  // 省略一些去重和排序的操作
  for (var i = 0; i < dirtyComponents.length; i++) {
    var component = dirtyComponents[i]
    // 判断组件是否需要更新，然后进行 diff 操作，最后更细 DOM
    ReactReconciler.performUpdateIfNecessary(component)
  }
}
```

`performUpdateIfNecessary()`  会调用 `Component.updateComponent()`，在 `updateComponent` 中，会从 `_pendingStateQueue` 中取出所有的值来更新。

```javascript
// 获取最新的 state
_processPendingState() {
  var inst = this._instance
  var queue = this._pendingStateQueue

  var nextState = {
    ...inst.state
  }

  for (var i = 0; i<queue.length; i++) {
    var partial = queue[i]
    Object.assign(nextState, typeof partial === 'function' ? partial(inst, nextState) : partial)
  }

  return nextState
}

updateComponent(prevParentElement, nextParentElement) {
  var inst = this._instance
  var prevProps = prevParentElement.props
  var nextProps = nextParentElement.props
  var nextState = _processPendingState()
  var shouldUpdate = !shallowEqual(prevProps, nextProps) || !shallowEqual(inst.state, nextState)
  if (shouldUpdate) {
    // diff、update DOM
  } else {
    inst.props = nextProps
    inst.state = nextState
  }
  // 后续的操作包括判断组件是否需要更新、diff、更新到 DOM
}
```

### setState 合并原因

按照上面讲解的逻辑，setState 的时候，`batchingStrategy.isBatchingUpdates` 为 `false` 会开启一个事务，将组件放入脏组件队列，最后进行更新操作，而且这里都是同步操作。

那么按照这个逻辑，setState 之后，我们可以立即拿到最新的 state。

然而，事实并非如此，在 React 的生命周期及其事件流中，`batchingStrategy.isBatchingUpdates` 的值早就被修改成了 `true`

见下图

![React 事件流](https://img-blog.csdnimg.cn/20201101200513364.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JvYW1pbmdjb2Rl,size_16,color_FFFFFF,t_70#pic_center)

![React 生命周期](https://img-blog.csdnimg.cn/20201101200513360.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JvYW1pbmdjb2Rl,size_16,color_FFFFFF,t_70#pic_center)

在组件 mount 和事件调用的时候，都会调用 `batchedUpdates`，这个时候已经开始了事务，所以只要不脱离 React，不管多少次 setState 都会把其组件放入脏组件队列等待更新。一旦脱离 React 的管理，比如 `SetTimeout` 中，setState 立马变成单打独斗。

### Concurrent 模式

React 16 引入的 Fiber 架构，就是为了后续的异步渲染能力做铺垫，虽然架构已经转换，但是异步的渲染能力并没有正式上线。

> Concurrent 模式是 React 的新功能，可帮助应用保持响应，并根据用户的设备性能和网速进行适当的调整。

除了 Concurrent 模式，React 还提供了另外两个模式， Legacy 模式依旧是同步更新的方式，可以认为和旧版本保持一致的兼容模式，而 Blocking 模式是一个过渡版本。

`Concurrent` 模式说白就是让组件更新异步化，切分时间片，渲染之前的调度、diff、更新都只在指定时间片进行，如果超时就暂停放到下个时间片进行，中途给浏览器一个喘息的时间。

> 浏览器是单线程，它将 GUI 描绘，时间器处理，事件处理，JS 执行，远程资源加载统统放在一起。当做某件事，只有将它做完才能做下一件事。如果有足够的时间，浏览器是会对我们的代码进行编译优化（JIT）及进行热代码优化，一些 DOM 操作，内部也会对 reflow 进行处理。reflow 是一个性能黑洞，很可能让页面的大多数元素进行重新布局。
>
> 浏览器的运作流程: `渲染 -> tasks -> 渲染 -> tasks -> 渲染 -> ....`
>
> 这些 tasks 中有些我们可控，有些不可控，比如 setTimeout 什么时候执行不好说，它总是不准时；资源加载时间不可控。但一些JS我们可以控制，让它们分派执行，tasks的时长不宜过长，这样浏览器就有时间优化 JS 代码与修正 reflow ！
>
> 总结一句，**就是让浏览器休息好，浏览器就能跑得更快**。
>
> -- by 司徒正美 [《React Fiber架构》](https://zhuanlan.zhihu.com/p/37095662)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201101200814601.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JvYW1pbmdjb2Rl,size_16,color_FFFFFF,t_70#pic_center)

### 如何使用

虽然很多文章都在介绍 Concurrent 模式，但是这个能力并没有真正上线，想要使用只能安装实验版本。也可以直接通过这个 cdn ：[unpkg.com/browse/reac…](https://unpkg.com/browse/react@0.0.0-experimental-94c0244ba/) 。

```bash
npm install react@experimental react-dom@experimental
```

如果要开启 Concurrent 模式，不能使用之前的 `ReactDOM.render`，需要替换成 `ReactDOM.createRoot`，而在实验版本中，由于AP不够稳定，需要通过 `ReactDOM.unstable_createRoot` 来启用 `Concurrent` 模式。

```javascript
import ReactDOM from 'react-dom'
import App from './App'

ReactDOM.unstable_createRoot(document.getElementById('root')).render(<App></App>)
```

### setState 合并更新

还记得之前 React15 的案例中，setTimeout 中进行 setState ，`state.val` 的值会立即发生变化。同样的代码，我们拿到 Concurrent 模式下运行一次。

```javascript
import React from 'react'

class App extends React.Component {
  state = { val: 0 }
  componentDidMount() {
    setTimeout(() => {
      // 第一次调用
      this.setState({ val: this.state.val + 1 })
      console.log('first setState', this.state)
  
      // 第二次调用
      this.setState({ val: this.state.val + 1 })
      console.log('second setState', this.state)
      
      this.setState({ val: this.state.val + 1 }, () => {
        console.log(this.state)
      })
    })
  }
  render() {
    return <div> val: { this.state.val } </div>
  }
}

export default App
```

**结果是**

```javascript
// first setState => { val: 0 }
// second setState => { val: 0 }
// in callback => { val: 1 }
```

说明在 Concurrent 模式下，即使脱离了 React 的生命周期，setState 依旧能够合并更新。主要原因是 Concurrent 模式下，真正的更新操作被移到了下一个事件队列中，类似于 Vue 的 nextTick。

### 更新机制变更

我们修改一下 demo，然后看下点击按钮之后的调用栈。

```javascript
import React from 'react';

class App extends React.Component {
  state = { val: 0 }
  clickBtn() {
    this.setState({ val: this.state.val + 1 });
  }
  render() {
    return (<div>
      <button onClick={() => {this.clickBtn()}}>click add</button>
      <div>val: { this.state.val }</div>
    </div>)
  }
}

export default App;
```

`onClick` 触发后，进行 setState 操作，然后调用 enqueueState 方法，到这里看起来好像和之前的模式一样，但是后面的操作基本都变了，因为 React 16 中已经没有了事务一说。

> Component.setState() => enqueueState() => scheduleUpdate() => scheduleCallback() => requestHostCallback(flushWork) => postMessage()

真正的异步化逻辑就在 `requestHostCallback` 、 `postMessage` 里面，这是 React 内部自己实现的一个调度器：[github.com/facebook/re…](https://github.com/facebook/react/blob/v16.13.1/packages/scheduler/index.js)

```javascript
function unstable_scheduleCallback(priorityLevel, calback) {
  var currentTime = getCurrentTime()
  var startTime = currentTime + delay
  var newTask = {
    id: taskIdCounter++,
    startTime: startTime,           // 任务开始时间
    expirationTime: expirationTime, // 任务终止时间
    priorityLevel: priorityLevel,   // 调度优先级
    callback: callback,             // 回调函数
  };
  if (startTime > currentTime) {
    // 超时处理，将任务放到 taskQueue，下一个时间片执行
    // 源码中其实是 timerQueue，后续会有个操作将 timerQueue 的 task 转移到 taskQueue
  	push(taskQueue, newTask)
  } else {
    requestHostCallback(flushWork)
  }
  return newTask
}
```

requestHostCallback 的实现依赖于 [MessageChannel](https://developer.mozilla.org/zh-CN/docs/Web/API/MessageChannel)，但是 MessageChannel 在这里并不是做消息通信用的，而是利用它的异步能力，给浏览器一个喘息的机会。说起 MessageChannel，Vue 2.5 的 nextTick 也有使用，但是 2.6 发布时又取消了。

MessageChannel 会暴露两个对象，`port1` 和 `port2`，`port1` 发送的消息能被 `port2` 接收，同样 `port2` 发送的消息也能被 `port1` 接收，只是接收消息的时机会放到下一个 macroTask 中。

```javascript
var { port1, port2 } = new MessageChannel()
// port1 接收 port2 的消息
port1.onmessage = function (msg) { console.log('MessageChannel exec') }
// port2 发送消息
port2.postMessage(null)

new Promise(r => r()).then(() => console.log('promise exec'))
setTimeout(() => console.log('setTimeout exec'))

console.log('start run')
```

**执行结果为**

```javascript
// start run
// promise exec
// MessageChannnel exec
// setTimeout exec
```

可以看到，`port1` 接收消息的时机比 Promise 所在的 microTask 要晚，但是早于 setTimeout。React 利用这个能力，给了浏览器一个喘息的时间，不至于被饿死。

还是回到代码层面，看看 React 是如何利用 MessageChannel 的。

```javascript
var isMessageLoopRunning = false // 更新状态
var scheduledHostCallback = null // 全局的回调
var channel = new MessageChannel()
var port = channel.port2

channel.port1.onmessage = function() {
  if (scheduledHostCallback !== null) {
    var currentTime = getCurrentTime()
    // 重置超时时间
    deadline = currentTime + yieldInterval
    var hasTimeRemaining = true
    // 执行callback
    var hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime)
    if (!hasMoreWork) {
      // 已经没有任务了，修改状态
      isMessageLoopRunning = false
      scheduledHostCallback = null
    } else {
      // 还有任务，放入下个任务队列执行，给浏览器喘息的机会
      port.postMessage(null)
    }
  } else {
    isMessageLoopRunning = false
  }
}

requestHostCallback = function (callback) {
  // 将 callback 挂载到 scheduledHostCallback
  scheduledHostCallback = callback
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true
    // 推送消息，下个队列调用 callback
    port.postMessage(null)
  }
}
```

再看看之前传入的 callback（`flushWork`），调用 `workLoop`，取出 taskQueue 中的任务执行。

```javascript
// 精简了相当多的代码
function flushWork(hasTimeRemaining, initialTime) {
  var currentTime = initialTime
  // scheduleCallback 进行了 taskQueue 的 push 操作
  // 这里是获取之前时间片未进行的操作
  currentTask = peek(taskQueue)
  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime) {
      // 超时需要中断任务
      break
    }

    currentTask.callback() // 执行任务回调
    currentTime = getCurrentTime() // 重置当前时间
    currentTask = peek(taskQueue)
  }

  // 如果当前任务不为空，表明是超时中断，返回true
  if (currentTask !== null) {
    return true
  } else {
    return false
  }
}
```

可以看出，React 通过 expirationTime 来判断是否超时，如果超时就把任务放到后面来执行。所以，异步模型中 setTimeout 里面进行 setState，只要当前时间片没有结束（currentTime 小于 expirationTime），依旧可以将多个 setState 合并成一个。

接下来我们再做一个实验，在 setTimeout 中连续进行 500 次的 setState，看看最后生效的次数

```javascript
import React from 'react'

class App extends React.Component {
  state = { val: 0 }
  clickBtn() {
    for (let i = 0; i < 500; i++) {
      setTimeout(() => {
        this.setState({ val: this.state.val + 1 })
      })
    }
  }
  render() {
    return (<div>
      <button onClick={() => {this.clickBtn()}}>click add</button>
      <div>val: { this.state.val }</div>
    </div>)
  }
}

export default App
```

先看看同步模式下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020110120070773.gif#pic_center)

再看看异步模式下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201101200724168.gif#pic_center)

最后 setState 的次数是 81 次，表明这里的操作在 81 个时间片下进行的，每个时间片更新了一次，同一个时间片中的 setState 会进行合并。

















