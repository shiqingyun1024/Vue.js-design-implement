   前文没有提到响应系统，响应系统也是Vue.js的重要组成部分，所以我们会花费大量篇幅介
绍。在本章中，我们讨论什么是响应式数据和副作用函数，然后尝试实现一个相对完整的响
应系统。在这个过程中，我们会遇到各种各样的问题，例如如何避免无限递归？为什么需要嵌套
的副作用函数？两个副作用函数之间会产生哪些影响？以及其他很多需要考虑的细节。接着，我
们会详细讨论与响应式数据相关的内容。我们知道Vue.js3采用Proxy实现响应式数据，这涉及
语言规范层面的知识。这部分内容包括如何根据语言规范实现对数据对象的代理，以及其中的一
些重要细节。接下来，我们就从认识响应式数据和副作用函数开始，一步一步地了解响应系统的
设计与实现。

## 4.1 响应式数据与副作用函数
副作用函数指的是会产生副作用的函数
```
副作用函数指的是会产生副作用的函数，如下面的代码所示：
function effect(){
    document.body.innerText = 'hello vue3'
}
   当effect函数执行时，它会设置body的文本内容，但除了effect函数之外的任何函数都可
以读取或设置body的文本内容。也就是说，effect函数的执行会直接或间接影响其他函数的执
行，这时我们说effect函数产生了副作用。副作用很容易产生，例如一个函数修改了全局变量，
这其实也是一个副作用，如下面的代码所示：
<!-- 全局变量 -->
let val = 1

function(){
    val = 2 // 修改全局变量，产生副作用。
}

理解了什么是副作用函数，再来说说什么是响应数据。假设在一个副作用函数中读取了某
个对象的属性：
const obj = { text:'hello world' }
function effect() {
    // effect 函数的执行会读取obj.text
    document.body.innerText = obj.text
}
如上面的代码所示，副作用函数effect会设置body元素的innerText属性，其值为obj.text,
当obj.text的值发生变化时，我们希望副作用函数effect会重新执行：
obj.text = ‘hello vue3’ // 修改obj.text的值，同时希望副作用函数会重新执行
这句代码修改了字段obj.text的值，我们希望当值变化后，副作用函数自动重新执行，如果
能实现这个目标，那么对象obj就是响应式数据。但很明显，以上面的代码来看，我们还做不到
这一点，因为obj是一个普通对象，当我们修改它的值时，除了值本身发生变化之外，不会有任何其他
反应。下一节中我们会讨论如何让数据变成响应式数据。
```
## 4.2 响应式数据的基本实现
```
接着上文思考，如何才能让obj变成响应式数据呢？通过观察我们能发现两点线索：
- 当副作用函数effect执行时，会触发字段obj.text的读取操作；（读取操作）
- 当修改obj.text的值时，会触发字段obj.text的设置操作；（设置操作）
如果我们能拦截一个对象的读取和设置操作，事情就变得简单了，当读取字段obj.text时，我们可以把副作用函数effect存储到一个“桶”里，如下所示：
执行effect()
    |
function effect() {
    document.body.innerText = obj.text
}
    |
触发读取操作
    |
存储副作用函数的“桶”（存储了effect）  
(将副作用函数存储到“桶”中)     

接着，当设置obj.text时，再把副作用函数effect从“桶”里取出并执行即可，如下所示：
obj.text = 'hello vue3'
    |
触发设置操作
    |
存储副作用函数的“桶”（存储了effect）
    |
取出effect并执行
（把副作用函数从“桶”内取出并执行）  

现在问题的关键变成了我们如何才能拦截一个对象属性的读取和设置操作。在ES2015之前，只能通过
object.defineProperty函数来实现，这也是Vue.js2所采用的方式。在ES2015+中，我们可以使
用代理对象proxy来实现，这也是Vue.js3所采用的方式。

接下来我们就根据如上思路，采用Proxy来实现：

<!-- 存储副作用函数的桶 -->
const bucket = new Set()

// 原始数据
const data = { text:'hello world' }
// 对原始数据的代理
const obj = new Proxy(data,{
    // 拦截读取操作
    get(target,key) {
        // 将副作用函数effect添加到存储副作用函数的桶中
        bucket.add(effect)
        // 返回属性值
        return target[key]
    },

    // 拦截设置操作
    set(target,key,newVal) {
        // 设置属性值
        target[key] = newVal
        // 把副作用函数从桶里面取出并执行
        bucket.forEach(fn=>fn())
        // 返回 true 代表设置操作成功
        return true
    }
})

function effect() {
    document.body.innerText = obj.text  // 记住这里是obj，使用的是代理之后的数据
}
effect()
setTimeout(function(){
    obj.text = 'hello vue3' // 记住这里是obj，使用的是代理之后的数据
},3000)

首先，我们创建了一个用于存储副作用函数的桶bucket，它是Set类型。接着定义了原始数据data，
obj是原始数据data的代理对象，我们分别设置了get和set拦截函数，用于拦截读取和设置操作。
当读取属性时将副作用函数effect添加到桶里，即bucket.add(effect),然后返回属性值；当设置
属性值时先更新原始数据，再将副作用函数从桶里取出并重新执行，这样我们就实现了响应式数据。可
以使用下面的代码来测试一下：
// 副作用函数
function effect(){
    document.body.innerText = obj.text  // 记住这里是obj，使用的是代理之后的数据
}
// 执行副作用函数，触发读取
effect()
// 1秒之后修改响应式数据(代理之后的数据)
setTimeout(()=>{
    obj.text = 'hello vue3' // 记住这里是obj，使用的是代理之后的数据
},1000)

在浏览器中运行上面这段代码，会得到期望的结果。

但是目前的实现还存在很多缺陷，例如我们直接通过名字（effect）来获取副作用函数，这种硬编码的
方式很不灵活。副作用函数的名字可以任意取，我们完全可以把副作用函数命名为myEffect，甚至是
一个匿名函数，因此我们要想办法去掉这种硬编码的机制。下一节会详细讲解这一点，这里大家只需要
理解响应式数据的基本实现和工作原理即可。
```
## 4.3 设计一个完善的响应系统
```
在上一节中，我们了解了如何实现响应式数据。但其实在这个过程中我们已经实现了一个微型响应
系统，之所以说“微型”，是因为它还不完善，本节我们将尝试构造一个更加完善的响应系统。
从上一节的例子中不难看出，一个响应系统的工作流程如下：
- 当读取操作发生时，将副作用函数收集到“桶”中；
- 当设置操作发生时，从“桶”中取出副作用函数并执行。
看上去很简单，但需要处理的细节还真不少。例如在上一节的实现中，我们硬编码了副作用函数的名字
（effect），导致一旦副作用函数的名字不叫effect，那么这段代码就不能正确地工作了。而我们希
望的是，哪怕副作用函数是一个匿名函数，也能够被正确的收集到“桶”中。为了实现这一点，我们需要
提供一个用来注册副作用函数的机制，如以下代码所示：
// 用一个全局变量存储被注册的副作用函数
let activeEffect;
// effect函数用于注册副作用函数
function effect(fn){
    // 当调用effect注册副作用函数时，将副作用函数fn赋值给activeEffect
    activeEffect = fn
    // 执行副作用函数
    fn()
}

首先，定义了一个全局变量activeEffect，初始值是undefined，它的作用是存储被注册的
副作用函数。接着重新定义了effect函数，它变成了一个用来注册副作用函数的函数，effect函
数接收一个参数fn，即要注册的副作用函数。我们会如下所示使用effect函数：
effect(
    // 一个匿名的副作用函数
    ()=>{
        document.body.innerText = obj.text
    }
)

可以看到，我们使用一个匿名的副作用函数作为effect函数的参数。当effect函数执行
时，首先会把匿名的副作用函数赋值给全局变量activeEffect。接着执行被注册的匿名
副作用函数fn，这将会触发响应式数据obj.text的读取操作，进而触发代理对象Proxy的
get拦截函数：
const obj = new Proxy(data,{
    get(target,key){
      // 将activeEffect中存储的副作用函数收集到“桶”中
      if(activeEffect){ // 新增
        bucket.add(activeEffect) // 新增
      } // 新增
      return target[key]
    },
    set(target,key,newVal){
      target[key] = newVal
      bucket.forEach(fn=>fn())
      return true
    }
})
如上面的代码所示，由于副作用函数已经存储到了activeEffect中，所以在get拦截函数内
应该把activeEffect收集到‘桶’中，这样响应系统就不依赖副作用函数的名字了。（**？？解决了
依赖副作用函数的名字的问题。？？**）
但如果我们再对这个系统稍加测试，例如在响应式数据obj上设置一个不存在的属性时：
effect(
    <!-- 匿名副作用函数 -->
    ()=>{
      console.log('effect run') // 会打印2次
      document.body.innerText = obj.text
    }
)
```
