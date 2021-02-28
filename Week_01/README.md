- `{}` 花括号可以提供一个作用域；
- JavaScript 的异步机制
  - callback
  - Promise
  - yield/generator （出现的比 async/await 早，出现的目的是为了模拟 async/await；generator 返回一个 iterator）
  - async/await （基于 Promise 的一种语法支持封装，实际内部运行也是基于 Promise 去管理和运行的）
- async/generator 与 for await of 的用法


eg. async/generator 和 for await of 简单用法
````javascript
async function counter () {
  let i = 0
  while (true) {
    await sleep(1000)
    yield i++
  }
}

(await function () {
  for await (let v of counter()) {
    console.log(v)
  }
})()
```