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

本提案的一个目标是：通过某些语法，允许访问相同类的其他实例的私有字段。并且使用不常见标识符来指向属性是不符合 JS way 的（譬如 `with`关键字，通常被认为是一个失败的设计）。

## 鉴于`this.#x`可以访问私有字段，那`this['x']`为什么不行？

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

### `this.#x`与`this['x']`有不同的语义破坏了当前语义的平衡吗？

不完全是，但这确实是一个争议。`this.#x`是新的语法，从这点上是毫无影响的。

但是另一方面，两者的不同让人感到意外，这也确实当前提案的负面影响。

## 为什么不省掉点语法，直接用`this#x`访问？

稍后我们会介绍一种简写：`#x`代替`this.#x`。这会使简写语法有一个[ASI风险](https://github.com/tc39/proposal-private-fields/issues/39#issuecomment-237121552)

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

## 为什么本提案允许访问相同类的其他实例的私有字段？其他语言通常不会禁止这样做吗？

这很有用，请看readme中，`Point`的`equals`方法的例子（译者按：我没找到）。事实上，[其他语言](https://github.com/tc39/proposal-private-fields/issues/90#issuecomment-307201358)也因为相同的原因而允许这样做，下面的例子在Java中完全合法：

```java
class Point {
	private int x = 0;
	private int y = 0;
	public boolean equals(Point p) { return this.x == p.x && this.y == p.y; }
}
```

## 在所有的Unicode符号中，为什么选择了`#`？

没人说`#`是一个更好看、更符合直觉的符号来表示私有状态。这是排除法的结果：

- `@`是首选，但是它被装饰器先占用了。TC39考虑过交换装饰器与私有状态符，但最终委员会决定遵循现有转换编译器用户的使用习惯。

- `_`会导致现存JavaScript代码的兼容性问题。长时间以来，js都允许在标识符和（公有）属性名前加`_`。

- 其他可以被用作中缀操作符且不是前缀操作符的符号，例如`%`，`^`，`&`，`?`——在当前椰树的语法环境下，`x.%y`不合法因而不会有二义性——都是有可能的。然后简写会导致ASI问题，比如下面的例子看起来就像是在使用中缀操作符：

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

- - 如果`private(that)`表示一个对象含有私有字段，这会使优化这个对象额外的独立空间变得困难。

- - 