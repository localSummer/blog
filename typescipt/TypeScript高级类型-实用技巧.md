### TypeScript高级类型-实用技巧

[TOC]

#### 预备知识

1. [TypeScript高级类型-Partial](https://blog.csdn.net/roamingcode/article/details/104111165)
2. [TypeScript高级类型-条件类型](https://blog.csdn.net/roamingcode/article/details/104114372)

#### 类型递归

在 TypeScript 中有这样一个内置类型工具 `Required<T>`，它可以将对象类型 `T` 上的所有 `可选属性` 转化为 `必填属性`。

先看一下 `Required<T>` 是如何定义的

```javascript
/**
 * Make all properties in T required
 */
type Required<T> = {
    [P in keyof T]-?: T[P];
};
```

类型转化过程如下：

```javascript
interface IUser {
  name: string;
  age: number;
  sex?: string;
  department?: string;
  address?: string;
}
  
type requiredUser = Required<IUser>
/* type requiredUser = {
  name: string;
  age: number;
  sex: string;
  department: string;
  address: string;
} */
```

经过 `Required<T>` 处理后，可以看到类型 `requiredUser` 里面的所有属性都被转为了必选属性。



现在假如我们有如下一个具有嵌套的类型，再看一下转化结果

```javascript
interface ICompany {
  id: string;
  companyName: string;
  companyAddress?: string;
}

interface IUser {
  name: string;
  age: number;
  sex?: string;
  department?: string;
  address?: string;
  company: ICompany; // 嵌套类型中包含可选属性
}


type requiredUser = Required<IUser>
/* type requiredUser = {
    name: string;
    age: number;
    sex: string;
    department: string;
    address: string;
    company: ICompany; // 嵌套类型中的可选类型未被转为必选属性
} */
```

转化结果发现嵌套类型 `ICompany` 中的可选属性并未被转化为必选属性，这是因为 `Required<T>` 只被用于当前类型 `T` 的转化，而对于内部嵌套的类型并不做处理。

**这里想处理深层嵌套属性，就必须用到类型递归**

结合 JavaScript 的处理方式，我们可以得到 TypeScript 的处理方式

```javascript
type DeepRequired<T> = {
  [P in keyof T]-?: T[P] extends object ? DeepRequired<T[P]> : T[P]
};
```

判断属性值类型是否为对象类型，是则递归类型转化操作。

看一下转化结果

```javascript
type requiredUser = DeepRequired<IUser>
/* type requiredUser = {
    name: string;
    age: number;
    sex: string;
    department: string;
    address: string;
    company: DeepRequired<ICompany>;
} */
```

**注意：**

如果 `IUser` 中的 `company` 为`可选属性`，那么经过 `DeepRequired<IUser>` 转化后，依然为 `company: ICompany; // 嵌套类型中的可选类型未被转为必选属性`。

经过排查不难发现，当 ``company`` 为可选属性，那么其对应的类型为 `ICompany | undefined`，此时可以得到  `ICompany | undefined extends object`，并不满足赋值条件，故而得到 `false` 分支，即（ICompany）。

#### 特殊关键字

其中 `+`、 `-` 这两个关键字用于映射类型中给属性添加修饰符，比如我们再上文中 `Required<T>` 的定义

```javascript
type Required<T> = {
    [P in keyof T]-?: T[P];
};

type RemoveReadonly<T> = {
  -readonly [P in keyof T]: T[P]
}
```

用到了 `-?`，它代表将可选属性变为必选，`-readonly` 代表将只读属性变为非只读。

#### 注释

我们可以使用 `JSDoc` 的方式来添加注释，借助于 IDE 可以给我们提供更友好的提示

```javascript
enum EStatus {
  /**
   * 成功状态
   */
  Success,
  /**
   * 失败状态
   */
  Error
}
// 在调用对应状态的时候，会给出友好的文字提示
```

#### is 关键字

`is` 关键字经常用于依赖布尔值的判断来缩小参数的类型范围

比如：

```javascript
function isString(test: any): test is string{
    return typeof test === 'string';
}

function getStringLength(foo: number | string){
    if(isString(foo)){
        console.log(foo.length); // string function
    }
}
```

`getStringLength` 在执行时，当 `isString` 返回 `true` 时，可以明确知道 `foo ` 是字符串类型，故而调用 `length` 属性正常运行。

然而用下面这种方式则会报错

```javascript
function isString(test: any): boolean{
    return typeof test === 'string';
}
```

经过该函数的判断并不能明确 `foo` 的类型，故而调用 `length` 报错。

#### 泛型约束

在使用泛型的过程中，我们的泛型几乎可以是任何类型

```javascript
/**
 * Make all properties in T readonly
 */
type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};
```

如果我们可以明确传入的泛型是属于哪一类，那么应用泛型约束将会使得代码有更好可维护性。

比如，我们知道上面代码中的泛型 `T` 属于 `object` 类型，那么对其添加泛型约束如下：

```javascript
type ObjectReadonly<T extends object> = {
    readonly [P in keyof T]: T[P];
};
```

这样当我们在使用 工具类型 `ObjectReadonly<T>` 时，将会限制我们只能传入 `object` 类型。












