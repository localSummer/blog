### TypeScript-工具类型的设计

[TOC]

#### 预备知识

1. [TypeScript高级类型-Partial](https://blog.csdn.net/roamingcode/article/details/104111165)
2. [TypeScript高级类型-条件类型](https://blog.csdn.net/roamingcode/article/details/104114372)（重要前置知识）
3. [TypeScript高级类型-实用技巧](https://blog.csdn.net/roamingcode/article/details/104134351)

#### 尝试解一道面试题

> [原题地址](https://github.com/LeetCode-OpenSource/hire/blob/master/typescript_zh.md)

详细描述可见上面链接，这里只说明一下核心点

我们有一个`EffectModule` 的类，这个类中的方法只可能有两种类型签名，另外这个类中还可能有一些任意的**非函数属性**，如下

```typescript
interface Action<T> {
  payload?: T;
  type: string;
}

// 该签名对应于 delay
asyncMethod<T, U>(input: Promise<T>): Promise<Action<U>>

// 该签名对应于 setMessage
syncMethod<T, U>(action: Action<T>): Action<U>

class EffectModule {
  count = 1;
  message = "hello!";

  delay(input: Promise<number>) {
    return input.then(i => ({
      payload: `hello ${i}!`,
      type: 'delay'
    });
  }

  setMessage(action: Action<Date>) {
    return {
      payload: action.payload!.getMilliseconds(),
      type: "set-message"
    };
  }
}
```

现在有一个叫 `connect` 的函数，它接受 EffectModule 实例，将它变成另一个对象，这个对象上只有**EffectModule 的同名方法**，但是方法的类型签名被改变了，如下：

```typescript
asyncMethod<T, U>(input: Promise<T>): Promise<Action<U>>  
//变成了
asyncMethod<T, U>(input: T): Action<U> 
  
syncMethod<T, U>(action: Action<T>): Action<U>  
//变成了
syncMethod<T, U>(action: T): Action<U>
```

connect 之后的结果是：

```typescript
// 方法同名，函数签名改变
type Connected = {
  delay(input: number): Action<string>
  setMessage(action: Date): Action<number>
}
const effectModule = new EffectModule()
const connected: Connected = connect(effectModule)
```

**这道题的核心点就在于从 `EffectModule` 类型中推导出 `Connected` 的类型。**

先来分析一下解决步骤：

1. 找到 `EffectModule` 中的函数名称
2. 转换函数签名
3. 组合分支形成主干

先来完成第一步，设计一个类型工具用来获取对象类型中的函数属性名

```typescript
// 预备知识中前两个进行过详细的描述，这里就不说明了
type FunctionKeys<T> = {
  [K in keyof T]: T[K] extends Function ? K : never
}[keyof T];
```

第二步中，先把转换前后的方法类型写出来

```typescript
type asyncMethod<T, U> = (input: Promise<T>) => Promise<Action<U>> // 转换前
type asyncMethodConnect<T, U> = (input: T) => Action<U> // 转换后
type syncMethod<T, U> = (action: Action<T>) => Action<U> // 转换前
type syncMethodConnect<T, U> = (action: T) => Action<U> // 转换后
```

那么如何转化呢，就需要用到 `条件类型` 和`推断类型`

```typescript
// T就是EffectModule中方法对应的签名
type EffectModuleMethodsConnect<T> = T extends asyncMethod<infer U, infer V>
	? asyncMethodConnect<U, V>
	: T extends syncMethod<infer U, infer V>
	? syncMethodConnect<U, V>
	: never;
                                                           
```

最后组合得到：

```typescript
type Connect = (module: EffectModule) => {
  [M in FunctionKeys<EffectModule>]: EffectModuleMethodsConnect<EffectModule[M]>
}
```

#### 工具类型分析与设计注意事项

1. 核心知识点的掌握（泛型、索引查询操作符、索引访问符、映射类型、条件类型、分布式有条件类型、条件类型推断，这些预备知识中都已包含，可选择性查看）
2. 读懂题意，明确“输入”、“产出”的对应关系，搞明白我们要做什么
3. 拆解步骤、尝试组合目标结果，根据已知条件推导未知结果
4. 尝试步骤的优化与合并，找出最优解

#### 如何训练工具类型的编写呢？

重视基础知识的掌握、多做练习。

这里我们给出一个工具类型的代码库，大家可以尝试读一下，在读懂的情况下尝试编写一下。

 [utility-types](https://github.com/piotrwitek/utility-types)






