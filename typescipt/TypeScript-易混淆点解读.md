### TypeScript-易混淆点解读

[TOC]

#### 字面量类型

字面量是JavaScript本身提供的一个准确变量，其主要分为字符串字面量类型、数字字面量类型、真值字面量类型、枚举字面量类型、大整数字面量类型。

```typescript
// 字符串字面量类型
let foo: 'Hello';
foo = 'Bar'; // Error: 'bar' 不能赋值给类型 'Hello'

// 数字字面量类型组成的联合类型
type OneToFive = 1 | 2 | 3 | 4 | 5;

// 真值字面量类型组成的联合类型
type Bools = true | false;
```

**字面量类型的要和实际的值的字面量一一对应**，否则就会报错。

字面量类型本身并不是很实用，但是可以在一个联合类型中组合创建一个强大的（实用的）抽象，比如：

```typescript
type CardinalDirection = 'North' | 'East' | 'South' | 'West';

function move(distance: number, direction: CardinalDirection) {
  // ...
}

move(1, 'North'); // ok
move(1, 'Nurth'); // Error
```

#### 类型字面量

类型字面量与JavaScript对象字面量语法相近

```typescript
type IUser = {
  name: string;
  age: number;
  eat(food: string): void;
}
```

这个结构我们使用 `interface` 同样可以实现，那么在一定程度上 `类型字面量` 这种类型别名是可以替代接口的。

两者的区别我们放到下文中说明。

#### 可辨识联合类型

当类中含有字面量成员时，我们可以用该类的属性来辨识联合类型。

```typescript
interface Square {
  kind: 'square'; // 字面量类型
  size: number;
}

interface Rectangle {
  kind: 'rectangle'; // 字面量类型
  width: number;
  height: number;
}

type Shape = Square | Rectangle;

```

如果你使用类型保护风格的检查（`==`、`===`、`!=`、`!==`）或者使用具有判断性的属性（在这里是 `kind`），TypeScript 将会认为你会使用的对象类型一定是拥有特殊字面量的，并且它会为你自动把类型范围变小

```typescript
function area(s: Shape) {
  if (s.kind === 'square') {
    // 现在 TypeScript 知道 s 的类型是 Square
    // 所以你现在能安全使用它
    return s.size * s.size;
  } else {
    // 不是一个 square ？因此 TypeScript 将会推算出 s 一定是 Rectangle
    return s.width * s.height;
  }
}
```

其实看到这里我们很容易想到 redux 中 reducer 的使用

```typescript
import { createStore } from 'redux';

// action中的type为字符串字面量类型
type Action =
  | {
      type: 'INCREMENT';
    }
  | {
      type: 'DECREMENT';
    };
function counter(state = 0, action: Action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1;
    case 'DECREMENT':
      return state - 1;
    default:
      return state;
  }
}

// Create a Redux store holding the state of your app.
// Its API is { subscribe, dispatch, getState }.
let store = createStore(counter);

// You can use subscribe() to update the UI in response to state changes.
// Normally you'd use a view binding library (e.g. React Redux) rather than subscribe() directly.
// However it can also be handy to persist the current state in the localStorage.

store.subscribe(() => console.log(store.getState()));

// The only way to mutate the internal state is to dispatch an action.
// The actions can be serialized, logged or stored and later replayed.
store.dispatch({ type: 'INCREMENT' });
// 1
store.dispatch({ type: 'INCREMENT' });
// 2
store.dispatch({ type: 'DECREMENT' });
// 1
```

#### 类型别名与接口的异同点

类型别名的语法如下：

```typescript
type SomeName = someValidTypeAnnotation
```

##### 相同点

**1. 都可以描述一个对象或者函数的形状**

```typescript
interface IUser {
 name: string
 age: number
}
 
interface ISetUser {
 (name: string, age: number): void;
}

type TUser = {
 name: string
 age: number
};
 
type TSetUser = (name: string, age: number): void;
```

**2. 都允许拓展（extends）**

interface 和 type 都可以拓展，并且两者并不是相互独立的，也就是说 interface 可以 extends type, type 也可以 extends interface，但是两者的语法是不一样的。

```typescript
// interface extends interface
interface IName { 
 name: string; 
}
interface IUser extends IName { 
 age: number; 
}

// type extends type
type TName = { 
 name: string; 
}
type TUser = TName & { age: number };

// interface extends type
type TName = { 
 name: string; 
}
interface IUser extends TName { 
 age: number; 
}

// type extends interface
interface IName { 
 name: string; 
}
type TUser = IName & { 
 age: number; 
}
```

##### 不同点

**1. type 可以而 interface 不行**

type 可以声明基本类型别名，联合类型，元组等类型

```typescript
type TName = string
type TText = string | { text: string };
type TCoordinates = [number, number];
```

type 语句中还可以使用 typeof 获取实例的类型进行赋值

```typescript
// 当你想获取一个变量的类型时，使用 typeof
let fn = (str: string) => str;
type TFn = typeof fn; // type TFn = (str: string) => string
```

**2. interface 可以而 type 不行**

interface 能够声明合并

```typescript
interface IUser {
 name: string
 age: number
}
 
interface IUser {
 sex: string
}
 
/*
interface IUser {
 name: string
 age: number
 sex: string 
}
*/
```

**建议：可以使用 `interface` 时则使用 `interface`，否则使用 `type`**