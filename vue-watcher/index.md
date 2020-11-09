## 什么是Watcher
在Vue中，我们数据中的每个属性都对应一个Dep(收集)实例，那么我们在收集什么，那就是Watcher实例，当属性发生变化的时候，这个属性对应的dep将
通知watcher进行更新操作，然后就有个新旧虚拟DOM比较，做UI更新。


在Vue中我们使用watcher时，通常都是使用$watch这个实例方法，就算我们在组件选项中配置了watch选项，其实也是调用了$watch进行实现的，
具体用法如下
```
vm.$watch(expOrFn,callbcak,[options])
exOrFn:可以是一个属性访问路径，也可以是一个函数
callback:函数，当对应观察的属性发生变化时，我们的回调函数。
options: 配置对象，一般可配置  deep:true  深度观察   immediate:true  以当前表达式的值立马调用回调函数
改方法还有个返回值unwatch，函数，用于取消监听

```
我们先看下Vue源码中$watch是怎么实现的
```
Vue.prototype.$watch=function(expOrFn,callBack,options){
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if(options.immediate){
        cb.call(this,watcher.value)
    }
     return function unwatchFn () {
        watcher.teardown()
    }
}

```
上述代码中，我们在配置相中新增呢user配置表示当前watcher是用户自己创建的，同时实现当前immediate为true时直接调用
回调函数，然后返回一个函数，用于取消监听，因为Vue中为了实现数据依赖收集、数据更新依赖触发也去实例化watcher,所有我们
用user去区分当前是否用户自己创建的watch。

我们在介绍Vue实现数据依赖收集、数据更新依赖触发中实现过一个简单的Watcher,代码如下，
````
class Watcher{
    constructor(expOrFn,callBack,options){
        this.getter=expOrFn;
        this.get();    
    }
    get(){
        Dep.target=this;
        this.getter();
        Dep.target=null;
    }
    update(){
        //更新
    }
}

````
上面的代码必然不能满足要求，我们下面将一点点的去完善他。

我们需要实现一个原型方法teardown，实现取消监听，其本质找到收集这个watcher的dep并从这些dep中将这个watcher去掉。
我们需求修改dep和watcher才能知道watcher被哪些dep收集了
```
class Watcher{
    constructor(vm,expOrFn,callBack,options){
        //因为expOrFn可能为函数或者字符串，所以这个初始化需要修改下
        this.getter=Object.prototype.toString.call(expOrFn)==='Function' ? expOrFn : parsePath(expOrFn);
        
        this.value=this.get();
        this.deps=[];
        this.depIds=[];
    }
    get(){
        Dep.target=this;
        let value=this.getter.call(vm,vm);
        Dep.target=null;
        return value
    }
    addDep(dep){
        //保证收集该watcher的dep不存在重复的
        if(this.depIds.indexOf(dep.id)<0){
            this.deps.push(dep);
            this.depIds.push(dep.id);
            dep.addSub(this);
        }
    }    

    update(){}
    
    teardown(){
        for(let i=0;i<this.dep.length;i++){
            this.dep[i].removeSub(this);
        }
    }
    
}

//修改对应的dep
let uid=0
class Dep{
    //code...
    constructor(){
        this.id=uid++
        this.subs=[]
    }
    addSub(watcher){
        this.subs.push(watcher)
    }
    depend(){
        if(Dep.target){
            Dep.target.addDep(this);
        }
    },
    removeSub(watcher){
        remove(this.subs,watcher)
    }
}
function remove(array,target){
    array.filter(function(item){
        if(item===target){
            return false
        }else{
            return true
        }
    })
}

```
上述代码中我们将watcher和dep互相引用，当属性的getter触发收集时，我们这个时候在通过Dep.target调用watcher的实例
方法addDep,将收集这个watcher的dep添加到watcher的deps属性中，并且通过dep.id来防止重复收集，我们在用dep.addSub，将
这个watcher收集到dep的subs属性中，这样就实现呢  dep中存在收集的watcher，watcher中存在收集他的dep，因为当属性变化时触发setter
,然后通过dep去通知watcher，然后执行更新，那么当我们需要取消某个属性的watch时直接将这个watcher从每个之前收集过他的dep中去掉就好，
所以我们执行会循环收集过这个watch的dep集合，然后执行dep.removeSub,在dep中的subs中找到对应的watch去掉就好。


因为watcher在实例化时，参数expOrFn可以为表单式（访问数据的属性eg:a.b.c） 或者为一个函数，我们暂时只处理了函数，现在我们
需要处理下当为表达式时。

通过watcher的原型方法get(),我们知道通过expOrFn需要得到一个函数，在这个函数中我们需要进行访问属性的操作，从而触发getter进行收集依赖，上面
我们通过判断当是表达式时我们调用parsePath函数，下面我们来实现以下

```
function parsePath(path){
    let paths=path.split('.');
    return function(obj){
        let value=obj;
        for(let i=0;i<paths.length;i++){
            if(value) value=value[paths[i]];
        }
        return value;
    }
}

```
上述代码，我们解析传进来的字符串表达式，然后去访问数据，得到最终的value；
假如传的是个函数，在这个函数中访问到的属性都将被监听,因为我们每访问一个属性都将触发getter进行依赖收集。

```
data(){
    return {
        name:'hepeng',
        age:'22'
    }
}
this.$watch(function(){
    return this.age+this.name
})

```
上面的代码，我们的表达式是一个函数，在里面访问了name和age，这种情况下，watcher会收集到name的Dep和age的Dep，这两个Dep
也同时收集了watcher，当name和age任何一个发生变化时都会通知这个watcher。

下面我们需要看看当dep去通知watcher更新时，watcher是怎么做的，当属性被改变时触发setter这个时候会执行对应属性的dep.notify,如下
```
class Dep{
    //code....

    notify(){
        let watchers=this.subs.slice();
        watchers.sort((a,b)=>a.id-b.id);
        for(let i=0;i<watchers.length;i++){
            watchers.update()
        }
    }
}
```
上述代码，我们新建一个对应dep收到的watcher副本，然后进行排序，因为组件在更新时都是由父到子这个顺序进行的。随后我们循环watchers,
调用update方法进行更新，


我们用$watch去创建一个监听的时候，可以配置options，即设置immediate和deep，下面我讲详细看看这两个配置属性

immediate值为Boolean类型，默认值为false，设为true既表示回调将在侦听开始之后被立即调用，在最前面介绍$watch的时候已经看过immediate是怎么实现的，很简单
就是当immediate为true的时候理解执行一下传入$watch的回调函数，既this.cb.call(vm,watcher.value);

deep值为Boolean类型，默认值为false，设为true时表示该回调会在任何被侦听的对象的 property 改变时被调用，不论其被嵌套多深,下面我们看看是怎么实现的。

```
//首先在Watcher.prototype.get中调用traverse，
get(){
    Dep.target=this;
    let value=this.getter.call(vm,vm);
    if(thia.deep){
        traverse(value)
    }
    Dep.target=null;
    return value
}


```
首先我们知道Watcher.prototype.get 这个方法其实就是在执行this.getter(),用来访问检测的对象属性，触发getter从而进行收集依赖，这样在数据变化的时候触发setter
从而执行对应的watcher和回调函数，当deep为true时，我们调用traverse方法，并传入对应的value，注意这个动作实在Dep.target=null之前进行的，那么我们可以猜测一下
traverse方法无非就是将value这个对象中所有的属性都访问一遍，这样才能触发getter从而进行收集依赖，先来看看Vue是怎么实现的

```
function traverse (val: any) {
    _traverse(val, seenObjects)
    seenObjects.clear()
}
function _traverse (val: any, seen: SimpleSet) {
    let i, keys
    const isA = Array.isArray(val)
    if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
        return
    }
    //防止重复访问
    if (val.__ob__) {
        const depId = val.__ob__.dep.id
        if (seen.has(depId)) {
            return
        }
        seen.add(depId)
    }
    //递归访问value中的每一个属性
    if (isA) {
        i = val.length
        while (i--) _traverse(val[i], seen)
    } else {
        keys = Object.keys(val)
        i = keys.length
        while (i--) _traverse(val[keys[i]], seen)
    }
}

```
和我们猜想的一样上述代码做的最重要的一件事就是递归访问value对象中的每一个属性value[key[i]]。

再来看看Watcher是怎么做更新操作的，当属性发生变化时会触发属性对应的setter，在setter中我们通过dep.notify()去通知更新，dep.notify()中循环这个dep
收集到的watcher实例，然后执行watch.update();这个时候Vue会维护一个更新队列，当下一次EventLoop时进行更新

```
class Watcher{
    //code.....

    update(){
        queueWatcher(this);
    }

    run(){
        if (this.active) {
            const value = this.get();
            if(value!==this.value || isObject(value) || this.deep){
                const oldValue = this.value
                this.value = value
                this.cb.call(this.vm, value, oldValue)
            }
        } 
    }
}
```
通过queueWatcher函数，我们将这个EventLoop内触发的依赖全部收集起来。

```
let has={}; //存储wathcer的id，防止重复添加
let queue=[]; //当前队列数组
let waiting = false   //当前队列是否执行  当为true时表示，等待被执行中
let flushing = false   //表示当前队列的状态

function queueWatcher (watcher: Watcher) {
    const id = watcher.id
    if (has[id] == null) { //记录当前watcher的id。，确保watcher没有收集重复
        has[id] = true
        if (!flushing) {//flushing  代表当前队列正在收集阶段。
            queue.push(watcher)
        }
        // queue the flush  队列刷新
        if (!waiting) {  //当前队列的状态是否等待中。默认false
            waiting = true
            nextTick(flushSchedulerQueue)
        }
    }
}
function flushSchedulerQueue () {
    flushing = true
    let watcher, id
    //对队列中的watcher进行排序，根据watcher.id，从小到大进行排序，因为组件更新是从父级到子级这个顺序更新的
    queue.sort((a, b) => a.id - b.id)
    for (index = 0; index < queue.length; index++) {
        watcher = queue[index]
        if (watcher.before) {
            watcher.before()
        }
        id = watcher.id
        has[id] = null
        watcher.run()
    }
    //重置一些状态，在队列收集中用到的一些状态 has index waittng flushing
    resetSchedulerState()
}
```

在queueWatcher函数我们会收集到这个dep对应的所有watcher，在里面我们做一些限制，已经存在与queue中的watcher不允许重复添加，然后通过waiting变量的限制
只执行一次nextTick函数，并传入flushSchedulerQueue函数作为参数。flushSchedulerQueue函数实际上就是执行队列中的watcher，至于什么时候执行，我们需要看下
nextTick函数

```
const p = Promise.resolve()
let timerFunc = () => {
    p.then(flushCallbacks)  
}
function nextTick(cb){
    let _resolve
    callbacks.push(() => {
    if (cb) {
        cb.call(ctx)
    } else if (_resolve) {
        _resolve(ctx)
    }
    })
    if (!pending) { //默认false
        pending = true
        timerFunc()
    }
}
function flushCallbacks () {
    pending = false
    const copies = callbacks.slice(0)
    callbacks.length = 0
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}
```
nextTick将函数收集到callbacks中，然后调用timerFunc。timerFunc在Promise异步中执行，这就实现了下一次EventLoop的时候统一更新UI视图。














