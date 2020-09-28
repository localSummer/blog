[TOC]



### 先明白几个关键词

1. Koa 中间件及其洋葱圈模型
   1. [koa-compose](https://github.com/koajs/compose/blob/master/index.js)
   2. [常见中间件原理浅析](https://blog.csdn.net/roamingcode/article/details/106428691)
2. Promise 链式调用
   1. [Promise 链式调用](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Using_promises)

### 问题场景

假设有一个大表单页面，表单提交之前需要做以下几件事

1. 表单字段校验（比如邮箱格式校验）
2. 个别字段的敏感词校验
3. 提交字段名或字段值格式化处理（比如时间格式）
4. 提交数据到后端

### 实现一：流水式处理

```javascript
// 表单提交需做几下4个阶段的处理
const submit = () => {
  /**
   * 1. 表单校验
   * 此处省略若干行表单字段校验逻辑
   */
  if (!formValid) {
    return false;
  }

  /**
   * 2. 敏感词校验
   * 此处省略若干行敏感词校验逻辑
   */
  if (!sensitiveValid) {
    return false;
  }

  /**
   * 3. 表单字段格式化处理
   * 此处省略若干行字段格式化处理逻辑
   */
  formFields.format();

  /**
   * 4. 表单提交
   * 此处省略若干行表单提交逻辑
   */
  formFields.submit();
};
```

`submit` 函数包含4种不同的处理逻辑，这样如果再想填写别的处理逻辑较为困难，后期维护成本高，同时违反了开闭原则

### 实现二：Promise链式调用

##### 拆分表单提交时要处理的各个子逻辑

```javascript
let formValidPromise = new Promise((resolve, reject) => {
  /**
   * 1. 表单校验
   * 此处省略若干行表单字段校验逻辑
   */
  if (formValid) {
    resolve(true);
  } else {
    resolve(false);
  }
});

let sensitiveValidPromise = new Promise((resolve, reject) => {
  /**
   * 2. 敏感词校验
   * 此处省略若干行敏感词校验逻辑
   */
  if (sensitiveValid) {
    resolve(true);
  } else {
    resolve(false);
  }
});

let formFormatPromise = new Promise((resolve, reject) => {
  /**
   * 3. 表单字段格式化处理
   * 此处省略若干行字段格式化处理逻辑
   */
  resolve(true);
});

let formSubmitPromise = new Promise((resolve, reject) => {
  /**
   * 4. 表单提交
   * 此处省略若干行表单提交逻辑
   */
  formFields.submit();
  resolve(true);
});
```

##### 汇总各个子逻辑到 `submit` 函数中

```javascript
const submit = () => {
  // 链式调用
  formValidPromise().then((isValid) => {
    if (isValid) {
      return formValidPromise();
    }
    return reject({
      formValid: false
    });
  }).then((sensitiveValid)  => {
    if (sensitiveValid) {
      return sensitiveValidPromise();
    }
    return reject({
      sensitiveValid: false
    });
  }).then(() => {
    return formFormatPromise();
  }).then(() => {
    return formSubmitPromise();
  }).catch(error => {
    console.log('error: ', error);
  });
};

const submit = async () => {
  // 基于 async/await 的同步调用
  try {
    await formValidPromise();
    await sensitiveValidPromise();
    await formFormatPromise();
    await formSubmitPromise();
  } catch(error) {
    // 捕获 reject 错误
    console.log('error: ', error);
  }
};
```

1. 抽离各个子处理逻辑以便于后期维护，减少对submit函数的污染
2. Promise链式调用使得整个提交流程更加可控

同时引出如下的问题

1. 当加入新的子处理逻辑需要修改 `submit` 函数
2. 调整子处理逻辑的顺序需要修改 `submit` 函数

### 实现三：使用可插拔的中间件模式

##### 1. 直接引用 `koa-compose` 提供的 `compose` 函数，用来组合我们所提供的中间件

```javascript
/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */

function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

##### 2. 提供类似 `Koa` 的中间件管理机制以及入口执行机制

```javascript
// 极简版
class Middleware {
  constructor () {
    this.middleare = [];
  }

  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    this.middleware.push(fn);
    return this;
  }

  init() {
    // 组合中间件的调用过程
    const fn = compose(this.middleware);

    const dispatch = () => {
      // 创建一个空的上下文对象
      const ctx = Object.create(null);
      return this.dispatch(ctx, fn)
    }

    return dispatch;
  }

  dispatch(ctx, fnMiddleware) {
    return fnMiddleware(ctx).then(() => {
      // 各个中间件执行完毕后会进入到这里
      console.log('ctx: ', ctx);
    }).catch((error) => {
      console.log(error);
    });
  }
}

export default Middleware;
```

##### 3. 具体使用

```javascript
let middleware = new Middleware();

middleare.use(async (ctx, next) => {
  console.log('ctx1: ', ctx);
  await next();
});

middleare.use(async (ctx, next) => {
  console.log('ctx2: ', ctx);
  await next();
});

let dispatch = middleware.init();

dispatch() // 依次执行中间件
```

##### 4. 结合最初的问题场景

```javascript
let formValidMd = async (ctx, next) => {
  /**
   * 1. 表单校验中间件
   * 此处省略若干行表单字段校验逻辑
   */
  if (formValid) {
    ctx.formValid = true;
    await next();
  } else {
    ctx.formValid = false;
  }
};

let sensitiveValidMd = async (ctx, next) => {
  /**
   * 2. 敏感词校验中间价
   * 此处省略若干行敏感词校验逻辑
   */
  if (sensitiveValid) {
    ctx.sensitiveValid = true;
    await next();
  } else {
    ctx.sensitiveValid = false;
  }
};

let formFormatMd = async (ctx, next) => {
  /**
   * 3. 表单字段格式化处理中间件
   * 此处省略若干行字段格式化处理逻辑
   */
  ctx.formFormat = true;
  await next();
};

let formSubmitMd = async (ctx, next) => {
  /**
   * 4. 表单提交中间件
   * 此处省略若干行表单提交逻辑
   */
  formFields.submit();
  ctx.formSubmit = true;
  await next();
};

// 加载中间件，也可以使用 use 的链式调用
let middleware = new Middleware();
middleare.use(formValidMd);
middleare.use(sensitiveValidMd);
middleare.use(formFormatMd);
middleare.use(formSubmitMd);

let dispatchSubmit = middleare.init();

```

下面来看看 `submit` 的函数实现

```javascript
const submit = () => {
  dispatchSubmit();
}
```

到目前为止我们的 `submit` 函数中代码越来越少

下面来回顾一下 `实现二` 中待解决的问题

1. 当加入新的子处理逻辑需要修改 `submit` 函数 ---> 现在添加只需要添加一个子逻辑处理中间件，然后 `use` 一下了事
2. 调整子处理逻辑的顺序需要修改 `submit` 函数 ---> 现在换顺序只需要调整中间件 `use` 的前后关系

子处理逻辑完全与 `submit` 函数解耦，符合开放封闭设计原则。
虽然写的代码别原来多了，但是维护成本也降低不少，具体取舍以需求所在场景，具体分析，具体解决。







