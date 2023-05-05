# new 操作符做了什么？

比如说 `var a = new B()`

1. 创建一个新对象 `{}`；
2. 将新创建对象 `{}` 的 `__proto__` 属性，指向构造函数的原型对象，也就是让新对象继承 `B.prototype`；
3. 把 `this` 的上下文指向这个创建的新对象 `{}`；
4. 如果构造函数 B 返回了一个 **对象**，则使用构造函数 B 返回的对象赋值给 a，否则的话返回创建的新对象。

# 实现一个 new 操作符

```js
function create (father, ...args) {
    // 先判断用来继承的参数 是不是一个函数
    if (typeof father !== 'function') {
        return
    }
    // 创建一个空对象，并吧空对象的 __proto__ 属性指向 要继承的构造函数的 prototype，这里可以直接用 Object.create 
    let obj = Object.create(father.prototype)
    // 使用传入的参数，将构造函数 father 的属性绑定到新对象上，初始化新创建的空对象
    let result = father.apply(obj, args)
    // 判断构造函数 father 返回的是不是一个对象，如果是则使用 father 返回的对象，否则返回我们创建的新对象
    return result instanceof Object ? result : obj
}

// 使用方法
function Person (name, age) {
    this.name = name
    this.age = age
}

Person.prototype.sayName = function () {
    console.log(this.name)
}

let child = create(Person, 'lisi', 10)

console.log(child.name) // lisi
console.log(child.age) // 10
```
