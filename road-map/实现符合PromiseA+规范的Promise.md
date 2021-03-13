### Promise/A+ 规范

1. “promise”：是一个具有 then 方法的对象或者函数，它的行为符合该规范。

2. “thenable”：是一个定义了 then 方法的对象或者函数。

3. “value”：可以是任何一个合法的 JavaScript 的值（包括 undefined、thenable 或 promise）。

4. “exception”：是一个异常，是在 Promise 里面可以用 throw 语句抛出来的值。

5. “reason”：是一个 Promise 里 reject 之后返回的拒绝原因。

### Promise 的内部状态描述

1. 一个 Promise 有三种状态：pending、fulfilled 和 rejected。

2. 当状态为 pending 状态时，即可以转换为 fulfilled 或者 rejected 其中之一。

3. 当状态为 fulfilled 状态时，就不能转换为其他状态了，必须返回一个不能再改变的值。

4. 当状态为 rejected 状态时，同样也不能转换为其他状态，必须有一个原因的值也不能改变。

### then方法

一个 Promise 必须拥有一个 then 方法来访问它的值或者拒绝原因，then方法执行后返回一个新的Promise。

then 方法有两个参数：

> promise.then(onFulfilled, onRejected)

onFulfilled 和 onRejected 都是可选参数。

如果 onFulfilled 是函数，则当 Promise 执行结束之后必须被调用，最终返回值为value，value会在then方法内部包装成一个新的promise并返回，这也是then方法后可以再次调用then的原因。而 onRejected 除了最后返回的是 reason 外，其他方面和 onFulfilled 在规范上的表述基本一样

> promise2 = promise1.then(onFulfilled, onRejected);

**注意：**这两个回调方法都是异步执行的

### 一步步实现 Promise

#### 构造函数

Promise 构造函数接受一个 executor 函数，executor 函数执行完同步或者异步操作后，调用它的两个参数 resolve 和 reject。

```javascript
class MyPromise {
  constructor(executor) {
    /**
     * Promise当前状态
     */
    this.status = 'pending'
    /**
     * Promise的值
     */
    this.data = undefined
    /**
     * Promise resolve时的回调函数集
     */
    this.onResolvedCallback = []
    /**
     * Promise reject时的回调函数集
     */
    this.onRejectedCallback = []

    /**
     * 执行 executor 并传入相应的参数，用try catch包裹防止报错
     */
    try {
      executor(this.resolve, this.reject)
    } catch (err) {
      this.reject(err)
    }
  }

  resolve(value) {
    // 防止多次调用resolve，status 只可以改变一次，暂不处理异步执行行为
    if (this.status === 'pending') {
      this.status = 'fulfilled'
      this.data = value
      for (let i = 0; i < this.onResolvedCallback.length; i++) {
        this.onResolvedCallback[i](this.data)
      }
    }
  }

  reject(reason) {
    // 防止多次调用reject，status 只可以改变一次，暂不处理异步执行行为
    if (this.status === 'pending') {
      this.status = 'rejected'
      this.data = reason;
      for (let i = 0; i < this.onRejectedCallback.length; i++) {
        this.onRejectedCallback[i](this.data);
      }
    }
  }
}
```

#### 实现then方法

```javascript
then(onResolved, onRejected) {
    let promise
    // 根据标准，如果then的参数不是function，则需要忽略它
    onResolved = typeof onResolved === 'function' ? onResolved : () => {}
    onRejected = typeof onRejected === 'function' ? onRejected : () => {}

    if (this.status === 'fulfilled') {
      return (promise = new MyPromise((resolve, reject) => {}))
    }

    if (this.status === 'rejected') {
      return (promise = new MyPromise((resolve, reject) => {}))
    }

    if (this.status === 'pending') {
      return (promise = new MyPromise((resolve, reject) => {}))
    }
  }
```

从上面的代码中可以看到，我们在 then 方法内部先初始化了 Promise 的对象，用来存放执行之后返回的 Promise，并且还需要判断 then 方法传参进来的两个参数必须为函数，这样才可以继续执行。

```javascript
then(onResolved, onRejected) {
    let self = this
    let promise
    // 根据标准，如果then的参数不是function，则需要忽略它
    onResolved = typeof onResolved === 'function' ? onResolved : () => {}
    onRejected = typeof onRejected === 'function' ? onRejected : () => {}

    if (self.status === 'fulfilled') {
      return (promise = new MyPromise((resolve, reject) => {
        try {
          let value = onResolved(self.data)
          if (value instanceof Promise) {
            value.then(resolve, reject)
          } else {
            resolve(value)
          }
        } catch (e) {
          reject(e)
        }
      }))
    }

    if (self.status === 'rejected') {
      return (promise = new MyPromise((resolve, reject) => {
        try {
          let value = onRejected(self.data)
          if (value instanceof Promise) {
            value.then(resolve, reject)
          } else {
            resolve(value)
          }
        } catch (e) {
          reject(e)
        }
      }))
    }

    if (self.status === 'pending') {
      return (promise = new MyPromise((resolve, reject) => {
        self.onResolvedCallback.push(function (data) {
          try {
            let value = onResolved(data)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })

        self.onRejectedCallback.push(function (reason) {
          try {
            let value = onRejected(reason)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }
```

在 Promise/A+ 规范中，onResolved 和 onRejected 这两项函数需要异步调用，因此需要改为异步调用的形式

```javascript
resolve(value) {
    if (value instanceof Promise) {
      return value.then(this.resolve, this.reject)
    }
    // 异步执行所有的回调函数
    setTimeout(() => {
      if (this.status === 'pending') {
        this.status = 'fulfilled'
        this.data = value
        for (let i = 0; i < this.onResolvedCallback.length; i++) {
          this.onResolvedCallback[i](this.data)
        }
      }
    })
  }

  reject(reason) {
    // 异步执行所有的回调函数
    setTimeout(() => {
      if (this.status === 'pending') {
        this.status = 'rejected'
        this.data = reason
        for (let i = 0; i < this.onRejectedCallback.length; i++) {
          this.onRejectedCallback[i](this.data)
        }
      }
    })
  }

  then(onResolved, onRejected) {
    let self = this
    let promise
    // 根据标准，如果then的参数不是function，则需要忽略它
    onResolved = typeof onResolved === 'function' ? onResolved : () => {}
    onRejected = typeof onRejected === 'function' ? onRejected : () => {}

    if (self.status === 'fulfilled') {
      return (promise = new MyPromise((resolve, reject) => {
        // 异步执行onResolved
        setTimeout(() => {
          try {
            let value = onResolved(self.data)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (self.status === 'rejected') {
      return (promise = new MyPromise((resolve, reject) => {
        // 异步执行onRejected
        setTimeout(() => {
          try {
            let value = onRejected(self.data)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (self.status === 'pending') {
      // 这里之所以没有异步执行，是因为这些函数必然会被resolve或reject调用，而resolve或reject函数里的内容已是异步执行，构造函数里的定义
      return (promise = new MyPromise((resolve, reject) => {
        self.onResolvedCallback.push(function (data) {
          try {
            let value = onResolved(data)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })

        self.onRejectedCallback.push(function (reason) {
          try {
            let value = onRejected(reason)
            if (value instanceof Promise) {
              value.then(resolve, reject)
            } else {
              resolve(value)
            }
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }
```

到此基本实现了一个符合标准的 then 方法。但是标准里提到了，还要支持不同的 Promise 进行交互，其中详细指定了如何通过 then 的实参返回的值来决定 Promise2（then中返回的新的promise） 的状态

```javascript
// 用于统一处理 then 中返回的 promise 的状态变化
resolvePromise(promise, value, resolve, reject) {
    let self = this
    let then
    let thenCalledOrThrow = false
    if (promise === value) {
      return reject(new TypeError('Chaining cycle detected for promise!'))
    }
    if (value instanceof Promise) {
      if (value.status === 'pending') {
        value.then((v) => {
          self.resolvePromise(promise, v, resolve, reject)
        }, reject)
      } else {
        value.then(resolve, reject)
      }
      return
    }
    if (value !== null && (typeof value === 'object' || typeof value === 'function')) {
      try {
        then = value.then
        if (typeof then === 'function') {
          then.call(
            value,
            function onResolved(v) {
              if (thenCalledOrThrow) return
              thenCalledOrThrow = true
              return self.resolvePromise(promise, v, resolve, reject)
            },
            function onRejected(r) {
              if (thenCalledOrThrow) return
              thenCalledOrThrow = true
              return reject(r)
            }
          )
        } else {
          resolve()
        }
      } catch (e) {
        if (thenCalledOrThrow) return
        thenCalledOrThrow = true
        return reject(e)
      }
    } else {
      resolve(value)
    }
  }

	then(onResolved, onRejected) {
    let self = this
    let promise
    // 根据标准，如果then的参数不是function，则需要忽略它
    onResolved = typeof onResolved === 'function' ? onResolved : () => {}
    onRejected = typeof onRejected === 'function' ? onRejected : () => {}

    if (self.status === 'fulfilled') {
      return (promise = new MyPromise((resolve, reject) => {
        // 异步执行onResolved
        setTimeout(() => {
          try {
            let value = onResolved(self.data)
            // resolvePromise 做统一状态管理
            self.resolvePromise(promise, value, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (self.status === 'rejected') {
      return (promise = new MyPromise((resolve, reject) => {
        // 异步执行onRejected
        setTimeout(() => {
          try {
            let value = onRejected(self.data)
            // resolvePromise 做统一状态管理
            self.resolvePromise(promise, value, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }

    if (self.status === 'pending') {
      // 这里之所以没有异步执行，是因为这些函数必然会被resolve或reject调用，而resolve或reject函数里的内容已是异步执行，构造函数里的定义
      return (promise = new MyPromise((resolve, reject) => {
        self.onResolvedCallback.push(function (data) {
          try {
            let value = onResolved(data)
            // resolvePromise 做统一状态管理
            self.resolvePromise(promise, value, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })

        self.onRejectedCallback.push(function (reason) {
          try {
            let value = onRejected(reason)
            // resolvePromise 做统一状态管理
            self.resolvePromise(promise, value, resolve, reject)
          } catch (e) {
            reject(e)
          }
        })
      }))
    }
  }

  catch(onRejected) {
    return this.then(null, onRejected)
  }
```

