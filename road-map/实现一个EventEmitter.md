### **实现一个 EventEmitter**

Node.js的events 模块对外提供了一个 EventEmitter 对象，用于对 Node.js 中的事件进行统一管理。因为 Node.js 采用了事件驱动机制，而 EventEmitter 就是 Node.js 实现事件驱动的基础。在 EventEmitter 的基础上，Node.js 中几乎所有的模块都继承了这个类，以实现异步事件驱动架构。

#### 基本使用

```javascript
let events = require('events');

let eventEmitter = new events.EventEmitter();

eventEmitter.on('say',function(name){
    console.log('Hello',name);
})

eventEmitter.emit('say','Jonh');

```

以上代码中，新定义的eventEmitter 是接收 events.EventEmitter 模块 new 之后返回的一个实例，eventEmitter 的 emit 方法，发出 say 事件，通过 eventEmitter 的 on 方法监听，从而执行相应的函数。

#### 常用的 EventEmitter 模块的 API

![图片1.png](https://s0.lgstatic.com/i/image6/M01/0E/14/Cgp9HWA8LMiAEVlGAAJEpecSYyo071.png)

#### 实现EventEmitter

那么结合上面介绍的内容，我们一起来实现一个基础版本的EventEmitter，包含基础的one、 off、emit、once、allOff 这几个方法。

```javascript
class EventEmitter {
  constructor() {
    // 用来存放自定义事件，以及自定义事件的回调函数
    this.events = {}
  }

  on(eventName, listener, once = false) {
    if (!eventName || !listener) return
    // 判断listener是否为函数
    if (typeof listener !== 'function') {
      throw new TypeError('listener must be a function')
    }
    let listeners = (this.events[eventName] = this.events[eventName] || [])
    // 防止注册同一个事件名的重复的listener
    if (listeners.indexOf(listener) === -1) {
      // 为每一个添加的listener添加一个是否只执行一次的标识符 once
      listener.once = once
      listeners.push(listener)
    }
    return this
  }

  emit(eventName, ...args) {
    let listeners = this.events[eventName]
    if (!listeners) return
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener.apply(this, args)
      // 如果只需要执行一次需要特殊处理，执行完成之后，将改监听器移除即可
      if (listener.once) {
        this.off(eventName, listener)
      }
    }
    return this
  }

  off(eventName, listener) {
    let listeners = this.events[eventName]
    if (!listeners) return
    let index = -1
    for (let i = 0; i < listeners.length; i++) {
      if (listeners[i] === listener) {
        index = i
        break
      }
    }
    if (index !== -1) {
      listeners.splice(index, 1)
    }
    return this
  }
}
```

再看下 once 方法和 allOff的实现

```javascript
	// 直接调用 on 方法，once 参数传入 true，待执行之后进行 once 处理
  once(eventName, listener) {
    return this.on(eventName, listener, true)
  }

  allOff(eventName) {
    if (eventName && this.events[eventName]) {
      this.events[eventName] = []
    } else {
      this.events = {}
    }
  }
```

从上面的代码中可以看到，once 方法的本质还是调用 on 方法，只不过传入的参数区分和非一次执行的情况。当再次触发 emit 方法的时候，once 绑定的执行一次之后再进行解绑。

这样，alloff 方法也很好理解了，其实就是对内部的 `events` 对象进行清空，清空之后如果再次触发自定义事件，也就无法触发回调函数了。

#### EventEmitter设计模式

上面代码已经很明显了就是 `发布-订阅模式`

另外，观察者模式和发布-订阅模式有些类似的地方，但是在细节方面还是有一些区别的，不要把这两个模式搞混了。发布-订阅模式其实是观察者模式的一种变形，区别在于：**发布-订阅模式在观察者模式的基础上，在目标和观察者之间增加了一个调度中心**。

#### 实际使用场景

Vue中不同组件中的通信，其中一种就是EventBus，它和 EventEmitter 思路类似