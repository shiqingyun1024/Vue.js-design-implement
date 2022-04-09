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
      console.log('effect run') // 会打印2次，因为在set会执行两次
      document.body.innerText = obj.text
    }
)
setTimeout(()=>{
    <!-- 副作用函数中并没有读取notExist属性的值 -->
    obj.notExist = 'hello vue3'
},1000)

可以看到，匿名副作用函数内部读取了字段obj.text的值，于是匿名副作用函数与字段obj.text
之间会建立响应联系。接着，我们开启了一个定时器，一秒钟后为对象obj添加新的notExist属性。
我们知道，在匿名副作用函数内并没有读取obj.notExist属性的值，所以理论上，字段obj.notExist
并没有与副作用函数建立响应联系，因此，定时器内语句的执行不应该触发匿名副作用函数重新执行。
但如果我们执行上述这段代码就会发现，定时器到时后，匿名副作用函数却重新执行了，这是不正确的。
为了解决这个问题，我们需要重新设计一下“桶”的数据结构。
在上一节的例子中，我们使用一个Set数据结构作为存储副作用函数的“桶”。导致该问题的根本原因是，
我们没有在**--副作用函数与被操作的目标字段之间建立明确的联系--**。例如当读取属性时，无论
读取的是那一个属性，其实都一样，都会把副作用函数收集到“桶”里；当设置属性时，无论设置的是哪个
属性，也都会把“桶”里的副作用函数取出并执行。副作用函数与被操作的字段之间没有明确的联系。解决
方法很简单，只需要在副作用函数与被操作的字段之间建立联系即可，这就需要我们重新设计“桶”的数据
结构，而不能简单地使用一个Set类型的数据作为“桶”了。
那应该设计怎样的数据结构呢？在回答这个问题之前，我们需要仔细观察下面的代码：
effect(function effectFn(){
    document.body.innerText = obj.text
})
在这段代码中存在三个角色：
- 被操作（读取）的代理对象obj；
- 被操作（读取）的字段名text；
- 使用effect函数注册的副作用函数effectFn。
如果用target来表示一个代理对象所代理的原始对象，用key来表示被操作的字段名，用
effectFn来表示被注册的副作用函数，那么可以为这三个角色建立如下关系：
target
   |
    --- key
         |
          --- effectFn
这是一种树型结构，下面举几个例子来对其进行补充说明。
effect(function effectFn1(){
    obj.text
})
effect(function effectFn2(){
    obj.text
})
那么关系如下：
target
   |
    --- text
         |
          --- effectFn1
              effectFn2
如果一个副作用函数中读取了同一个对象的两个不同属性：
effect(function effectFn(){
    obj.text1
    obj.text2
})
那么关系如下：
target
   |
    --- text1
         |
          --- effectFn
   |
    ——— text2
          |
           ——- effectFn  
如果在不同的副作用函数中读取了两个不同对象的不同属性：
effect(function effectFn1(){
    obj.text1
})
effect(function effectFn2(){
    obj.text2
})  
那么关系如下：
target
   |
    --- text1
         |
          --- effectFn1
   |
    ——— text2
          |
           ——- effectFn2   
总之，这其实就是一个树型数据结构。这个联系建立起来之后，就可以解决前文提到的问
题了。拿上面的例子来说，如果我们设置了obj.text2的值，就只会导致effectFn2函
数重新执行，并不会导致effectFn1函数重新执行。
   接下来我们尝试用代码来实现这个新的“桶”。首先，需要使用weakMap代替Set作为
“桶”的数据结构。  
// 存储副作用函数的桶
const bucket = new WeakMap()
然后修改get/set拦截器代码：
const obj = new Proxy(data,{
    // 拦截读取操作
    get(target,key){
        // 没有activeEffect，直接return
        if(!activeEffect) return
        // 根据target从“桶”中取得depsMap，它也是一个Map类型：key --> effects
        let depsMap = bucket.get(target) // 有个疑问，这个target是什么？
        // 如果不存在depsMap，那么新建一个Map并与target关联
        if(!depsMap){
          bucket.set(target,( depsMap = new Map()))
        }
        // 再根据key从depsMap中取得deps，它是一个Set类型，
        // 里面存储着所有与当前key相关联的副作用函数：effects
        let deps = depsMap.get(key)
        // 如果deps不存在，同样新建一个Set并与key关联
        if(!deps){
            depsMap.set(key,(deps = new set()))
        }
        // 最后将当前激活的副作用函数添加到“桶”里
        deps.add(activeEffect)
        // 返回属性值
        return target[key]
    },
    // 拦截设置操作
    set(target,key,newVal){
        // 设置属性值
        target[key] = newVal
        // 根据target从桶中取得desMap，它是key --> effects
        const depsMap = bucket.get(target)
        if(!depsMap) return
        // 根据key取得所有副作用函数effects
        const effects = depsMap.get(key)
        // 执行副作用函数
        effects && effects.forEach(fn=>fn())
    }
})
从这段代码可以看出构建数据结构的方式，我们分别使用了WeakMap、Map和Set：
- WeakMap由target --> Map 构成；
- Map由 key --> Set 构成。
其中WeakMap的键是原始对象target，WeakMap的值是一个Map实例，而Map的键是原
始对象target的key，Map的值是一个由副作用函数组成的Set。它们的关系如图4-3所
示。
WeakMap
  |
 key 
 target-1 ---Value--- Map
 target-2              |  
 Key-n                  --- key                依赖集合
                            key-1  -- Value -- Set
                            key-2              effect-1
                              .                effect-2
                              .                  .
                            key-n              effect-n
            图4-3 WeakMap、Map和Set之间的关系
为了方便描述，我们把图4-3中的Set数据结构所存储的副作用函数集合称为key的依赖
集合。
搞清了它们之间的关系，我们有必要解释一下这里为什么要使用WeakMap，这其实涉及
WeakMap和Map的区别，我们用一段代码来讲解：
const map = new Map();
const weakmap = new WeakMap();
(function(){
    const foo = { foo:1 };
    const bar = { bar:2 };
    map.set(foo,1);
    weakmap.set(bar,2)
})()
首先我们定义了map和weakmap常量，分别对应Map和WeakMap的实例。接着定义了一个
立即执行函数表达式（IIFE），在函数表达式内部定义了两个对象：foo和bar，这两个
对象分别作为map和weakmap的key。当该函数表达式执行完毕之后，对于对象foo来说，它仍然
作为map的key被引用着，因此垃圾回收器（grabage collector）不会把它从内存中移除，我们
仍然可以通过map.keys打印出对象foo。然而对于对象bar来说，由于WeakMap的key是弱引用，它
不影响垃圾回收期器的工作，所以一旦表达式执行完毕，垃圾回收器就会把对象bar从内存中移除，并且
我们无法获取weakmap的key值，也就无法通过weakmap取得了对象bar。
   简单地说，WeakMap对key是弱引用，不影响垃圾回收器的工作。据这个特性可知，一旦key
被垃圾回收，那么对应的键和值就访问不到了。所以WeakMap经常用于存储那些只有当key所引用的对象
存在时（没有被回收）才有价值的信息，例如上面的场景中，如果target对象没有任何引用了，说明用
户侧不再需要它了，这时垃圾回收器会完成回收任务。但如果使用Map来代替WeakMap，那么即使用户侧
的代码对target没有任何引用，这个target也不会被回收，最终可能导致内存溢出。
   最后，我们对上文中的代码做一些封装处理。在目前的实现中，当读取属性值时，我们直接在get拦
截函数里编写把副作用函数收集到“桶”里的这部分逻辑，但更好的做法是将这部分逻辑单独封装到一个track
函数中，函数的名字叫track是为了表达追踪的含义。同样，我们也可以把触发副作用函数重新执行的逻辑封装
到trigger函数中：
const obj = new Proxy(data,{
    // 拦截读取操作
    get(target,key){
        // 将副作用函数activeEffect添加到存储副作用函数的桶中
        track(target,key)
        // 返回属性值
        return target[key]
    },
    // 拦截设置操作
    set(target,key,newVal){
        // 设置属性值
        target[key] = newVal
        // 把副作用函数从桶中取出并执行
        trigger(target,key)
    }
})

// 在get拦截函数内调用track函数追踪变化
function track(target,key){
   // 没有activeEffect，直接return
   if(!activeEffect) return
   let depsMap = bucket.get(target)
   if(!depsMap){
       bucket.set(target,(deps=new Map()))
   }
   let deps = depsMap.get(key)
   if(!deps){
       depsMap.set(key,(deps=new Set()))
   }
   deps.add(activeEffect)
}
// 在set拦截函数内调用trigger函数触发变化
function trigger(target,key){
   const depsMap = bucket.get(target)
   if(!depsMap) return
   const effects = depsMap.get(key)
   effects && effects.forEach(fn=>fn())
}
如以上代码所示，分别把逻辑封装到track和trigger函数内，这能为我们带来极大的
灵活性。
```
## 4.4 分支切换与cleanup
```
首先，我们需要明确分支切换的定义，如下面的代码所示：
const data = { ok:true, text:"hello world" }
const obj = new Proxy(data,{/*....*/})
effect(function effectFn(){
    document.body.innerText = obj.ok?obj.text:'not'
})
``` 