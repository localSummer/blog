### TypeScript高级类型-内置实用工具类型

[TOC]

#### 预备知识

1. [TypeScript高级类型-Partial](https://blog.csdn.net/roamingcode/article/details/104111165)
2. [TypeScript高级类型-条件类型](https://blog.csdn.net/roamingcode/article/details/104114372)（重要前置知识）
3. [TypeScript高级类型-实用技巧](https://blog.csdn.net/roamingcode/article/details/104134351)

#### `Partial<T>`

将泛型 `T` 中的所有属性转化为可选属性

```typescript
/**
 * Make all properties in T optional
 */
type Partial<T> = {
    [P in keyof T]?: T[P];
};

interface IUser {
  name: string;
  age: number;
}

type t = Partial<IUser>;
/* type t = {
	name?: string;
	age?: number
} */
```

#### `Required<T>`

将泛型 `T` 中的所有属性转化为必选属性

```typescript
/**
 * Make all properties in T required
 */
type Required<T> = {
    [P in keyof T]-?: T[P];
};

interface IUser {
  name?: string;
  age?: number;
}

type t = Required<IUser>;
/* type t = {
	name: string;
	age: number
} */
```

#### `Readonly<T>`

将泛型 `T` 中的所有属性转化为只读属性

```typescript
/**
 * Make all properties in T readonly
 */
type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};

interface IUser {
  name: string;
  age: number;
}

type t = Readonly<IUser>;
/* type t = {
	readonly name: string;
	readonly age: number
} */
```

#### `Pick<T, K extends keyof T>`

从泛型 `T` 中检出指定的属性并组成一个新的对象类型

```typescript
/**
 * From T, pick a set of properties whose keys are in the union K
 */
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};

interface IUser {
  name: string;
  age: number;
  address: string;
}

type t = Pick<IUser, 'name' | 'age'>;
/* type t = {
	name: string;
	age: number
} */
```

#### `Record<K extends keyof any, T>`

Record 允许从 Union 类型中创建新类型，Union 类型中的值用作新类型的属性。

经常用于接口 `response` 类型声明

```typescript
/**
 * Construct a type with a set of properties K of type T
 */
type Record<K extends keyof any, T> = {
    [P in K]: T;
};

type Car = 'Audi' | 'BMW' | 'MercedesBenz'
type CarList = Record<Car, {age: number}>

const cars: CarList = {
    Audi: { age: 119 },
    BMW: { age: 113 },
    MercedesBenz: { age: 133 },
}
```

#### `Exclude<T, U>`

从泛型 `T` 中排除可以赋值给泛型 `U` 的类型

```typescript
/**
 * Exclude from T those types that are assignable to U
 */
type Exclude<T, U> = T extends U ? never : T;

type a = 1 | 2 | 3;

type t = Exclude<a, 1 | 2>;
/* type t = 3 */
```

#### `Extract<T, U>`

从泛型 `T` 中提取可以赋值给泛型 `U` 的类型

```typescript
/**
 * Extract from T those types that are assignable to U
 */
type Extract<T, U> = T extends U ? T : never;

type a = 1 | 2 | 3;

type t = Extract<a, 1 | 2>;
/* type t = 1 | 2 */
```

#### `Omit<T, K extends keyof any>`

从泛型 `T` 中提取出不在泛型 `K` 中的属性类型，并组成一个新的对象类型

```typescript
/**
 * Construct a type with the properties of T except for those in type K.
 */
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

interface IUser {
  name: string;
  age: number;
  address: string;
}

type t = Omit<IUser, 'name' | 'age'>;
/* type t = {
	address: string;
} */
```

#### `NonNullable<T>`

从泛型 `T` 中排除掉 `null` 和 `undefined`

```typescript
/**
 * Exclude null and undefined from T
 */
type NonNullable<T> = T extends null | undefined ? never : T;

type t = NonNullable<'name' | undefined | null>;
/* type t = 'name' */
```

#### `Parameters<T extends (...args: any) => any>`

以元组的方式获得函数的入参类型

```typescript
/**
 * Obtain the parameters of a function type in a tuple
 */
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;

type t = Parameters<(name: string) => any>; // type t = [string]
type t2 = Parameters<((name: string) => any)  | ((age: number) => any)>; // type t2 = [string] | [number]
```

#### `ConstructorParameters<T extends new (...args: any) => any>`

以元组的方式获得构造函数的入参类型

```typescript
/**
 * Obtain the parameters of a constructor function type in a tuple
 */
type ConstructorParameters<T extends new (...args: any) => any> = T extends new (...args: infer P) => any ? P : never;

type t = ConstructorParameters<(new (name: string) => any)  | (new (age: number) => any)>;
// type t = [string] | [number]
```

#### `ReturnType<T extends (...args: any) => any>`

获得函数返回值的类型

```typescript
/**
 * Obtain the return type of a function type
 */
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;

type t = ReturnType<(name: string) => string | number>
// type t = string | number
```

#### `InstanceType<T extends new (...args: any) => any>`

获得构造函数返回值的类型

```typescript
/**
 * Obtain the return type of a constructor function type
 */
type InstanceType<T extends new (...args: any) => any> = T extends new (...args: any) => infer R ? R : any;

type t = InstanceType<new (name: string) => {name: string, age: number}>
/* 
type h = {
    name: string;
    age: number;
}
*/
```

#### 总结

1. 重点理解这些内置的工具类型的定义
2. 能够以此为”引“写一些工具类型并运用到项目中去










