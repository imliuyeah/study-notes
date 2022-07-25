# 从一个bug谈起 ——探究 nextTick 在 vue 不同版本下的差异

## 问题

下面这一段代码，做了如下操作：

当点击 checkbox 时将 checkbox 的选中状态设置为false；

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <!-- import CSS -->
  <link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
</head>
<body>
  <div id="app">
    <el-checkbox
      v-for="city in cityOptions"
      :key="city.id"
      v-model="city.checked"
      @change="change(city)"
      >{{city.label}}</el-checkbox>
  </div>
</body>
  <!-- import Vue before Element -->
  <script src="https://cdn.jsdelivr.net/npm/vue@2.5.17/dist/vue.js"></script>
  <!-- import JavaScript -->
  <script src="https://unpkg.com/element-ui/lib/index.js"></script>
  <script>
    new Vue({
      el: '#app',
      data: function() {
        return {
          cityOptions: [
            {
              label: '上海',
              id: 1,
              indeterminate: true,
              checked: false
            }
          ]
        };
      },
      methods: {
        change(city) {
          city.checked = false
        }
      }
    })
  </script>
</html>
```

但是同样的操作在 `vue@2.5.17` 版本与 `vue@2.6.14` 版本下表现并不一致：

在 2.5.17 版本下，第一次点击 checkbox ，在回调中手动将 checkbox  的 check 状态设置为了 false 后，再点击 checkbox 成功勾选；

但在 2.6.14 版本下，第一次点击 checkbox，在回调中手动将 check 状态设置为了 false 后，再次点击 checkbox，勾选状态不变，需要第二次点击 checkbox 才能成功勾选；

## 探索

先看了一下 element 源码中对 el-checkbox 的实现：

```html
// template 部分
  // 使用 span 标签，模拟半选、勾选以及未勾选的状态
  <span class="el-checkbox__inner"></span>
  // 将真正的 input 框隐藏
  <input
    type="checkbox"
    v-model="value"
    @change="handleChange">
```

[^注]: 这里为了方便理解对 input 上绑定的属性做了一些修改和删减。

```javascript
// js 部分
methods: {
    handleChange(ev) {
        // ......
        let value;
        if (ev.target.checked) {
            value = this.trueLabel === undefined ? true : this.trueLabel;
        } else {
            value = this.falseLabel === undefined ? false : this.falseLabel;
        }
        this.$emit('change', value, ev);
    }    
}

```

```css
// css 样式
.el-checkbox__inner:after {
  box-sizing: content-box;
  content: "";
  border: 1px solid #fff;
  border-left: 0;
  border-top: 0;
  height: 7px;
  left: 4px;
  position: absolute;
  top: 1px;
  transform: rotate(45deg) scaleY(0);
  width: 3px;
  transition: transform .15s ease-in .05s;
  transform-origin: center
}
// 将真正的 input 框隐藏
.el-checkbox__original {
  opacity: 0;
  outline: none;
  position: absolute;
  margin: 0;
  width: 0;
  height: 0;
  z-index: -1
}
```

在 element 源码中，实际上是利用 css 样式将原生的 input DOM 元素进行了隐藏，并额外使用一个 span 标签来模拟半选、勾选以及未勾选状态下的样式；

这里我将 input 框的样式进行修改，使它能够在页面上展示出来，并且测试了在不同 vue 版本下元素 input 元素的勾选状态的变化：

![image-20220721213448472](C:\Users\666\AppData\Roaming\Typora\typora-user-images\image-20220721213448472.png)

## 过程分析

1. 点击 el-checkbox，span 和 input 元素被点击；

2. input 勾选状态由 未勾选 变为 已勾选，同时 checked 属性被添加到 input 标签上；

3. input 元素上使用 v-model 双向绑定了一个变量（对于 checkbox 类型的 v-model，实际上是 `v-bind:checked="xxx"` 以及 `v-on:change=“function(){xxx}”` 的语法糖），由于 checked 属性被改变，因此 v-on 的 change 回调被触发，向外 emit change 事件；

   ```html
   // 
   <input
       type="checkbox"
       v-model="value"
       @change="handleChange">
   ```

   ```javascript
   props: {
       value: {}
   },
   methods: {
       // input change 事件
       handleChange(ev) {
           // ......
           let value;
           if (ev.target.checked) {
               value = this.trueLabel === undefined ? true : this.trueLabel;
           } else {
               value = this.falseLabel === undefined ? false : this.falseLabel;
           }
           // 向外 emit change 事件
           this.$emit('change', value, ev);
       }    
   }
   ```

4. el-checkbox 接收到 input emit 出来的事件，触发 change 回调，在 change 回调中，我们将 checked 设为 false；

   ```javascript
   // el-checkbox 上绑定的 change 事件
   change(city) {
      city.checked = false
   }
   ```

5. 同样 el-checkbox 也使用 v-model 双向绑定了 checked 的值（这里的 v-model 默认绑定的属性相当于 `v-bind: value ="city.checked"` ）；由于在第 4 步中，`city.checked` 被置为 false ，则子组件中的 v-model 的 value 属性也会变化，最终改变 input 元素上的 checked 属性。

   ```html
   <el-checkbox
       v-for="city in cityOptions"
       v-model="city.checked"
       @change="change(city)"
       >{{city.label}}</el-checkbox>
   ```


**2.5 版本：**

![动画2](E:\yeah\02.study\notes\asserts\动画2.gif)

**2.6 版本：**

![动画](E:\yeah\02.study\notes\asserts\动画.gif)

从 gif 中可以看出，**2.5 版本**当我们首次点击半选状态下的 span 标签时，input 实际上是先被勾选，随后立即又取消了勾选（状态一闪而过，gif可能看不出来）；

而在 **2.6 版本**下，当我们首次点击半选状态下的 span 标签时，input 被勾选，而后并没有取消勾选，也就是说 `city.checked = false` 这段代码看起来并没有生效的亚子；

此时，没生效？为啥没生效？冒泡？渲染时机？nextTick？。。。一系列问号。。。

## 原因

那么又是什么原因导致的同一段代码在两个版本的 vue 下表现不一致呢？
