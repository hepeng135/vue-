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
        this.getter=exoOrFn();
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