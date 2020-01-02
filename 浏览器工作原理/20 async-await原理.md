`async`/`awiat`提供了在不阻塞主线程的情况下使用同步代码实现异步访问资源的能力，并且使得代码逻辑更加清晰。

#### 1. 协程

协程是一种比线程更加轻量级的存在。

可以把协程看成跑在线程上的任务，一个线程可以存在多个协程，但在协程上只能同时执行一个任务。如果从A协程启动B协程，那么称A协程为B协程的父协程。

最重要的是，协程不是被操作系统内核管理，而是完全由程序控制哦，也就是用户态执行。这样带来的好处是性能得到了很大的提升，不会像切换线程那样消耗资源。

`generator`函数执行时：

+ `gen`协程和父协程在主线程上交互执行，而不是并发执行，他们的切换是通过`gen.next`和`yield`来配合的。
+ 当在`gen`协程中调用了`yield`方法时，`js`引擎会保存`gen`协程当前的调用栈，并恢复父协程的调用栈信息。同样，在父协程执行`gen.next`时，会保存父协程的调用栈信息，同时恢复`gen`协程的调用栈。

其实，生成器就是协程的一种实现方式。

#### 2. `async`

`async`是一个通过异步执行并隐式返回一个Promise的函数。

异步执行+隐式返回promise

#### 3. await

```javascript
async function foo() {
  console.log(1)
  let a = await 100
  console.log(a)
  console.log(2)
}
console.log(0)
foo()
console.log(3)
```

以该代码为例

1. 首先代码在主线程上执行，打印0；
2. foo执行，保存父协程调用栈，开启foo协程；
3. 打印1；
4. 执行到await 100，会默认创建一个Promise对象

```javascript
let promise_ = new Promise((resolve) => {
  resolve(100)
})
```

5. 在executor函数中执行了resolve函数，js引擎会把这个任务提交到微任务队列，然后暂停当前协程的执行，把主线程的控制权交给父协程，同时把该promise_对象返回给父协程。（即async函数返回的promise对象）；
6. 父协程继续执行，打印3；
7. 父协程执行完毕，执行微任务队列；
8. 微任务队列中有resolve(100)等待执行，执行到这里，会执行promise_.then的回调函数

```javascript
promise_.then(value => {
  //回调函数被激活后，将主线程的控制权交给foo协程，并把value的值交给foo协程
})
```

9. a被赋值100，打印100,2。