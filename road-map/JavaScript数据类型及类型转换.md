## JavaScript 数据类型

到目前为止为 **8** 种

1. undefined
2. null
3. string
4. number
5. boolean
6. symbol
7. bigint
8. Object
   1. Array
   2. RegExp
   3. Date
   4. Math
   5. Function

其中前7种为基本数据类型，最后一种为引用类型，如下图

![Lark20210108-154509.png](https://s0.lgstatic.com/i/image2/M01/04/F0/CgpVE1_4DdGAJ_EXAAE38RQC0js096.png)

因为各种 JavaScript 的数据类型最后都会在初始化之后放在不同的内存中，因此上面的数据类型大致可以分成两类来进行存储：

1. 基础类型数据存储在栈内存，被引用或拷贝时，会创建一个完全相等的变量
2. 引用类型数据存储在堆内存中，存储的是地址，多个引用指向同一个地址，这里会涉及一个共享的问题

```javascript
let a = {
  name: 'lee',
  age: 18
}

let b = a;
console.log(a.name);  //第一个console输出 lee
b.name = 'son';
console.log(a.name);  //第二个console输出 son
console.log(b.name);  //第三个console 输出 son

```

这里就体现了引用类型的**“共享”**的特性，即这两个值都存在同一块内存中共享，一个发生了改变，另外一个也随之跟着变化。

### 思考题

```javascript
let a = {
  name: 'Julia',
  age: 20
}
function change(o) {
  o.age = 24;
  o = {
    name: 'Kath',
    age: 30
  }
  return o;
}
let b = change(a);
console.log(b.age);    // 第一个console
console.log(a.age);    // 第二个console
```

## 数据类型检测

### 第一种 typeof

```javascript
typeof 1 // 'number'
typeof '1' // 'string'
typeof undefined // 'undefined'
typeof true // 'boolean'
typeof Symbol() // 'symbol'
typeof null // 'object'
typeof [] // 'object'
typeof {} // 'object'
typeof console // 'object'
typeof console.log // 'fuction'
```

虽然 typeof null 会输出 object，但这只是 JS 存在的一个历史 Bug（其二进制表示全为0，而JavaScript中规定，二进制中前三位为0则认为它是对象），不代表 null 就是引用数据类型，并且 null 本身也不是对象。

此外，用 typeof 来判断引用数据类型的话，除了 function 会被判断正确外，其余都会返回 `'object'`

因此常用 `typeof` 来判断基础数据类型，而不用于引用类型的判断

### 第二种 instanceof

```javascript
let Car = function() {}
let benz = new Car()
benz instanceof Car // true
let car = new String('Mercedes Benz')
car instanceof String // true
let str = 'Covid-19'
str instanceof String // false	
```

instanceof 适用于引用类型的判断

#### 自行实现 instanceof

```javascript
const myInstanceof = (left, right) => {
  // 基础类型直接退出
  if (typeof left !== 'object' || left === null) return false
  let proto = Object.getPrototypeOf(left)
  while (true) {
    if (proto === null) return false
    if (proto === right.prototype) {
      return true
    }
    proto = Object.getPrototypeOf(proto)
  }
}

// 验证一下自己实现的myInstanceof是否OK
console.log(myInstanceof(new Number(123), Number)) // true
console.log(myInstanceof(123, Number)) // false

```

typeof 与 intanceof 这两种判断数据类型的方案有什么差异？

1. instanceof 可以准确地判断复杂引用数据类型，但是不能正确判断基础数据类型
2. 而 typeof 也存在弊端，它虽然可以判断基础数据类型（null 除外），但是引用数据类型中，除了 function 类型以外，其他的也无法判断

总之，不管单独用 typeof 还是 instanceof，都不能满足所有场景的需求，而只能通过二者混写的方式来判断。

### 第三种 Object.prototype.toString

toString() 是 Object 的原型方法，调用该方法，可以统一返回格式为 “[object Xxx]” 的字符串，其中 Xxx 就是对象的类型。

对于Object 对象，直接调用 toString() 就能返回 [object Object]；

而对于其他对象，则需要通过 call 来调用，才能返回正确的类型信息。

```javascript
Object.prototype.toString({})       // "[object Object]"
Object.prototype.toString.call({})  // 同上结果，加上call也ok
Object.prototype.toString.call(1)    // "[object Number]"
Object.prototype.toString.call('1')  // "[object String]"
Object.prototype.toString.call(true)  // "[object Boolean]"
Object.prototype.toString.call(function(){})  // "[object Function]"
Object.prototype.toString.call(null)   //"[object Null]"
Object.prototype.toString.call(undefined) //"[object Undefined]"
Object.prototype.toString.call(/123/g)    //"[object RegExp]"
Object.prototype.toString.call(new Date()) //"[object Date]"
Object.prototype.toString.call([])       //"[object Array]"
Object.prototype.toString.call(document)  //"[object HTMLDocument]"
Object.prototype.toString.call(window)   //"[object Window]"
```

在写判断条件的时候一定要注意，使用这个方法最后返回统一字符串格式为 "[object Xxx]" ，而这里字符串里面的 "Xxx" ，**第一个首字母要大写**（注意：使用 typeof 返回的是小写），这里需要多加留意。

#### 通用的数据类型判断方法

```javascript
const getType = (obj) => {
  let type = typeof obj
  // 基础类型直接返回，function类型也返回,返回的类型小写
  if (type !== 'object' || obj === null) return type
  return Object.prototype.toString.call(obj).replace(/^\[object (\S+)\]$/, '$1')
}

getType([])     // "Array" typeof []是object，因此toString返回
getType('123')  // "string" typeof 直接返回
getType(window) // "Window" toString返回
getType(null)   // null，typeof null是object，obj === null 拦截
getType(undefined)   // "undefined" typeof 直接返回
getType()            // "undefined" typeof 直接返回
getType(function(){}) // "function" typeof能判断，因此首字母小写
getType(/123/g)      //"RegExp" toString返回
```

## 数据类型转换

在日常的业务开发中，经常会遇到 JavaScript 数据类型转换问题，有的时候需要我们主动进行强制转换，而有的时候 JavaScript 会进行隐式转换，隐式转换的时候就需要我们多加留心。

```javascript
'123' == 123   // true
'' == null    // false
'' == 0        // true
[] == 0        // true
[] == ''       // true
[] == ![]      // true
null == undefined // true
Number(null)     // 0
Number('')      // 0
parseInt('');    // NaN
{}+10           // 10
let obj = {
    [Symbol.toPrimitive]() {
        return 200;
    },
    valueOf() {
        return 300;
    },
    toString() {
        return 'Hello';
    }
}

console.log(obj + 200); // 400
```

#### 强制类型转换

强制类型转换方式包括 Number()、parseInt()、parseFloat()、toString()、String()、Boolean()，这几种方法都比较类似，通过字面意思可以很容易理解，都是通过自身的方法来进行数据类型的强制转换

**Number() 方法的强制转换规则**

1. 如果是布尔值，true 和 false 分别被转换为 1 和 0；
2. 如果是数字，返回自身；
3. 如果是 null，返回 0；
4. 如果是 undefined，返回 NaN；
5. 如果是字符串，遵循以下规则：
   1. 如果字符串中只包含数字（或者是 0X / 0x 开头的十六进制数字字符串，允许包含正负号），则将其转换为十进制；
   2. 如果字符串中包含有效的浮点格式，将其转换为浮点数值；
   3. 如果是空字符串，将其转换为 0；
   4. 如果不是以上格式的字符串，均返回 NaN；
6. 如果是 Symbol，抛出错误；
7. 如果是对象，并且部署了 [Symbol.toPrimitive] ，那么调用此方法，否则调用对象的 valueOf() 方法，然后依据前面的规则转换返回的值；如果转换的结果是 NaN ，则调用对象的 toString() 方法，再次依照前面的顺序转换返回对应的值（Object 转换规则会在下面细讲）。

```javascript
Number(true);        // 1
Number(false);       // 0
Number('0111');      //111
Number(null);        //0
Number('');          //0
Number('1a');        //NaN
Number(-0X11);       //-17
Number('0X11')       //17
```

**Boolean() 方法的强制转换规则**

这个方法的规则是：除了 undefined、 null、 false、 ''、 0（包括 +0，-0）、 NaN 转换出来是 false，其他都是 true。

```javascript
Boolean(0)          //false
Boolean(null)       //false
Boolean(undefined)  //false
Boolean(NaN)        //false
Boolean(1)          //true
Boolean(13)         //true
Boolean('12')       //true
```

#### 隐式类型转换

凡是通过逻辑运算符 (&&、 ||、 !)、运算符 (+、-、*、/)、关系操作符 (>、 <、 <= 、>=)、相等运算符 (==) 或者 if/while 条件的操作，如果遇到两个数据类型不一样的情况，都会出现隐式类型转换。

**'==' 的隐式类型转换规则**

1. 如果类型相同，无须进行类型转换；
2. 如果其中一个操作值是 null 或者 undefined，那么另一个操作符必须为 null 或者 undefined，才会返回 true，否则都返回 false；
3. 如果其中一个是 Symbol 类型，那么返回 false；
4. 两个操作值如果为 string 和 number 类型，那么就会将字符串转换为 number；
5. 如果一个操作值是 boolean，那么转换成 number；
6. 如果一个操作值为 object 且另一方为 string、number 或者 symbol，就会把 object 转为原始类型再进行判断（调用 object 的 valueOf/toString 方法进行转换）。

```javascript
null == undefined       // true  规则2
null == 0               // false 规则2
'' == null              // false 规则2
'' == 0                 // true  规则4 字符串转隐式转换成Number之后再对比
'123' == 123            // true  规则4 字符串转隐式转换成Number之后再对比
0 == false              // true  e规则 布尔型隐式转换成Number之后再对比
1 == true               // true  e规则 布尔型隐式转换成Number之后再对比
var a = {
  value: 0,
  valueOf: function() {
    this.value++;
    return this.value;
  }
};
// 注意这里a又可以等于1、2、3
console.log(a == 1 && a == 2 && a ==3);  //true f规则 Object隐式转换
// 注：但是执行过3遍之后，再重新执行a==3或之前的数字就是false，因为value已经加上去了，这里需要注意一下
```

**'+' 的隐式类型转换规则**

'+' 号操作符，不仅可以用作数字相加，还可以用作字符串拼接。仅当 '+' 号两边都是数字时，进行的是加法运算；如果两边都是字符串，则直接拼接，无须进行隐式类型转换。

除了上述比较常规的情况外，还有一些特殊的规则，如下所示。

- 如果其中有一个是字符串，另外一个是 undefined、null 或布尔型，则调用 toString() 方法进行字符串拼接；如果是纯对象、数组、正则等，则默认调用对象的转换方法会存在优先级，然后再进行拼接。
- 如果其中有一个是数字，另外一个是 undefined、null、布尔型或数字，则会将其转换成数字进行加法运算，对象的情况还是参考上一条规则。
- 如果其中一个是字符串、一个是数字，则按照字符串规则进行拼接。

```javascript
1 + 2        // 3  常规情况
'1' + '2'    // '12' 常规情况
// 下面看一下特殊情况
'1' + undefined   // "1undefined" 规则1，undefined转换字符串
'1' + null        // "1null" 规则1，null转换字符串
'1' + true        // "1true" 规则1，true转换字符串
'1' + 1n          // '11' 比较特殊字符串和BigInt相加，BigInt转换为字符串
1 + undefined     // NaN  规则2，undefined转换数字相加NaN
1 + null          // 1    规则2，null转换为0
1 + true          // 2    规则2，true转换为1，二者相加为2
1 + 1n            // 错误  不能把BigInt和Number类型直接混合相加
'1' + 3           // '13' 规则3，字符串拼接
```

整体来看，如果数据中有字符串，JavaScript 类型转换还是更倾向于转换成字符串，因为第三条规则中可以看到，在字符串和数字相加的过程中最后返回的还是字符串

**Object 的转换规则**

对象转换的规则，会先调用内置的 [ToPrimitive] 函数，其规则逻辑如下：

1. 如果部署了 Symbol.toPrimitive 方法，优先调用再返回；
2. 调用 valueOf()，如果转换为基础类型，则返回；
3. 调用 toString()，如果转换为基础类型，则返回；
4. 如果都没有返回基础类型，会报错。

```javascript
var obj = {
  value: 1,
  valueOf() {
    return 2;
  },
  toString() {
    return '3'
  },
  [Symbol.toPrimitive]() {
    return 4
  }
}
console.log(obj + 1); // 输出5
// 因为有Symbol.toPrimitive，就优先执行这个；如果Symbol.toPrimitive这段代码删掉，则执行valueOf打印结果为3；如果valueOf也去掉，则调用toString返回'31'(字符串拼接)
// 再看两个特殊的case：
10 + {}
// "10[object Object]"，注意：{}会默认调用valueOf是{}，不是基础类型继续转换，调用toString，返回结果"[object Object]"，于是和10进行'+'运算，按照字符串拼接规则来，参考'+'的规则C
[1,2,undefined,4,5] + 10
// "1,2,,4,510"，注意[1,2,undefined,4,5]会默认先调用valueOf结果还是这个数组，不是基础数据类型继续转换，也还是调用toString，返回"1,2,,4,5"，然后再和10进行运算，还是按照字符串拼接规则，参考'+'的第3条规则
```

## 总结

1. 数据类型的基本概念：这是必须掌握的知识点，作为深入理解 JavaScript 的基础。
2. 数据类型的判断方法：typeof 和 instanceof，以及 Object.prototype.toString 的判断数据类型、手写 instanceof 代码片段，这些是日常开发中经常会遇到的。
3. 数据类型的转换方式：两种数据类型的转换方式，日常写代码过程中隐式转换需要多留意，如果理解不到位，很容易引起在编码过程中的 bug，得到一些意想不到的结果。