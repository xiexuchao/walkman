彻底搞懂injector -- instantiate
=================================

> 在invoke使用注入参数调用函数的同时， instantiate会使用注入它的构造器参数构造一个新的对象。

> instantiate给我们一个绝妙无伦的视角， javascript对象是如何工作的。在javascript中， 类就仅仅是一个函数， 类实例也只是通过new操作符对函数的调用。

> 假设我们有一个简单的类:

```
function Person(firstName, lastName) {
  this.firstName = firstName;
  this.lastName = lastName;
}
```
> 我们可以通过函数原型属性prototype向该类添加方法。
```
Person.prototype.beNiceTo = function() {
  console.log(this.firstName + ' ' + this.lastName + ' is my friend';
}
```

> 然后我们可以使用new操作符来创建这个类的实例

```
var plum = new Person('Professor', 'Plonk');
plum.beNiceTo();

// Professor Plonk is my friend
```

> 所有的事情都不错哈。

> 现在我们为了调用一个方法，并且注入参数到它里边， 注入器使用apply方法的优势。 这个方法允许你调用函数， 并传入一个数组， 作为传入函数的参数。

> 非常有用， 但是使用apply的问题是它仅仅在你想要调用函数的时候才会工作。 没有办法混合new和apply. 注入器使用了一种比较聪明的方式解决这个问题。



