# Vue.set 实现原理探究
## 使用方法:  
Vue.set(target, key, value)  
例子:   
Vue.set(a, 'check', true)  
## 源码
相关实现在 src/core/observer/index.js 下

```javascript
   // src/core/observer/index.js
   /**
     * Set a property on an object. Adds the new property and
     * triggers change notification if the property doesn't
     * already exist.
     * 给一个对象设置属性.如果这个属性不存在对象上,那么添加新属性并且触发通知依赖
     * 使用: Vue.set(target, key, value)
     * 例子: Vue.set(a, 'check', true)
     */
    export function set (target: Array<any> | Object, key: any, val: any): any {
      if (process.env.NODE_ENV !== 'production' &&
        (isUndef(target) || isPrimitive(target))
      ) {
        warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
      }
      // 如果 target 是数组 并且key是有效的数组索引
      if (Array.isArray(target) && isValidArrayIndex(key)) {
        // 修改数组的长度?
        target.length = Math.max(target.length, key)
        // 调用数组的 splice 方法, 从索引 key 的位置开始删除 1 个元素，插入 val
        target.splice(key, 1, val)
        return val
      }
      // 判断 target 上本身有无对应的 key
      if (key in target && !(key in Object.prototype)) {
        target[key] = val
        return val
      }
      // 这里可以通过 target.__ob__ 来判断 target 是不是一个响应式对象
      const ob = (target: any).__ob__
      if (target._isVue || (ob && ob.vmCount)) {
        process.env.NODE_ENV !== 'production' && warn(
          'Avoid adding reactive properties to a Vue instance or its root $data ' +
          'at runtime - declare it upfront in the data option.'
        )
        return val
      }
      // 如果 target 不是个响应式对象, 那么直接给 target[key] 赋值 并返回值
      if (!ob) {
        target[key] = val
        return val
      }
      // 如果 target 是个响应式对象, 那么调用 defineReactive 把新加的 val 也变成响应式的
      defineReactive(ob.value, key, val)
      // 调用观察者上的依赖的 notify 方法
      ob.dep.notify()
      return val
    }
   
```