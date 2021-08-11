在Vue2中给数据添加响应化操作是利用Object.defineProperty来进行的，拦截getter操作收集依赖，拦截setter操作触发依赖，但无法拦截数据的操作，对于数组的触发依赖操作，我们是重写 push 、pop 、shift、unshift、splice、sort、reverse方法来，这些方法的共性都是调用后会触发依赖然后进行数据DOM的更新渲染，在Vue3中我们更改用ES6中的Proxy对数据进行响应化操作，他可以拦截数组的操作，包括单不限于变更length、单个数组元素的直接变更、数组方法等,下面我们来具体看看Vue3中响应化。

#### 响应化API

##### reactive
我们直接从调用reactive的入口开始说起，见如下代码。

```
export const enum ReactiveFlags {
  SKIP = '__v_skip', //是否跳过reactive,表示当前对象无法被响应化
  IS_REACTIVE = '__v_isReactive',//是否响应化处理，表示当前对象已经是响应化对象
  IS_READONLY = '__v_isReadonly',//是否只读的标识，表示当前对象是只读的
  RAW = '__v_raw' //响应化数据的原始数据
}
function reactive(target){
    //有传入当前对象，并且当前对象不是响应式只读对象
    if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
        return target
    }
    //创建
    return createReactiveObject(
        target,
        false,
        mutableHandlers,
        mutableCollectionHandlers,
        reactiveMap
    )
}
```
在介绍响应化源码之前，我们需要了解四中状态，在后续的代码中非常有用，
 * ReactiveFlags.IS_REACTIVE的__v_isReactive状态，在调用reactive创建响应化数据时，会添加一个__v_isReactive（ReactiveFlags.IS_REACTIVE的值）属性，为true时表示当前数据是响应化数据。
 
 * ReactiveFlags.IS_READONLY的__v_isReadonly状态，在调用readonly创建只读响应化数据后，在get中会拦截key为__v_isReadonly的访问，总是返回true。

 * ReactiveFlags.SKIP的__v_skip状态，在调用markRaw是，添加一个__v_skip（ReactiveFlags.SKIP的值）属性，为true时表示当前数据永远不会转换成一个代理对象即被响应化处理。

 * ReactiveFlags.RAW的__v_raw状态，当使用toRaw无获取一个响应式对象的原始对象时，在get中会拦截key为__v_raw的访问操作，然后直接返回这个响应式对象的原始数据。

 回到reactive函数中，在该函数中我们先进行判断当前对象是否存在并且是否响应式的只读对象，是的话我们直接返回这个对象，否则我们调用createReactiveObject去创建一个代理。

 ###### createReactiveObject函数
在响应式API中，调用 reactive、shallowReactive、readonly、shallowReadonly这几个方法创建代理对象的时候，在对应的函数实现中都是调用createReactiveObject这个函数，只是传递的参数不一样。
```
@params target:当前需要处理的原始对象。
@params isReadonly:当前是否做只读操作
@params baseHandlers:当targetProxy的第二个参数handler，当target是数组或者Object时用这个handler
@params collectionHandlers:当targetProxy的第二个参数handler，当target是Map、Set、WeakMap、WeakSet时的       handler
@params proxyMap:当前存放数据的Map,存放原始数据作为key，代理对象为value的map，分别为 reactiveMap ：调用reactive创建的、shallowReactiveMap：调用shallowReactive创建的、readonlyMap：调用readonly创建的、shallowReadonlyMap：调用shallowReadonly创建的。

function createReactiveObject(target,isReadonly,baseHandlers,collectionHandlers,proxyMap){
    //如果目标不是对象，则直接禁止执行以下代码
    if (!isObject(target)) { 
        return
    }
    //目标已经是个响应式对象时，直接返回
    if (target[ReactiveFlags.RAW] && !(isReadonly && target[ReactiveFlags.IS_REACTIVE])) {
        return target
    }
    //目标对象已经被响应化，即已经存在于对应的集合中,则值返回这个响应式对象
    const existingProxy = proxyMap.get(target) 
    if (existingProxy) {
        return existingProxy
    }
    //如果当前对象拥有__v_skip，则表示当前对象无法被被响应化，直接返回这个目标对象
    const targetType = getTargetType(target)
    if (targetType === TargetType.INVALID) {
        return target
    }
    //创建响应化对象
    const proxy = new Proxy(
        target,
        targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
    )
    //将创建的响应式对象添加的对应的map对象中
    proxyMap.set(target, proxy)
    return proxy
    
}

```
