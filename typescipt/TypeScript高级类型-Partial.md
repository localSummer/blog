### TypeScript高级类型-Partial

> 预备知识：
>
> [TypeScript类型系统](https://jkchao.github.io/typescript-book-chinese/typings/overview.html#%E8%81%94%E5%90%88%E7%B1%BB%E5%9E%8B)
> [接口](https://jkchao.github.io/typescript-book-chinese/typings/interfaces.html)
> [泛型](https://jkchao.github.io/typescript-book-chinese/typings/generices.html)



先来看一下  `Partial` 类型的定义
``` typescript
/**
 * Make all properties in T optional
 */
type Partial<T> = {
    [P in keyof T]?: T[P];
};
```

假设我们有一个定义 `user` 的接口，如下
``` typescript
interface IUser {
  name: string
  age: number
  department: string
}
```

经过 `Partial` 类型转化后得到

```typescript
type optional = Partial<IUser>

// optional的结果如下
type optional = {
    name?: string | undefined;
    age?: number | undefined;
    department?: string | undefined;
}
```

那么 `Partial<T>` 是如何实现类型转化的呢？

1. 遍历入参 `T` ，获得每一对 `key, value`

2. 将每一对中的 `key` 变为可选，即添加 `?`

3. 希望得到的是由 `key, value` 组成的新类型

   

以上对应到 `TypeScript` 中是如何实现的呢？

对照最开始 `Partial` 的类型定义，能够捕捉到以下重要信息

- `keyof` 是干什么的？

- `in` 是干什么的？

- `[P in keyof T]` 中括号是干什么的？

- `?` 是将该属性变为可选属性

- `T[P]` 是干什么的？

  

**keyof**

`keyof`，即 `索引类型查询操作符`，我们可以将 `keyof` 作用于`泛型 T` 上来获取`泛型 T` 上的`所有 public 属性名`构成的 `联合类型`

> 注意："public、protected、private"修饰符不可出现在类型成员上

例如：

```typescript
type unionKey = keyof IUser

// unionKey 结果如下，其获得了接口类型 IUser 中的所有属性名组成的联合类型
type unionKey = "name" | "age" | "department"
```



**in**

我们需要遍历 `IUser` ，这时候 `映射类型`就可以用上了，其语法为 `[P in Keys]`

- P：类型变量，依次绑定到每个属性上，对应每个属性名的类型
- Keys：字符串字面量构成的联合类型，表示一组属性名（的类型），可以联想到上文 `keyof` 的作用



上述问题中 `[P in keyof T]` 中括号是干什么的？这里也就很清晰了。



**T[P]**

我们可以通过 `keyof` 查询索引类型的属性名，那么如何获取属性名对应的属性值类型呢？

这里就用到了 `索引访问操作符`，与 JavaScript 种访问属性值的操作类似，访问类型的操作符也是通过 `[]` 来访问的，即 `T[P]`，其中”中括号“中的 `P` 与 `[P in keyof T]` 中的 `P` 相对应。

例如

```typescript
type unionKey = keyof IUser // "name" | "age" | "department"

type values = IUser[unionKey] // string | number 属性值类型组成的联合类型
```



最后我们希望得到的是由多个 `key, value` 组成的新类型，故而在 `[P in keyof T]?: T[P];`  外部包上一层大括号。

到此我们解决遇到的所有问题，只需要逐步代入到 `Partial` 类型定义中即可。



-----

总结：

1. 针对高级类型的编写，我们可以尝试将其以 `JavaScript` 的代码方式表述出来，然后使用 `TypeScript` 对其进行实现。
2. 在每一个小步骤中遇到不懂的，可以结合最后的结果进行比对（比如本文中 `Partial` 的类型定义），发现问题点在哪里，然后针对性查证并解决。

 










