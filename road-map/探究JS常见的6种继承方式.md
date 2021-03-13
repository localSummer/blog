### **探究 JS 常见的 6 种继承方式**

#### 第一种：原型链继承

原型链继承是比较常见的继承方式之一，其中涉及的构造函数、原型和实例，三者之间存在着一定的关系，即每一个构造函数都有一个原型对象，原型对象又包含一个指向构造函数的指针，而实例则包含一个原型对象的指针。

```javascript
function Parent1() {
  this.name = 'parent1'
  this.play = [1, 2, 3]
}

function Child1() {
  this.type = 'child1'
}

Child1.prototype = new Parent1()

let child1 = new Child1()

console.log(child1)
```

上面的代码看似没有问题，虽然父类的方法和属性都能够访问，但其实有一个潜在的问题

```javascript
  let c1 = new Child1();

  let c2 = new Child1();

  c1.play.push(4);

  console.log(c1.play, c2.play);

```

这段代码在控制台执行之后，可以看到结果如下：

![Drawing 0.png](https://s0.lgstatic.com/i/image2/M01/05/2A/CgpVE1_9B-GAE-pIAAAYEv_K_84787.png)

明明只改变了 c1 的 play 属性，为什么 c2 也跟着变了呢？原因很简单，因为两个实例使用的是同一个原型对象。它们的内存空间是共享的，当一个发生变化的时候，另外一个也随之进行了变化，这就是使用原型链继承方式的一个缺点。

#### 第二种：构造函数继承（借助 call）

```javascript
function Parent1() {
  this.name = 'parent1'
}

Parent1.prototype.getName = function () {
  return this.name
}

function Child1() {
  // 借助call方法改变this指向，从而将Parent中的属性添加至Child中
  Parent1.call(this)
  this.type = 'child1'
}

let child = new Child1()

// 正常运行
console.log(child)

// 运行报错，因为child没有getName方法
console.log(child.getName())

```

运行结果如下

![Drawing 1.png](https://s0.lgstatic.com/i/image2/M01/05/2A/CgpVE1_9B-qAHGGjAABBe0l-7oE835.png)

![Drawing 2.png](https://s0.lgstatic.com/i/image/M00/8D/42/Ciqc1F_9B_KACrgnAABDSFXfnx0666.png)

从上面的结果就可以看到构造函数实现继承的优缺点，它使父类的引用属性不会被共享，优化了第一种继承方式的弊端；但是随之而来的缺点也比较明显——**只能继承父类的实例属性和方法，不能继承原型属性或者方法**。

#### 第三种：组合继承（前两种组合）

这种方式结合了前两种继承方式的优缺点，结合起来的继承，代码如下

```javascript
function Parent3() {
  this.name = 'parent3'
  this.play = [1, 2, 3]
}

Parent3.prototype.getName = function () {
  return this.name
}

function Child3() {
  // 第二次调用Parent
  Parent3.call(this)
  this.type = 'child3'
}

// 第一次调用Parent
Child3.prototype = new Parent3()

// 手动挂上构造器，指向自己的构造函数
Child3.prototype.constructor = Child3

let c1 = new Child3()
let c2 = new Child3()
c1.play.push(4)
console.log(c1.play, c2.play) // 不互相影响
console.log(c1.getName()) // 正常输出'parent3'
console.log(c2.getName()) // 正常输出'parent3'

```

执行上面的代码，可以看到控制台的输出结果，之前方法一和方法二的问题都得以解决。

![Drawing 3.png](https://s0.lgstatic.com/i/image/M00/8D/42/Ciqc1F_9B_uAHQtBAAAgMta5Vz8933.png)

但是这里又增加了一个新问题：通过注释我们可以看到 Parent3 执行了两次，第一次是改变Child3 的 prototype 的时候，第二次是通过 call 方法调用 Parent3 的时候，那么 Parent3 多构造一次就多进行了一次性能开销，这是我们不愿看到的。

上面介绍的更多是围绕着构造函数的方式，那么对于 JavaScript 的普通对象，怎么实现继承呢？

#### 第四种：原型式继承

这里不得不提到的就是 ES5 里面的 Object.create 方法，这个方法接收两个参数：一是用作新对象原型的对象、二是为新对象定义额外属性的对象（可选参数）。

```javascript
let parent4 = {
  name: 'parent4',
  friends: ['p1', 'p2', 'p3'],
  getName: function () {
    return this.name
  },
}

let person = Object.create(parent4)

person.name = 'Tom'
person.friends.push('jerry')

let person2 = Object.create(parent4)
person2.friends.push('lucy')

console.log(person.name)
console.log(person.name === person.getName())
console.log(person2.name)
console.log(person.friends)
console.log(person2.friends)

```

通过 Object.create 这个方法可以实现普通对象的继承，不仅仅能继承属性，同样也可以继承 getName 的方法，请看这段代码的执行结果。

![Drawing 4.png](https://s0.lgstatic.com/i/image/M00/8D/4E/CgqCHl_9CASAJMvwAAA_d30-jH8783.png)

最后两个输出结果是一样的，讲到这里你应该可以联想到浅拷贝的知识点，关于引用数据类型“共享”的问题，其实 Object.create 方法是可以为一些对象实现浅拷贝的。

那么关于这种继承方式的缺点也很明显，**多个实例的引用类型属性指向相同的内存，存在篡改的可能**，接下来我们看一下在这个继承基础上进行优化之后的另一种继承方式——寄生式继承。

#### 第五种：寄生式继承

使用原型式继承可以获得一份目标对象的浅拷贝，然后利用这个浅拷贝的能力再进行增强，添加一些方法，这样的继承方式就叫作寄生式继承。

虽然其优缺点和原型式继承一样，但是对于普通对象的继承方式来说，寄生式继承相比于原型式继承，还是在父类基础上添加了更多的方法。

```javascript
let parent5 = {
  name: 'parent5',
  friends: ['p1', 'p2', 'p3'],
  getName: function () {
    return this.name
  },
}

function clone(original) {
  let clone = Object.create(original)
  clone.getFriends = function () {
    return this.friends
  }
}

let person = clone(parent5)

console.log(person.getName())
console.log(person.getFriends())

```

通过上面这段代码，我们可以看到 person 是通过寄生式继承生成的实例，它不仅仅有 getName 的方法，而且可以看到它最后也拥有了 getFriends 的方法

![Drawing 5.png](https://s0.lgstatic.com/i/image/M00/8D/4E/CgqCHl_9CA2AT-ozAAAWLoCKBTA043.png)



从最后的输出结果中可以看到，person 通过 clone 的方法，增加了 getFriends 的方法，从而使 person 这个普通对象在继承过程中又增加了一个方法，这样的继承方式就是寄生式继承。

在上面第三种组合继承方式中提到了一些弊端，即两次调用父类的构造函数造成浪费，下面要介绍的寄生组合继承就可以解决这个问题。

#### 第六种：寄生组合式继承

结合第四种中提及的继承方式，解决普通对象的继承问题的 Object.create 方法，我们在前面这几种继承方式的优缺点基础上进行改造，得出了寄生组合式的继承方式，这也是所有继承方式里面相对最优的继承方式

```javascript
function Parent6() {
  this.name = 'parent6'
  this.play = [1, 2, 3]
}
Parent6.prototype.getName = function () {
  return this.name
}

function Child6() {
  Parent6.call(this)
  this.friends = 'child5'
}

function clone(parent, child) {
  child.prototype = Object.create(parent.prototype)
  child.prototype.constructor = child
}

clone(Parent6, Child6)

Child6.prototype.getFriends = function () {
  return this.friends
}

let person = new Child6()
console.log(person)
console.log(person.getName())
console.log(person.getFriends())

```

通过这段代码可以看出来，这种寄生组合式继承方式，基本可以解决前几种继承方式的缺点，较好地实现了继承想要的结果，同时也减少了构造次数，减少了性能的开销，我们来看一下上面这一段代码的执行结果。

![Drawing 6.png](https://s0.lgstatic.com/i/image/M00/8D/4E/CgqCHl_9CBWATQbEAABszTJIdBQ249.png)

整体看下来，这六种继承方式中，寄生组合式继承是这六种里面最优的继承方式。另外，ES6 还提供了继承的关键字 extends，通过ES6转ES5的方式可以看到，其底层实现逻辑也是采用了寄生组合式继承，因此也证明了这种方式是较优的解决继承的方式。

#### 总结

![图片7.png](https://s0.lgstatic.com/i/image/M00/8D/4A/Ciqc1F_9SVuAfHXWAAEfwyAfiC0647.png)