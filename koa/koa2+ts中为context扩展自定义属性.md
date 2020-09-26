### 问题来源

为了简化 `ctx.body` 赋值操作，想要在 `ctx` 扩展两个自定义方法， `success`  及 `error`

使用起来如下

```javascript
// 响应成功状态请求
ctx.success({
  username: 'test'
});

// 等价于
ctx.body = {
  code: 1,
  data: {
    username: 'test'
  }
};

// 响应失败状态请求
ctx.error("参数不正确");

// 等价于
ctx.body = {
  code: 0,
  data: null,
  msg: '参数不正确'
};
```

`success`、`error` 这两个方法的扩展是基于 `koa` 中间件的套路来做的

其核心代码如下

```typescript
const koaResponse = async (ctx: Koa.Context, next: Koa.Next) => {
  ctx.success = (data = null, status = Types.EResponseStatus.SUCCESS) => {
    ctx.status = status;
    ctx.body = {
      code: Types.EResponseCode.SUCCESS,
      data
    };
  };

  ctx.error = (
    msg = Types.EResponseMsg.DEFAULT_ERROR,
    data = null,
    status = Types.EResponseStatus.SUCCESS
  ) => {
    ctx.status = status;
    ctx.body = {
      code: Types.EResponseCode.ERROR,
      data,
      msg
    };
  };

  next();
};
```

具体使用时便会遇到问题

```typescript
// 给路由添加了一个 请求参数  校验的中间件 和 一个 请求核心逻辑处理的中间件
router.get('/', Validator.validLogin, UserController.login);
```

请求参数校验中间件

```typescript
// 这里使用假数据做测试
class Validator {
  static async validLogin(ctx: Koa.Context, next: Koa.Next) {
    const result = loginModel.check({
      username: 'test',
      email: 'test@qq.com',
      age: 20
    });

    if (
      (Object.keys(result) as ['username', 'email', 'age']).filter((name) => result[name].hasError)
        .length > 0
    ) {
      ctx.error(Types.EResponseMsg.INVALID_PARAMS); // error 类型丢失，没有代码提示
    } else {
      next();
    }
  }
}
```

请求核心逻辑处理中间件

```typescript
class UserController {
  static async login(ctx: Koa.Context, next: Koa.Next) {
    ctx.success({ // success 类型丢失，没有代码提示
      username: 'test'
    });
    next();
  }
}
```

### 问题解决过程

#### 试验一

app.ts做如下修改

```typescript
// 实例化 app 时，传入自定义属性作为 defaultContext
const app = new Koa<{}, {
  success: Function;
  error: Function;
}>();

// logger 中做测试
app.use(async (ctx, next) => {
  // ctx 类型为
  /* (parameter) ctx: Koa.ParameterizedContext<{}, {
    success: Function;
    error: Function;
  }> */
  
  const start = Date.now();
  
  app.context.success // 具有正确类型提示 (property) success: Function
  ctx.success // 具有正确类型提示 (property) success: Function
  
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

为上面👆 logger 中 ctx 指定类型声明

```typescript
app.use(async (ctx: Koa.context, next: Koa.Next) => {
	// ctx 类型为 (parameter) ctx: Koa.Context

  const start = Date.now();
  
  app.context.success // 具有正确类型提示 (property) success: Function
  ctx.success // 类型丢失
  
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

发生上面的问题原因在于，`app.use` 中具有类型推断，当不手动设置 `ctx` 类型时，其推断正是我们想要的

```typescript
// 就是这个东东
Koa.ParameterizedContext<{}, {
  success: Function;
  error: Function;
}>
```

当手动设置后变为

```typescript
Koa.context // 其类型声明中是不具备 success、error 这两个类型的
```

那么解决方案来了，`ctx` 的类型如果都是下面这个，是不是就对了

```typescript
Koa.ParameterizedContext<{}, {
  success: Function;
  error: Function;
}>
  
// logger
app.use(async (ctx: Koa.ParameterizedContext<{}, {
  success: Function;
  error: Function;
}>, next) => {
  const start = Date.now();
  
  app.context.success // 具有正确类型提示 (property) success: Function
  ctx.success // 具有正确类型提示 (property) success: Function
  
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

这样可能引出另一个问题

`ctx` 有很多地方在使用，那么每个 `ctx` 的类型每次都要 **这么声明一遍** 或者 定义一个全局的类型来导入使用（每次导入也难受）

那能不能通过 `Koa` 声明合并的方式，为 `Context` 全局添加 `success` 、`error` 类型声明

于是有了实验二

#### 实验二

打开 `node_modules/@types/koa/index.d.ts`，大致浏览会看到这么个东西

```typescript
type DefaultStateExtends = any;
/**
 * This interface can be augmented by users to add types to Koa's default state
 */
interface DefaultState extends DefaultStateExtends {}

type DefaultContextExtends = {};
/**
 * This interface can be augmented by users to add types to Koa's default context
 */
interface DefaultContext extends DefaultContextExtends {
  /**
   * Custom properties.
   */
  [key: string]: any;
}
```

**重点**

1. DefaultState 可以扩展 `state`
2. DefaultContext 可以扩展 `context`

来看看怎么进行声明合并，`src/types/index.ts`

```typescript
declare module 'koa' {
  interface DefaultState {
    stateProperty: boolean;
  }

  interface DefaultContext {
    success: TSuccess;
    error: TError;
  }
}
```

再来看看 `app.ts`

```typescript
// logger ctx 类型写或者不写，结果都是正确的
app.use(async (ctx: Koa.Context, next) => {
  const start = Date.now();
  
  app.context.success // 具有正确类型提示 (property) success: Function
  ctx.success // 具有正确类型提示 (property) success: Function
  
  await next();
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});
```

此时已经达到我买的目的了，反过来看一下我们将 类型合并声明 放到了什么地方，OK，here， `src/types/index.ts`

那为什么不放到 `src/global.d.ts` 中呢，测试过就会发现，如果放到这里面，我们的类型合并声明就会失败，失败方是 `@types/koa` 中提供的类型声明。原因就在于，我们导入的 `koa` 的类型声明被 ``src/global.d.ts` 中的声明给拦截了，导致并未读取 `@types/koa` 中提供的类型声明

问题到此就基本解决了

> 测试项目 [koa2-ts](https://github.com/localSummer/koa2-ts)

### `参考文档`

1. [https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/koa/test/index.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/koa/test/index.ts)
2. [https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/koa/test/default.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/koa/test/default.ts)

















