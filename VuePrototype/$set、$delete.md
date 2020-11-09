
#### Vue.prototype.$set
$set：向响应式对象中添加一个property,并确保这个property同样是响应式的，且触发视图更新。
我们知道Vue的响应式只能用作提前声明好的对象属性上，动态的向一个响应式对象添加一个属性并不能保证这个新增加的属性也是响应式的，所有我们就
拥有了$set这个原型方法。

源码解析：首先我们需要区分当前这个需要添加属性的目标对象是数组还是json，如果是数组，我们就需要用数字的splice方法去添加，这样就直接能触发响应式更新视图，
并且给新增加的属性添加响应式，如果是json对象，我们需要判断当前目标对象是否已经拥有这个key，已经拥有则直接赋值，如果当前目标对象没有响应式，这样
用户可能只是给一个普通的对象添加一个属性，我们只需要简单的添加新属性就好，当目标对象存在响应式既__ob__没属性，并且新增加的可以在目标对象上
并不存在，这样的话则需要调用defineReactive进行添加新增加属性的响应式，然后手动触发notify,做视图更新，并重新收集依赖。
```
Vue.prototype.$set=function(target,key,val){
    //判断当前目标对象（target）是否数组，并进行处理
    if(Array.isArray(target)){
        target.length=Math.max(target.length,key);
        //运用splice这个方法去改变对应的数值，自动触发响应式导致视图更新
        target.splice(key,1,val)
        return val
    }
    //如果当前key已经存在于目标对象（target）上时
    if(key in target  && !(key in Object.prototype)){
        target[key]=val
        return val
    }
    //因为在给属性定义getter和setter时会给对应的value添加__ob__属性，指向这个value对于的dep。
    //所以这个时候我们可以获取这个__ob__属性用作判断当前目标对象（target）是否存在响应式
    //当不存在响应式式，证明用户可能只是需要给一个普通的json对象上添加一个新的属性。
    const ob=target.__ob__;
    if(!ob){
        target[key]=val;
        return val
    }
    //当存在__ob__这个属性的时候，我们需要调用defineReactive手动对新增加的key做响应式处理，对应的value也会做响应式处理
    defineReactive(target,key,val);
    //因为添加了新属性，这个时候我们需要手动的调用目标对象（target）上的dep，让他去触发依赖，做视图更新，并重新收集依赖。
    ob.dep.notify();
    return val
}
```




#### Vue.prototype.$delete
$delete :删除对象的property,如果对象是响应式的则同时更新视图。因为Vue无法检测到property被删除的限制。


源码解析：大致思路和$set一样，首先我们判断是否数组，是的话就直接运用splice直接删除对应位置的val，如果是json对象，则先判断当前目标对象（target）
上是否拥有这个key，没有的话我们什么都不需要执行，如果有的话就直接删除，对于有响应的target（拥有__ob__属性）我们需要更新视图，这个时候
获取这个目标对象的（target）的dep，然后手动调用dep.notify()进行视图更新。
```
Vue.prototype.$delete=function del (target: Array<any> | Object, key: any) {
    if (Array.isArray(target) && isValidArrayIndex(key)) {
        target.splice(key, 1)
        return
    }
    const ob = (target: any).__ob__
    if (!hasOwn(target, key)) {
        return
    }
    delete target[key]
    if (!ob) {
        return
    }
    ob.dep.notify()
}
```