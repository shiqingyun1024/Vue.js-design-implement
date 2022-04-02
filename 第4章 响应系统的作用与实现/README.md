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

