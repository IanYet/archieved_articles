# javascript private 语法 FAQ [翻译]

---

> [原文链接：PRIVATE_SYNTAX_FAQ](https://github.com/tc39/proposal-class-fields/blob/master/PRIVATE_SYNTAX_FAQ.md)

## 什么是 `#`？

`#`是新的`_`符号。

```js
class A {
  _hidden = 0;
  m() {
    return this._hidden;
  }
}
```

以上代码声明了一个带有私有字段的class，但该私有属性是 **程序员间的默认约定** ，从代码层面来讲，`_hidden`字段仍是完全public的。

```js
class B {
  #hidden = 0;
  m() {
    return this.#hidden;
  }
}
```

以上代码才真正声明了一个对`class B`私有的字段`#hidden`，该私有属性是**语言层面强制保证**的。

因为web兼容性的原因，我们不能直接更改`_`的用法。

## 为什么不能直接访问`this.x`

一个叫`x`的私有字段，不应该妨碍同时有一个叫`x`的公有字段，所以不能通过常规的方式去访问一个私有字段。

这只是Javascript的一个问题：缺少静态类型。静态类型语言不需要一个额外的符号，只需要用类型声明区分 external-public/internal-private 两种情况。然而动态类型语言缺少分辨这些情况的静态信息。

### 为什么此提案允许私有字段`#x`与公有字段`x`同时存在？

1. 如果私有字段与公有字段冲突，那将会破坏封装，理由如下

2. 私有字段的一个非常重要的属性是：子类不需要知道它们。即使父类有一个叫`x`的私有字段，子类也应该可以定义一个叫`x`的属性。

这是其他语言普遍承认的特性。比如以下代码在Java里完全合法

```java
class Base {
  private int x = 0;
}

class Derived extends Base {
  public int x = 0;
}
```

### 为什么不在运行时做接收者的类型检查，以此来决定是访问私有字段x还是公有字段x？

属性访问的语义已经足够复杂了。如果加上这个feature，会使每一次属性访问变得更慢。

这也会允许该类的方法可以通过取巧的手段操作一个非该类实例的公有同名字段，即使此字段在该类实例里是私有的。具体可以看[这个例子](https://github.com/tc39/proposal-private-fields/issues/14#issuecomment-153050837)。

### 为什么不干脆一直持有一个`obj.x`来指向 声明了私有字段`x`的类 的私有字段？

类的方法常常去操作非该类实例的对象。如果，仅仅是因为某段代码需要执行在一个声明了私有字段`x`的类里，就让`obj.x`突然不再指向`obj`的公有字段`x`。当`obj`不是该类实例的时候，会导致某些意想不到的后果。

### 为什么不给`this`关键字特殊语义？

`this`已经是JS令人困惑的源头之一，还是不要让它更难懂了吧。并且，还有一个重构代码的风险：如果`this.x`与`const that = this; that.x`有不同的语义，结果会难以预料。

### 为什么不能直接访问`this`的字段，比如持有一个不常见的`x`来指向私有字段`x`?

本提案的一个目标是：通过某些语法，允许访问相同类的其他实例的私有字段（见下）。并且使用不常见标识符来指向属性是不符合 JS way 的（譬如 `with`关键字，通常被认为是一个失败的设计）。

## 鉴于`this.#x`可以访问私有字段，那`this['#x']`为什么不行？

1. 这会使属性访问语义变复杂。

2. 动态访问私有字段是违背'private'概念的，比如：

```js
class Dict extends null {
  #data = something_secret;
  add(key, value) {
    this[key] = value;
  }
  get(key) {
    return this[key];
  }
}

(new Dict).get('#data'); // returns something_secret
```

### `this.#x`与`this['#x']`有不同的语义破坏了当前语义的平衡吗？

不完全是，但这确实是一个争议。`this.#x`是新的语法，从这点上是毫无影响的。

但是另一方面，两者的不同让人感到意外，这也确实当前提案的负面影响。

## 为什么不省掉点语法，直接用`this#x`访问？

稍后我们会介绍一种简写：`#x`代替`this.#x`。不加点会使简写语法有[ASI风险](https://github.com/tc39/proposal-private-fields/issues/39#issuecomment-237121552)

更一般地说，委员会大多认为：`this.#`更明显地表示了字段访问。

## 为什么不声明 `private x`？

这是其他语言（尤其是Java）采用的声明方式，这往往表示`this.x`是能正常访问私有字段的。考虑这种情况（见之上的论述），JavaScript只会创建或者访问一个公有字段而不是抛出错误。这是一个潜在的bug源，也会使本应该是私有的字段变成public。

不使用private关键字同时保证了声明与读取的平衡，就像声明与访问公有字段一样：

```js
class A {
  pub = 0;
  #priv = 1;
  m() {
    return this.pub + this.#priv;
  }
}
```

## 为什么本提案允许访问相同类的其他实例的私有字段？其他语言不禁止这样吗？

这很有用，请看readme中，`Point`的`equals`方法的例子（译者按：我没找到）。事实上，[其他语言](https://github.com/tc39/proposal-private-fields/issues/90#issuecomment-307201358)也因为相同的原因而允许这样做，下面的例子在Java中完全合法：

```java
class Point {
	private int x = 0;
	private int y = 0;
	public boolean equals(Point p) { return this.x == p.x && this.y == p.y; }
}
```

## 在所有的Unicode符号中，为什么选择了`#`？

没人说`#`是一个更好看、更符合直觉的表示私有状态的符号。这是排除法的结果：

- `@`是首选，但是它被装饰器先占用了。TC39考虑过交换装饰器与私有状态符，但最终委员会决定遵循现有转换编译器用户的使用习惯。

- `_`会导致现存JavaScript代码的兼容性问题。长时间以来，js都允许在标识符和（公有）属性名前加`_`。

- 其他可以被用作中缀操作符且不是前缀操作符的符号，例如`%`，`^`，`&`，`?`——在当前js的语法环境下，`x.%y`不合法因而不会有二义性——都是有可能的。但是简写会导致ASI问题，比如下面的例子看起来就像是在使用中缀操作符：

```js
class Foo {
  %x;
  method() {
    calculate().my().value()
    %x.print()
  }
}
```

用户本想调用`this.%x`的`print`方法，但结果却用了模运算。

- 其他不在 ASCII 或者 IDStart中的unicode符号也可以，但是某些用户可能难以用键盘输入——他们在常规键盘上并不常见。

最终，唯一的其他选择就是更长序列的标点符号，它们明显不如单个字符。

## 为什么不用 `private.x`来指向`this`的私有字段，用`private(that).x`指向另一个对象的私有字段？

这种方法有以下几个问题：

- 如果通过`private x`声明，开发者可能更愿意用`this.x`而不是`private.x`来访问字段；如果声明与读取都用`private.x`（比如没有初始程序），那么代码只是just work。

- `private(that)`看起来像是一个单独的表达式，但是这会导致几个问题：

- - 如果`private(that)`表示一个有私有字段的对象，这会使优化这个对象额外的独立空间变得困难。

- - 如果`private(that)`单独使用是语法错误，必须要在其后进行属性访问，这与函数和伪函数(如`import()`)在JavaScript中的工作方式截然不同。

## 为什么本提案不允许机制（mechanism）从声明该私有字段的类外（比如测试程序）访问或者修改它？其他语言也不允许这样吗？

这样做会破坏封装（理由见下）。其他语言允许这样做也不是一个十分充分的理由，特别是某些语言(例如c++)，是通过直接修改内存来实现的。这不是一个必要的目的。

## “封装”/“强私有（hard private）”是什么意思？

它们表示私有字段*纯粹*是内部的：如果不直接查看类的源码，除非类主动暴露，否则类外部的JS代码（包括子类与父类）应该无法探知或影响该类实例的任何私有字段的存在、名称或值，

这意味着诸如[getOwnPropertySymbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols)等映射方法是不能暴露私有字段的。

这也意味着，如一个类有个叫`x`的私有字段，那么该类的实例`obj`在类外执行`obj.x`的时候，应该去访问共有字段，就好像该类没有私有字段一样。禁止访问私有字段，否则，就要抛出错误。注意，在其他像Java一样，在编译时期进行类型检查，并且只能通过映射APIs来动态访问字段的语言中，这并不常见。

## 封装为什么是本提案的一个目标？

1. 库的作者们发现，用户会使用任何暴露在外的接口，哪怕这些接口并不在文档里。同时，作者们不认为自己可以仅仅因为，用户使用了本不想让用户使用的接口，就去破坏用户们编写的页面与app。因此，他们希望用强私有(hard private)状态来更完备地隐藏实现细节。

2. 尽管现在已经可以用per-instance闭包或WeakMaps（见下）实现封装，但这两种方法反人类地使用了class，并且在内存使用语义上让人迷惑。除此之外，per-instance闭包不允许同一个类的不同实例相互访问私有字段（见上），而WeakMaps有潜在的暴露私有数据的风险。另外，WeakMaps大多数情况下要比私有字段慢。

3. 使用Symbol作为属性名同样可以实现“隐藏”但不封装。（见下）

本提案目前偏向于强私有（hard private），让装饰器或者其他机制（mechanism）为类提供一个可选的对外接口。我们打算在此过程中收集反馈，以确定这是否是正确的语义。

了解更多请看[此issue](https://github.com/tc39/proposal-private-fields/issues/33)。

### 如何用WeakMaps实现封装？

```js
const Person = (function(){
  const privates = new WeakMap;
  let ids = 0;
  return class Person {
    constructor(name) {
      this.name = name;
      privates.set(this, {id: ids++});
    }
    equals(otherPerson) {
      return privates.get(this).id === privates.get(otherPerson).id;
    }
  }
})();
let alice = new Person('Alice');
let bob = new Person('Bob');
alice.equals(bob); // false
```

这种方法存在潜在危险。

假设我们想增加一个自定义的回调函数来构造 greetings：

```js
const Person = (function(){
  const privates = new WeakMap;
  let ids = 0;
  return class Person {
    constructor(name, makeGreeting) {
      this.name = name;
      privates.set(this, {id: ids++, makeGreeting});
    }
    equals(otherPerson) {
      return privates.get(this).id === privates.get(otherPerson).id;
    }
    greet(otherPerson) {
      return privates.get(this).makeGreeting(otherPerson.name);
    }
  }
})();
let alice = new Person('Alice', name => `Hello, ${name}!`);
let bob = new Person('Bob', name => `Hi, ${name}.`);
alice.equals(bob); // false
alice.greet(bob); // === 'Hello, Bob!'
```

虽然看上去是ok的，但是：

```js
let mallory = new Person('Mallory', function(name) {this.id = 0; return `o/ ${name}`;});
mallory.greet(bob); // === 'o/ Bob'
mallory.equals(alice); // true. Oops!
```

### 如何用Symbols实现隐藏但不封装？

```js
const Person = (function(){
  const _id = Symbol('id');
  let ids = 0;
  return class Person {
    constructor(name) {
      this.name = name;
      this[_id] = ids++;
    }
    equals(otherPerson) {
      return this[_id] === otherPerson[_id];
    }
  }
})();
let alice = new Person('Alice');
let bob = new Person('Bob');
alice.equals(bob); // false

alice[Object.getOwnPropertySymbols(alice)[0]]; // == 0, which is alice's id.
```

### 此提案与brand checks的关系？

本提案仅仅在对象构造函数里增加了私有字段，也增添了访问不存在私有字段的异常。有有些人将“brand check”归因于这些属性的组合，但是社区一直在讨论到底要提供哪种“保证”。可以明确的是，目前可以提供的保证就是。**如果一个实例含有 声明在特定类里的私有字段，那么该类的构造函数只为此实例运行**。

比如，给出如下定义：

```js
class F { #f; checkF() { this.#f; } }
class G extends F { #g; checkG() { this.#g; } }
```

如果一个对象`obj`没有响应 `F.prototype.checkF.call(obj)`，我们可以认为 `obj`是被 `new F`创建的。

相比而言，由于原型链是可变的，我们并不知道“一个不是从`G.prototype.checkG.call(obj)`抛出的对象”的全部信息——在这种情况下，我们只知道`new G`被调用了，然后父类的构造函数返回了`obj`。比如，以下可能是一个用了“super return trick"的情况：

```js
let obj = { };
Object.setPrototypeOf(G, class { constructor() { return obj; });
new G;
Object.setPrototypeOf(G, F);
G.prototype.checkG.call(obj);  // doesn't throw
F.prototype.checkF.call(obj);  // throws
```

这个奇怪的幽灵对象`obj`有"G"的 “brand”而不是“F”。这不能被认为是“broken”，这只是类的动态原型链给了开发者做这种事情的灵活性而已。有时候，“可预测性”比“灵活性”更有用。

我们可以通过防止在构造对象时动态查询的原型链的突变来阻止这种模式。比如，你可以调用`Object.preventExtensions(G)`来确保`G`的父类是`F`。这种情况下，对象即使不响应`checkG`，也仍然有一个私有字段`#F`。

### 私有字段与Proxy如何一起使用？

一个Proxy目标对象的私有字段不能被Proxy本身访问。比如，下面的例子应该抛出类型错误：

```js
class K { #x; get x() { return this.#x; } }
let k = new Proxy(new K, { });
k.x  // TypeError
```

这有点像内部插槽或WeakMaps：目标对象里的私有字段并没有传递给Proxy。虽然这种行为不满足“Proxy以一种透明的方式观察对象操作”的期望，但是不传递行为支持以下功能：

- 私有字段可用于匹配内部插槽的行为的补充，而不会对Proxy的行为产生影响。

- Proxy目标对象被封装，不需要将目标对象的信息泄露给Proxy使用，也不需要将私有字段的任何信息泄露给Proxy traps

- 语义类似于WeakMap，它为程序员提供了一个简单的心智模型(所有东西要么是属性要么是一个WeakMap)，而不是第三种选择。

但是反过来，Proxy对象可以用“super return trick”在其上直接定义私有字段。例子如下：

```js
class Super { constructor(x) { return x; } }
class Trick extends Super {
  #x;
  checkX() { this.#x; }
}

let target = { };
let proxy = new Proxy(target, { });
new Trick(proxy);
Trick.prototype.checkX.call(proxy);   // No exception thrown
Trick.prototype.checkX.call(target);  // TypeError
```

解决“目标对象的私有字段不能传递给Proxy”的一种办法是用 membrane pattern，例子见[Salesforce's Observable Membrane](https://github.com/salesforce/observable-membrane)。