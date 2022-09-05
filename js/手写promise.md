参考：[面试官：“你能手写一个 Promise 吗”](https://zhuanlan.zhihu.com/p/183801144)

# Promise 规范

​	要手写一个 Promise，那么就需要知道 Promise 都有哪些规范，并且一个一个实现它们。

- Promise 有三种状态：**pending、fulfilled、rejected**；默认状态是 pending；
- Promise 只能**从 pending 到 rejected，或者 从 pending 到 fulfilled**；
- Promise 状态一旦确认就不能改变；
- 当我们 `new Promise((resolve,reject) => {})` 时，被传入的函数会立即执行；因此，我们需要给 promise 传递一个立即执行的执行器（executor）；
- 执行器（executor）接受两个参数，`resolve` 和 `reject`；
- Promise 有一个 `value` 能够**保存成功状态的值**，值可能是 `undefined` 或 `thenable`（具有 then 方法的对象），或者是一个 promise 对象，无论何时去访问 Promise，都能得到这个 value；
- Promise 有一个 `reason` **保存失败状态的值**，无论何时去访问 Promise，都能得到这个值；
- Promise 必须有个 `then` 方法，这个方法接受两个参数，分别是 Promise 成功时的回调 `onFulfilled` 和 失败时的回调 `onRejected`
- 调用 `then` 方法时，如果 promise **已经成功，那么执行 `onFulfilled`**；
- 调用 `then` 方法时，如果 promise **已经失败，那么执行 `onRejected`**；
- 如果在调用 `then` 方法时抛出了异常，那么会把这个异常作为参数，传递给下一个 `then` 的失败回调，也就是 `onRejected` 方法；



# 基础实现

```js
class Promise {
    constructor(executor) {
        // 默认状态 pending
        this.status = 'pending'
        // 成功状态的默认值
        this.value = undefined
        // 失败状态的默认值
        this.reason = undefined
        
        // 调用此方法，会把 promise 的状态变为 fulfilled，并且保存成功状态的值
        let resolve = (value) => {
            // 状态为 PENDING 时才可以更新状态，防止 executor 中调用了两次 resovle/reject 方法
            if (this.status === 'pending') {
                // 将 promise 的状态变更为 fulfilled
            	this.status = 'fulfilled'
                // 保存成功状态的值
            	this.value = value
            }
        }
        
        // 调用此方法，会把 promise 的状态变为 rejected，并且保存失败原因 reason
        let reject = (reason) => {
            if (this.status === 'pending') {
                // 将 promise 状态变为 rejected
                this.status = 'rejected'
                // 保存失败的原因
                this.reason = reason
            }
        }
    }
    
    // then 方法
    then(onFulfilled, onRejected) {
        // 如果 promise 状态是 fulfilled，则调用 onFulfilled()
        if (this.status === 'fulfilled') {
            onFulfilled(this.value)
        }
        // 如果 promise 状态是 rejected，则调用 onRejected()
        if (this.status === 'rejected') {
            onRejected(this.reason)
        }
    }
}
```

​	以上代码在同步操作时已经可以正确运行了：

```js
const promise = new Promise((resolve, reject) => {
  resolve('成功');
}).then(
  (data) => {
    console.log('success', data)
  },
  (err) => {
    console.log('faild', err)
  }
)
// 输出 “success 成功”
```



# 支持异步执行器的实现

​	但是，如果传入的是一个异步的操作，如下：

```js
const promise = new Promise((resolve, reject) => {
  // 传入一个异步操作
  setTimeout(() => {
    resolve('成功');
  },1000);
}).then(
  (data) => {
    console.log('success', data)
  },
  (err) => {
    console.log('faild', err)
  }
)
```

​	测试后会发现，promise 没有任何反应；

​	这是因为，我们传入的 setTimeout 是一个宏任务，根据事件循环的机制，会被加入任务队列，等待主线程执行完毕以后，再去执行 setTimeout 中的回调。

​	所以，`then()` 方法调用的时候，`setTimeout() ` 中的 `resolve()` 还没有被执行，promise 仍然是 `pending` 状态，就不会去执行 `then()` 传入的方法。

​	所以，**如果当 `then()` 方法被调用时，当前状态是 `pending` 的话，需要先将成功和失败的回调储存起来**，在执行器（executor）中的 `resolve()` 或者 `reject()` 方法被调用的时候，再去执行对应的被储存起来的回调。

​	代码如下： 

```js
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';

class Promise {
  constructor(executor) {
    this.status = PENDING;
    this.value = undefined;
    this.reason = undefined;
    // 存放成功的回调
    this.onResolvedCallbacks = [];
    // 存放失败的回调
    this.onRejectedCallbacks= [];

    let resolve = (value) => {
      if(this.status ===  PENDING) {
        this.status = FULFILLED;
        this.value = value;
        // 依次将对应的函数执行
        this.onResolvedCallbacks.forEach(fn=>fn());
      }
    } 

    let reject = (reason) => {
      if(this.status ===  PENDING) {
        this.status = REJECTED;
        this.reason = reason;
        // 依次将对应的函数执行
        this.onRejectedCallbacks.forEach(fn=>fn());
      }
    }
	
    // 使用 try...catch... 是为了避免， executor 中如果执行出错则直接交给 reject 处理
    try {
      executor(resolve,reject)
    } catch (error) {
      reject(error)
    }
  }

  then(onFulfilled, onRejected) {
    if (this.status === FULFILLED) {
      onFulfilled(this.value)
    }

    if (this.status === REJECTED) {
      onRejected(this.reason)
    }

    if (this.status === PENDING) {
      // 如果promise的状态是 pending，需要将 onFulfilled 和 onRejected 函数存放起来，等待状态确定后，再依次将对应的函数执行
      // [() => { onFulfilled(this.value) }, () => { onFulfilled(this.value) }]
      this.onResolvedCallbacks.push(() => {
        onFulfilled(this.value)
      });

      this.onRejectedCallbacks.push(()=> {
        onRejected(this.reason);
      })
    }
  }
}
```



# then 的链式调用&值穿透特性

- `then` 方法能够链式调用；
- `then` 方法能够实现值的穿透，`promise.then().then()`，后面的 `then` 方法能够得到之前 `then` 返回的值；
- then 的参数 `onFulfilled` 和 `onRejected` 可以缺省，如果 `onFulfilled` 或者 `onRejected`不是函数，将其忽略，且依旧可以在下面的 then 中获取到之前返回的值；「规范 Promise/A+ 2.2.1、2.2.1.1、2.2.1.2」
- promise 可以 then 多次，每次执行完 promise.then 方法后返回的都是一个“新的promise"；「规范 Promise/A+ 2.2.7」
- 如果 then 的返回值 x 是一个普通值，那么就会把这个结果作为参数，传递给下一个 then 的成功的回调中；
- 如果 then 中抛出了异常，那么就会把这个异常作为参数，传递给下一个 then 的失败的回调中；「规范 Promise/A+ 2.2.7.2」
- 如果 then 的返回值 x 是一个 promise，那么会等这个 promise 执行完，promise 如果成功，就走下一个 then 的成功；如果失败，就走下一个 then 的失败；如果抛出异常，就走下一个 then 的失败；「规范 Promise/A+ 2.2.7.3、2.2.7.4」
- 如果 then 的返回值 x 和 promise 是同一个引用对象，造成循环引用，则抛出异常，把异常传递给下一个 then 的失败的回调中；「规范 Promise/A+ 2.3.1」
- 如果 then 的返回值 x 是一个 promise，且 x 同时调用 resolve 函数和 reject 函数，则第一次调用优先，其他所有调用被忽略；「规范 Promise/A+ 2.3.3.3.3」



# 完整代码

```js
const PENDING = 'pending'
const FULFILLED = 'fulfilled'
const REJECTED = 'rejected'

class Promise {
  constructor(executor) {
    this.status = PENDING
    this.value = undefined
    this.reason = undefined
    this.onFulfilledCallbacks = []
    this.onRejectedCallbacks = []

    function resolve(value) {
      this.status = FULFILLED
      this.value = value
      this.onFulfilledCallbacks.forEach((fn) => {
        fn()
      })
    }

    function reject(reason) {
      this.status = REJECTED
      this.reason = reason
      this.onRejectedCallbacks.forEach((fn) => {
        fn()
      })
    }

    try {
      executor(resolve, reject)
    } catch (error) {
      reject(error)
    }
  }

  then(onFulfilled, onRejected) {
    onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value
    onRejected = typeof onRejected === 'function' ? onRejected : error => {
      throw error
    }
    let promise2 = new Promise((resolve, reject) => {
      if (this.status === PENDING) {
        this.onFulfilledCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onFulfilled(this.value)
              resolvePromise(promise2, x, resolve, reject)
            } catch (error) {
              reject(error)
            }
          })
        })

        this.onRejectedCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onRejected(this.reason)
              resolvePromise(promise2, x, resolve, reject)
            } catch (error) {
              reject(error)
            }
          })
        })
      }

      if (this.status === FULFILLED) {
        setTimeout(() => {
          try {
            let x = onFulfilled(this.value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      }

      if (this.status === REJECTED) {
        setTimeout(() => {
          try {
            let x = onRejected(this.reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      }

    })
    return promise2
  }
}

function resolvePromise(promise2, x, resolve, reject) {
  let called

  if (promise2 === x) {
    return reject(new TypeError('Chaining cycle detected for promise!'))
  }
  // 处理 then 方法的回调onFulfilled/onRejected的 返回值 x 是个 promise 对象的情况
  // 如果这个 x 是个 promise 并且状态是 pending 的话，
  if (x instanceof Promise) {
    if (x.status === PENDING) {
      x.then((y) => {
        resolvePromise(promise2, y, resolve, reject)
      }, reject)
    } else {
      x.then(resolve, reject)
    }
    return
  }
  // 由于，不同库之间对 promise 的实现有可能不相同，这里是为了保证不同库之间的 promise 能够相互交互
  // 根据规范规定，onFulfilled/onRejected的返回值 x ，即使返回了一个带有then属性但并不遵循Promise标准的对象：
  // 比如说这个x把它then里的两个参数都调用了，或者是出错后又调用了它们，或者then根本不是一个函数，这些情况都尽可能正确处理。
  if ((typeof x === 'object' && x !== null) || (typeof x === 'function')) {
    try {
      // 这里的 then 可能是 promise 的 then 方法，也可能是普通对象的 then 属性，或者是一个函数
      let then = x.then
      if (typeof then === 'function') {
        then.call(x,
          y => {
            // called 用来保证，then 只能被调用一次
            if (called) return
            called = true
            resolvePromise(promise2, y, resolve, reject);
          },
          r => {
            if (called) return;
            called = true;
            reject(r);
          }
        )
      } else {
        resolve(x)
      }
    } catch (error) {
      if (called) return
      called = true
      return reject(error)
    }
  } else {
    resolve(x)
  }
}

```

