```js
// 基础变量的定义
const STATUS = {
  PENDING: 'PENDING',
  FULFILLED: 'FULFILLED',
  REJECTED: 'REJECTED',
};

const isFunction = (fn) => typeof fn === 'function';

// 难点
const doThenFunc = (promise, value, resolve, reject) => {
  // 循环引用
  if (promise === value) {
    reject(new TypeError('Chaining cycle detected for promise'));
    return;
  }

  // 如果 value 是 promise 对象
  if (value instanceof Deferred) {
    // 调用 then 方法，等待结果
    value.then(
      function (val) {
        doThenFunc(promise, val, resolve, reject);
      },
      function (reason) {
        reject(reason);
      }
    );
    return;
  }

  // 如果非 promise 对象，则直接返回
  resolve(value);
};

class Deferred {
  constructor(callback) {
    this.value = undefined;
    this.status = STATUS.PENDING;

    this.resolveQueus = [];
    this.rejectQueus = [];

    let called; // 用于判断状态是否被修改

    const resolve = (value) => {
      if (called) return;
      called = true;

      // 异步调用
      setTimeout(() => {
        this.value = value;
        // 修改状态
        this.status = STATUS.FULFILLED;

        // 调用回调
        for (const fn of this.resolveQueus) {
          fn(this.value);
        }
      });
    };

    const reject = (reason) => {
      if (called) return;
      called = true;

      // 异步调用
      setTimeout(() => {
        this.value = reason;
        // 修改状态
        this.status = STATUS.REJECTED;

        // 调用回调
        for (const fn of this.rejectQueus) {
          fn(this.value);
        }
      });
    };

    try {
      callback(resolve, reject);
    } catch (err) {
      // 出现异常直接进行 reject
      reject(err);
    }
  }

  then(onResolve, onReject) {
    // 解决值穿透 .then().then().then((val) => {})
    onResolve = isFunction(onResolve)
      ? onResolve
      : (value) => {
          return value;
        };
    onReject = isFunction(onReject)
      ? onReject
      : (reason) => {
          throw reason;
        };

    if (this.status === STATUS.PENDING) {
      // 将回调放入队列中
      const rejectQueus = this.rejectQueus;
      const resolveQueus = this.resolveQueus;

      const promise = new Deferred((resolve, reject) => {
        // 暂存到成功回调等待调用
        resolveQueus.push(function (innerValue) {
          try {
            const value = onResolve(innerValue);
            // 改变当前 promise 的状态
            // resolve(value)
            doThenFunc(promise, value, resolve, reject);
          } catch (err) {
            reject(err);
          }
        });

        // 暂存到失败回调等待调用
        rejectQueus.push(function (innerValue) {
          try {
            const value = onReject(innerValue);
            // 改变当前 promise 的状态
            // resolve(value)
            doThenFunc(promise, value, resolve, reject);
          } catch (err) {
            reject(err);
          }
        });
      });
      return promise;
    } else {
      const innerValue = this.value;
      const isFulFilled = this.status === STATUS.FULFILLED;
      const promise = new Deferred((resolve, reject) => {
        try {
          const value = isFulFilled
            ? onResolve(innerValue) // 成功状态调用 onResolve
            : onReject(innerValue); // 失败状态调用 onReject
          // 返回结果给后面的 then
          // resolve(value)
          doThenFunc(promise, value, resolve, reject);
        } catch (err) {
          reject(err);
        }
      });
      return promise;
    }
  }

	// 相当于 then 方法的一个简写
  catch(onReject) {
    return this.then(null, onReject);
  }

  static resolve(value) {
    return new Deferred((resolve, reject) => {
      resolve(value);
    });
  }

  static reject(reason) {
    return new Deferred((resolve, reject) => {
      reject(reason);
    });
  }

  static all(promises) {
    // 非数组参数，抛出异常
    if (!Array.isArray(promises)) {
      return Deferred.reject(new TypeError('args mu be an array'));
    }

    // 用于存储每个 promise 对象的结果
    const result = [];
    const length = promises.length;
    // 如果 remaining 归零，表示所有 promise 对象已经 fulfilled
    let remaining = length;
    const promise = new Deferred((resolve, reject) => {
      // 如果数组为空，则返回空结果
      if (promises.length === 0) {
        return resolve(result);
      }

      function done(index, value) {
        doThenFunc(
          promise,
          value,
          (val) => {
            // resolve 的结果放入 result 中
            result[index] = val;
            if (--remaining === 0) {
              // 如果所有的 promise 都已经返回结果
              // 然后运行后面的逻辑
              resolve(result);
            }
          },
          reject
        );
      }

      // 放入异步队列
      setTimeout(() => {
        for (let i = 0; i < length; i++) {
          done(i, promises[i]);
        }
      });
    });
    return promise;
  }

  static race(promises) {
    if (!Array.isArray(promises)) {
      return Deferred.reject(new TypeError('args must be an array'));
    }

    const length = promises.length;
    const promise = new Deferred(function (resolve, reject) {
      if (promises.length === 0) return resolve([]);

      function done(value) {
        doThenFunc(promise, value, resolve, reject);
      }

      // 放入异步队列
      setTimeout(() => {
        for (let i = 0; i < length; i++) {
          done(promises[i]);
        }
      });
    });
    return promise;
  }
}

```

