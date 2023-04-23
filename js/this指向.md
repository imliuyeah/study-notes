
[toc]

# this 指向

- this 永远指向一个对象；
- this 的指向，就是**函数运行时所在的环境**；由于 JavaScript 中**运行环境是可以动态切换**的，所以只有在函数运行时，才能确定 this 的指向。

# this 指向为什么会改变？

    因为，在 JS 中函数是引用类型，可以作为值被传递

```js
var A = {
    name: '张三',
    f: function () {
        console.log('姓名：' + this.name);
    }
};

var B = {
    name: '李四'
};

B.f = A.f;
B.f()   // 姓名：李四
A.f()   // 姓名：张三
```

    上面代码中，`A.f`属性被赋给 `B.f`，也就是 `A`对象将匿名函数的 **地址** 赋值给 `B`对象；

    所以在调用时，函数会根据运行环境的不同，分别指向对象 A 和 对象 B

```js
function foo() {
    console.log(this.a);
}
var obj2 = {
    a: 2,
    fn: foo
};
var obj1 = {
    a: 1,
    o1: obj2
};
obj1.o1.fn(); // 2
```

    `obj1`对象的 `o1`属性值是 `obj2`对象的地址，而 `obj2`对象的 `fn`属性的值是函数 `foo`的地址；

    函数`foo`的调用环境是在 `obj2`中的，因此 `this`指向对象 `obj2`;

# this 指向总结

    可以分为以下常见的 5 种：

- 对象中的方法（见上方例子）
- 事件绑定
- 构造函数
- 定时器
- call()、apply() 方法

## 事件绑定中的 this 指向

### 行内绑定

    行内绑定可以分为两种情况

1. 事件绑定的语法 **在html节点内**，这种方式，函数运行时如果是在全局环境，那么 this 就指向 window，如下：

   ```html
   <input type="button" value="按钮" onclick="clickFun()">
   <script>
       function clickFun(){
           this // 此函数的运行环境在全局window对象下，因此this指向window;
       }
   </script>
   ```
2. 如果不是一个函数调用，直接在当前节点对象环境下使用this，那么显然this就会指向当前节点对象；

   ```html
   <input type="button" value="按钮" onclick="this">
   <!-- 运行环境在节点对象中，因此this指向本节点对象 -->
   ```

### 动态绑定和事件监听

动态绑定

```html
<input type="button" value="按钮" id="btn">
<script>
    var btn = document.getElementById('btn');
    // 相当于 把函数地址 赋值给了 btn 这个对象上的 onclick 属性
    btn.onclick = function(){ 
        this ;  // this指向本节点对象
    }
</script>
```

事件监听

```html
<input type="button" value="按钮" id="btn">
<script>
    var btn = document.getElementById('btn');
    // 相当于 把函数地址 赋值给了 btn 这个对象上的 onclick 属性
    btn.addEventListener('click', function(){
        this
    })
</script>
```

    这里相当于把函数的地址，赋值给了 btn 这个对象上的 onclick 属性，那么函数在**调用时的环境就是 btn 对象**，因此 this 指向 btn；

## 构造函数的 this 指向

    构造函数中的 this ，会强制指向**被构造函数创造出来的实例**。

```js
function Pro(){
    this.x = '1';
    this.y = function(){};
}
// 此时，this 指向 p
// 所以 p.x = 1
var p = new Pro(); 
```

## 定时器中的 this 指向

```js
var obj = {
    fun: function(){
        this;
    }
}
setInterval(obj.fun, 1000);      // this指向window对象
```

    上面的代码，相当于把`obj.fun` 的地址当作参数传递给了 `setInterval` 方法；**1000 毫秒之后，`obj.fun` 函数被运行，此时运行的环境已经变成了 window 对象**；

```js
var obj = {
    fun: function(){
        this;
    }
}
setInterval('obj.fun', 1000);      // this指向obj对象
```

    而在`setInterval('obj.fun()',1000)` 中的第一个参数，实际则是传入的一段**可执行的 JS 代码**；1000毫秒后当 JS 引擎来执行这段代码时，则是**通过 `obj` 对象来找到 `fun` 函数并调用执行，那么函数的运行环境依然在 对象 `obj` 内**，所以函数内部的this也就指向了 `obj` 对象；

## call() 、apply() 方法中的 this 指向

    call() 方法 和 apply() 方法都可以接受一个对象作为参数，用来指定**调用这两个方法的函数的 this 指向**；

    两个方法的作用一致，区别仅在**函数实参参数传递的方式上**；

### call() 方法

    语法规则：

    **函数名.call(obj, arg1, arg2...argN)**

- obj：函数的 this 指向的对象
- arg1, arg2...argN：参数列表，参数之间用逗号隔开

### apply() 方法

    语法规则：

    **函数名.apply(obj, [arg1, arg2...argN])**

- obj：函数的 this 指向的对象
- [arg1, arg2...argN]：参数列表，是个数组

# 箭头函数 this 指向
