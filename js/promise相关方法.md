# 要求
请你编写一个异步函数 promisePool ，它接收一个异步函数数组 functions 和 池限制 n。它应该返回一个 promise 对象，当所有输入函数都执行完毕后，promise 对象就执行完毕。
池限制 定义是一次可以挂起的最多 promise 对象的数量。promisePool 应该开始执行尽可能多的函数，并在旧的 promise 执行完毕后继续执行新函数。promisePool 应该先执行 functions[i]，再执行 functions[i + 1]，然后执行 functions[i + 2]，等等。当最后一个 promise 执行完毕时，promisePool 也应该执行完毕。

```js
// 输入：
const functions = [
  () => new Promise(res => setTimeout(res, 300)),
  () => new Promise(res => setTimeout(res, 400)),
  () => new Promise(res => setTimeout(res, 200))
]
// n = 1
// 解释：
// 在 t=0 时，执行第一个函数。池大小为1。
// 当 t=300 时，第一个函数执行完毕后，执行第二个函数。池大小为 1。
// 当 t=300+400=700 时，第二个函数执行完毕后，执行第三个函数。池大小为 1。
// 在 t=300+400+200=900 时，第三个函数执行完毕后。池大小为 0，因此返回的 Promise 也执行完成。
```

# 思路
如果不做限制，按照正常的流程去执行这些 promise，并且等待他们全部执行完成以后返回值的话，可以像下面这样：
```js
  const result = []
  for (const promise of promises) {
    promise().then(res => {
      result.push(res)
      if (result.length === promises.length) {
        return result
      }
    })
  }
```
由于现在需要：
1. 限制每次并发数为 n；
2. 出一个promise，就立即进一个 promise;
我们可以考虑用一个队列来控制:
```js
const promisePool = async() => {
  const queue = new Set()
  const result = []
  for (const promise of promises) {
    // 当这个 job 被执行的时候，也就是 queue 中的 promise 被执行的时候
    // 这里我们需要在 then 方法里把这个 promise 的执行结果保存下来，并把这个 job 从 queue 中删除
    const job = promise().then(res => {
      result.push(res)
      queue.delete(job)
    })
    // 将 job 添加到 queue 中
    queue.add(job)
    // 如果我们队列找中的 promise 比限制的并发数大，那么说明我们这一批 promise 就可以发送了
    if (queue.size >= n) {
      // 这里利用了 await + race 方法的特性，将队列里的所有 promise 都发送
      // 并且利用 await 来阻塞，等待有任意一个 promise 处理完之后才会继续后面的循环逻辑；
      await Promise.race(queue)
    }
  }
   // 执行完所有任务后才返回执行结果
  await Promise.allSettled(queue);
  return result
}
```


# 实现 Promise.all
首先来看一下 `Promise.all([]).then(res => {})` 的用法：
1. 它接收一个可迭代元素；
2. 它会等待等待所有 promise 都完成或者 第一个promise 失败
3. 如果入参是非 promise 对象，那么会被原样返回；
4. 它的返回值也是一个数组，返回数组的元素的顺序和入参的顺序一直；
```js
const promiseAll = (promises) => {
  const result = []
  const count = 0
  return new Promise((resolve, reject) => {
    for(const promise of promises) {
      // 用 Promise.resolve 包一层做兼容，能把非 promise 对象尽量转成 promise 对象
      Promise.resolve(promise).then((value, index) => {
        // 按顺序返回
        result[index] = value
        count++
        // 如果count长度和入参长度一直，说明所有 promise 都处理完毕了
        // 那么久调用 resolve 方法将结果返回
        if (count === promises.length - 1) {
          resolve(result)
        }
      }, reason => {
        reject(reason)
      })
    }
  })
}
```
# 实现 Promise.allSellted
和 `Promise.all` 方法很像，有几点不同：
1. 它会等待所有 promise 完成，不管成功还是拒绝；
2. 返回的值是个对象，包含了这个 promise 的状态，以及成功的值或者失败的原因
```js
const promiseAllSellted = (promises) => {
  const result = []
  const count = 0
  return new Promise((resolve, reject) => {
    for(const promise of promises) {
      // 用 Promise.resolve 包一层做兼容，能把非 promise 对象尽量转成 promise 对象
      Promise.resolve(promise).then((value, index) => {
        // 按顺序返回
        result[index] = {
          status: 'fulfilled',
          value
        }
        count++
        // 如果count长度和入参长度一直，说明所有 promise 都处理完毕了
        // 那么久调用 resolve 方法将结果返回
        if (count === promises.length - 1) {
          resolve(result)
        }
      }, reason => {
        result[index] = {
          status: 'rejected',
          reason
        }
        if (count === promises.length - 1) {
          reject(result)
        }
      })
    }
  })
}
```