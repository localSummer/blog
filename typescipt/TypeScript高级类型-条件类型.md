### TypeScript高级类型-条件类型
[TOC]
> 预备知识：
>
> [泛型](http://www.typescriptlang.org/docs/handbook/generics.html)
>
> [高级类型](http://www.typescriptlang.org/docs/handbook/advanced-types.html#conditional-types)

#### 为什么需要条件类型？

在TypeScript使用过程中，我们一般会直接指定具体类型

比如：

```typescript
let str: string = 'test';
```

然而，我们在编写代码的过程中，会遇到不能明确指定其具体类型的情况

比如：

```typescript
declare function f<T extends boolean>(x: T): T extends true ? string : number;

// Type is 'string | number'
let x = f(Math.random() < 0.5)
// Type is 'number'
let y = f(false)
// Type is 'string'
let z = f(true)
```

在编写函数 `f` 时，只知道返回值的范围，但不知道其具体类型，其具体类型需要等到函数执行时进行确定，换句话说，只有类型系统中给出 `充足的条件` 之后,它才会根据条件推断出类型结果。

----

#### 条件类型是什么及其使用

先看一下条件类型是什么

```typescript
T extends U ? X : Y
```

上面的类型表示：若 `T` 能够分配（赋值）给 `U`，那么类型是 `X`，否则为 `Y`，有点类似于JavaScript中的三元条件运算符。

上文说到只有类型系统中给出 `充足的条件` 之后,它才会根据条件推断出类型结果，如果判断条件不足，则会得到第三种结果，即 `推迟` 条件判断，等待充足条件。

例如：

```typescript
interface Foo {
    propA: boolean;
    propB: boolean;
}

declare function f<T>(x: T): T extends Foo ? string : number;

function foo<U>(x: U) {
    // 因为 ”x“ 未知，因此判断条件不足，不能确定条件分支，推迟条件判断直到 ”x“ 明确，
  	// 推迟过程中，”a“ 的类型为分支条件类型组成的联合类型，
    // string | number
    let a = f(x);

    // 这么做是完全可以的
    let b: string | number = a;
}
```

条件类型经常用于TypeScript类型编程当中，在高级类型编写时会经常见到它的身影

比如：

```typescript
/**
 * Exclude from T those types that are assignable to U
 */
type Exclude<T, U> = T extends U ? never : T;
```

后面文章会逐渐讲解到！

----

#### 分布式条件类型

什么样的条件类型称为分布式条件类型呢？

答案是：条件类型里待检查的类型必须是裸类型（`naked type parameter`）

到目前为止，我们可以捕获到两个疑问点

1. 什么类型是裸类型？
2. 分布式如何理解？

**先看什么类型是裸类型**

裸类型是指类型参数没有被包装在其他类型里,比如没有被数组、元组、函数、Promise等等包裹，简而言之裸类型就是未经过任何其他类型修饰或包装的类型。

比如：

```typescript
// 裸类型参数,没有被任何其他类型包裹，即T
type NakedType<T> = T extends boolean ? "YES" : "NO"
// 类型参数被包裹的在元组内，即[T]
type WrappedType<T> = [T] extends [boolean] ? "YES" : "NO";
```

**分布式如何理解**

分布式条件类型在实例化时会自动分发成联合类型

什么意思呢？

例如，`T extends U ? X : Y`使用类型参数`A | B | C` 实例化 `T` 解析为 `(A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y)`

结合 `乘法分配律` 理解一下！



接下来结合具体实例我们来看一下`分布式条件类型` 与 `不含有分布式特性的条件类型`

```typescript
// 含有分布式特性的，待检查类型必须为”裸类型“
type Distributed = NakedType<number | boolean> //  = NakedType<number> | NakedType<boolean> =  "NO" | "YES"（结合一下乘法分配律便于理解与记忆哦~）

// 不含有分布式特性的，待检查的类型为包装或修饰过的类型
type NotDistributed = WrappedType<number | boolean > // "NO"
```

搞明白了分布式条件类型，我们编写这样一个类型工具 `NonNullable<T>` ，即从类型 `T` 中排除 `null 和 undefined` ，我们期待的结果如下：

```typescript
type a = NonNullable<string | number | undefined | null> // 得到type a = string | number
```

借助条件类型可以很容易写出来

```typescript
type NonNullable<T> = T extends null | undefined ? never : T
```

> 注意：`never` 类型表示不会是任何值，即什么都没有

----

#### 条件类型与映射类型

条件类型与映射类型的结合经常会被作为考点，常见题型多为设计类型工具方法

> 映射类型相关内容见 [TypeScript高级类型-Partial分析](https://github.com/localSummer/blog/blob/master/typescipt/TypeScript%E9%AB%98%E7%BA%A7%E7%B1%BB%E5%9E%8B-Partial.md)

接下来设计一个这样的类型工具`NonFunctionKeys<T>` ，通过使用 `NonFunctionKeys<T>`  得到对象类型 `T` 中非函数的属性名组成的联合类型

```typescript
type MixedProps = { name: string; setName: (name: string) => void };

// Expect: "name"
type Keys = NonFunctionKeys<MixedProps>;
```

那么如何设计呢？

1. 使用JavaScript表述出来
   1. 遍历 `MixedProps` 的 `key，value`，
   2. 找出每个 `value` 是否是函数类型，是则排除掉，否则保留（ts中保留的为对应的key，这样方便使用索引访问操作符取出）
   3. 取出所有保留的 `key`
2. 使用TypeScript进行实现

```typescript
type NonFunctionKeys<T> = {
  [P in keyof T]: T[P] extends Function ? never : P // 这里保留的value被替换为了key
}[keyof T]
```

> 里面设计到的 `keyof`、 `in`、 `T[P]` 可参考  [TypeScript高级类型-Partial分析](https://github.com/localSummer/blog/blob/master/typescipt/TypeScript%E9%AB%98%E7%BA%A7%E7%B1%BB%E5%9E%8B-Partial.md)

----

#### 条件类型中的类型推断

在 `extends` 条件类型的子句中，现在可以含有 `infer` 引入要推断的类型变量的声明，可以在条件类型的真实分支中引用此类推断的类型变量，另外，`infer` 同一类型变量可能有多个位置。

简单而言 infer 关键字就是声明一个类型变量，当类型系统给足条件的时候类型就会被推断出来。

例如，以下代码提取函数类型的返回类型：

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any;
```

下面的示例演示在**协变位置**上**同一类型变量的多个候选类型**将会被推断为**联合类型**：

```typescript
type Foo<T> = T extends { a: infer U, b: infer U } ? U : never;
type t1 = Foo<{ a: string, b: string }>;  // string
type t2 = Foo<{ a: string, b: number }>;  // string | number
```

同样在**逆变位置**上**同一类型变量的多个候选类型**将会被推断为**交叉类型**：

```typescript
type Bar<T> = T extends { a: (x: infer U) => void, b: (x: infer U) => void } ? U : never;
type t1 = Bar<{ a: (x: string) => void, b: (x: string) => void }>;  // string
type t2 = Bar<{ a: (x: string) => void, b: (x: number) => void }>;  // string & number
```

> [协变与逆变](https://jkchao.github.io/typescript-book-chinese/tips/covarianceAndContravariance.html)

> 注意：`infer` 对于常规类型参数（泛型约束），不能在约束子句中使用 `infer` 声明

比如：

```typescript
type ReturnType<T extends (...args: any[]) => infer R> = R;  // Error, not supported
```

下面我们实现一下 `ConstructorParameters<T>` ，用于提取构造函数中参数类型

```typescript
class TestClass {
    constructor(public name: string, public age: number) {}
}
// 期待结果如下
type paramsType = ConstructorParameters<typeof TestClass> // [string, number]
```

1. 拿到 `TestClass` 构造函数签名
2. 使用 infer 推断构造函数的入参类型

```typescript
type ConstructorParameters<T extends new (...args: any[]) => any> = 
  T extends new (...args: infer P) => any ? P : never;
```

重点：

- `new (...args: any[]` 指构造函数

- `infer P` 代表待推断的构造函数参数,如果接受的类型 `T` 是一个构造函数，那么返回构造函数的参数类型 `P`，否则什么也不返回，即 `never` 类型

**infer 的应用也是非常广泛的**

比如：

tuple 转 union，`[string, number] -> string | number`

```typescript
type ElementOf<T> = T extends Array<infer E> ? E : never;

type TTuple = [string, number];

type ToUnion = ElementOf<TTuple>; // string | number
```

union 转 intersection，如：`string | number -> string & number`

```typescript
type UnionToIntersection<U> = (U extends any ? (k: U) => void : never) extends ((k: infer I) => void) ? I : never;

type Result = UnionToIntersection<string | number>; // 注意：string & number 就是 never
```

重点：

- `U extends any` 是具有分布式有条件类型特性，因为待检查类型 `U` 为裸类型

- `(U extends any ? (k: U) => void : never) extends ((k: infer I) => void)` 最后一个 extends 前面作为待检查类型，因为被函数包装，因此不具有分布式有条件类型特性

  - ```typescript
    type UnionToIntersection<U> = ((k: string) => void | (k: number) => void) extends ((k: infer I) => void) ? I : never;
    ```

  - 根据 `逆变特性` ，推断出的 `I` 应该具备 `string 和 number` 的类型，故为交叉类型 `string & number`，而该交叉类型在 `vscode` 中表现为 `never`



----



**本文梳理**：

1. 需要明确条件类型的表达方式，即 `T extends U ? X : Y`
2. 需要明确什么是分布式有条件类型，以及判断为分布式有条件类型的前提条件
   1. 分布式
   2. 裸类型
3. infer关键字，明确它的使用范围及其作用