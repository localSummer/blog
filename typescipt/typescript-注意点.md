### class
1. 类是具有两个类型的：静态部分的类型和实例的类型，当一个类实现一个接口时，只对其实例部分进行类型检查。`constructor` 存在于类的静态部分，所以不在检查范围内。

2. 抽象类做为其他派生类的基类使用，他们一般不会直接被实例化，不同于接口，抽象类可以包含成员的实现细节。`abstract` 关键字是用于定义抽象类和在抽象类内部定义抽象方法。

### 泛型
使用泛型可以创建可重用的组件，一个组件可以支持多种类型的数据。
1. 泛型常用 T 表示，可整体使用同时可以部分使用
```typescript
function getFirstEleFromArr<T>(arr: T[]): T {
  return arr[0]
}
```
2. 泛型函数、泛型接口、泛型类
3. 泛型约束

### 高级类型
1. 交叉类型 - 是将多个类型合并为一个类型。我们可以把现有的多种类型叠加到一起成为一种类型
```typescript
  function extend<T extends Object, U extends Object>(first: T, second: U): T & U {
    let result = <T & U>{}

    for (let key in first) {
      (result as any)[key] = first[key]
    }

    for (let key in second) {
      if (!result.hasOwnProperty(key)) {
        (result as any)[key] = second[key]
      }
    }

    return result
  }
```

2. 联合类型 - 表示一个值可以是几种类型之一，如果一个值是联合类型，我们只能访问此联合类型的所有类型里共有的成员

3. 多态的this类型 - 表示某个包含类或接口的子类型
```typescript
// typescript中支持链式调用
class MyCalculator {
  public constructor(public number = 0) {}

  public add(n: number): this {
    this.number += n
    return this
  }

  public mul(n: number): this {
    this.number *= n
    return this
  }
}

let n = new MyCalculator(2).add(1).mul(5)
```

4. 索引类型查询操作符 - keyof。对于任何类型 T ，keyof T 的结果为 T 上已知的公共属性名的联合，常常配合索引访问操作符（in）一起使用

```typescript
type Partical<T extends Object> {
  [K in keyof T]?: T[K]
}
```

### 命名空间
命名空间主要作用是用来组织代码，以便于在记录他们类型的同时还不用担心与其他对象产生命名冲突

```typescript
declare namespace D3 {
  export interface Selectors {
    select: {
      (selector: string): Selection;
      (element: EventTarget): Selection;
    }
  }

  export interface Event {
    x: number;
    y: number;
  }

  export interface Base extends Selectors {
    event: Event
  }
}

declare var d3: D3.Base
```

### 使用第三方类库 lodash
1. 查找是否存在该类库所对应的类型声明文件 
```javascript
npm info @types/lodash
```
如若不存在则需要我们自身提供对应的类型声明文件，推荐配合 namespace 进行编写

存在便可以直接安装
```javascript
npm i -S @types/lodash
```