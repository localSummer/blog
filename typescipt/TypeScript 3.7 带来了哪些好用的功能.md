### TypeScript 3.7 带来了哪些好用的功能
@[toc]
#### Optional Chining（可选链）

`Optional Chining` 核心点在于它允许我们写出在遇到 `null` 或者 `undefined` 时，能立即停止运行的代码。

其语法为 `?.` ，可以用该运算符来访问一个对象中的可选属性。

现在看下面的代码

```javascript
let x = foo?.bar.baz();
```

这行代码表示的意思是：当 `foo` 存在（即该变量不为 *null 或 undefined*），那么表达式 `foo?.bar.baz()` 执行完成；反之，当 `foo` 是 `null` 或 `undefined` 时，TypeScript会立即停止运行，并返回 `undefined`。

用代码表达上面这句话的意思即为：

```javascript
let x = (foo === null || foo === undefined) ? undefined : foo.bar.baz();
```

**注意：**我们只对 `foo` 做了可选链的检测，那么当 `bar` 是 `null` 或 `undefined` 时，代码仍然是会报错的，同理 `baz` 也一样 。

`Optional Chining` 带来的好处是显而易见的，它会使得代码更加的简练并轻松替换掉一写前置判断条件。

比如：

```javascript
// 之前的前置判断大多是这么写的
if (foo && foo.bar && foo.bar.baz) {
    // ...
}

// 使用 Optional Chining 后
if (foo?.bar?.baz) {
    // ...
}
```

在大多数情况下 `&&` 和 `?.` 是可以替换的，但是两者之间的行为略有不同，不同点如下：

- `&&` 专用于假值（”falsy“），如 `false | "" | 0 | null | undefined`

- ?.` 则是针对于 `null | undefined` 的判断



Optional Chining 除了可用于**可选属性**的访问，还可用于**可选元素**的访问，以及**可选调用**的访问。

```javascript
// 可选元素->获取数组的第一个元素
function getFirstElement<T>(arr?: T[]) {
    return arr?.[0];
}

// 可选调用->函数调用
function wrapFn(fn?: (msg: string) => void) {
  fn?.('test');
}
```

**注意：Optional Chining 的「短路运算」行为被局限在属性的访问、函数的调用以及元素的访问，它不会沿伸到后续的表达式中**

看下面的代码

```javascript
let result = foo?.bar / someComputation()
```

`Optional Chining` 不会阻止除法运算或者 `someComputation()` 调用。

----

#### Nullish Coalescing

运算符 `??` 有点类似于默认值得处理方式，当处理 `null` 或 `undefined` 时，便会使用默认值。

```javascript
let x = foo ?? bar();
// 等价于
let x = (foo !== null && foo !== undefined) ? foo : bar();
```

这行代码表示：当 `foo` 值存在时，`foo` 会被使用，但是当它是 `null` 或 `undefined`，它会执行 `bar()`。

在 `??` 运算符出现之前，我们对默认值处理是使用 `||`

比如:

```javascript
// window.volume 为 0 时，则取值 0.5，而在使用 ?? 时 则取值为 0
let volume = window.volume || 0.5; 
```

与 `Optional Chining` 类似，运算符 `??` 与 `||` 在默认值的取值上区别在于

- `||` 判断范围为（”falsy“），如 `false | "" | 0 | null | undefined`
- `??`  则是针对于 `null | undefined` 的判断

----

#### `--declaration` and `--allowJs`

`--declaration` 选项可以让我们从 TypeScript 源文件（如 `.ts`、`.tsx` 文件）中生成 `.d.ts` 声明文件。

`--allowJs` 选项允许TypeScript解析 `.js` 、`jsx` 文件。

然而在 3.7 之前的版本中，两者并不能一起使用，因为 `--declaration` 选项会混合 TypeScript 和 JavaScript 输入文件。

在 TypeScript 3.7 中，两者可以一起使用，用户通过在 JavaScript 库中写的 JSDoc 注释，能帮助 TypeScript 用户进行类型检查。

```javascript
const assert = require('assert');

module.exports.blurImage = blurImage;

/**
 * Produces a blurred image from an input buffer.
 *
 * @param input {Uint8Array}
 * @param width {number}
 * @param height {number}
 */
function blurImage(input, width, height) {
  const numPixels = width * height * 4;
  assert(input.length === numPixels);
  const result = new Uint8Array(numPixels);

  // TODO

  return result;
}
```

将会产生如下的 `.d.ts`

```javascript
/**
 * Produces a blurred image from an input buffer.
 *
 * @param input {Uint8Array}
 * @param width {number}
 * @param height {number}
 */
export function blurImage(input: Uint8Array, width: number, height: number): Uint8Array;
```

以上为个人认为在 TypeScript 3.7 中需要重点关注的内容，其他具体内容见 [TypeScript 3.7 更新概览](http://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html)。

----

### TypeScript 3.8 中值得关注的功能

1. ECMAScript 私有字段

   - ```javascript
     class Person {
         #name: string
     
         constructor(name: string) {
             this.#name = name;
         }
     
         greet() {
             console.log(`Hello, my name is ${this.#name}!`);
         }
     }
     
     let jeremy = new Person("Jeremy Bearimy");
     
     jeremy.#name
     //     ~~~~~
     // Property '#name' is not accessible outside class 'Person'
     // because it has a private identifier.
     ```

2. `export * as ns` 语法

   - ```javascript
     export * as utilities from "./utilities.js";
     ```

   - 上面代码在 ECMAScript 2020 中被支持，TypeScript 3.8 实现了此语法

3. `Top-Level await`

   - 在当前的 JavaScript 中（以及其他具有相似功能的大多数其他语言），`await` 仅仅只能用于 `async` 函数内部。然而，使用 `top-level await` 时，我们可以在一个模块的顶层使用 `await`。

4. watchOptions 配置监听策略

具体细节见 [TypeScript 3.8 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-3-8-beta/)


