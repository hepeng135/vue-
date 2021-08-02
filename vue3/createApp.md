#### 调用createApp发生的一些事

创建一个应用实例，在这个应用实例中，应用实例其实就是一个json对象，会拥有一些属性和方法。主要方法有use，component、directive,给应用实例添加插件、组件、指令，和mount、unmount,挂载根组件和卸载根组件。以及查看该应用的默认配置。

与vue2的不同(实例时没传el的情况下)
vue2中，当我们new Vue后，我们就创建了根组件实例，并且根组件的beforeCreate和created构造函数已经执行，即已经做完数据响应化操作，property 和方法的运算，watch/event 事件回调。

Vue3这样做的好处：应用实例的config配置不会共享，每个应用实例都会拥有一个单独的配置对象。

##### 执行流程
定位文件：runtime-dom/src/index.md 中的createApp函数
###### 1. 执行createApp函数，首先调用 ensureRenderer 函数。
```
function createApp(){
    const app = ensureRenderer().createApp(...args)
    //code...
}
```
###### 2. 在ensureRenderer函数中我们会调用createRenderer，并传入rendererOptions作为参数，并返回调用这个函数所得到值。
```
function ensureRenderer(){
    return renderer || (renderer = createRenderer<Node, Element>(rendererOptions)) 
}
```
* rendererOptions参数的详解
通过Object.assign()去将 {  patchProp, forcePatchProp } 和 nodeOps对象合并。
```
/**
*   extend 其实就是 Object.assign , 
*   patchProp:DOM原生的属性处理，比如，class、style等和添加DOM事件操作
*   forcePatchProp:判断当前传入的 key 是否是字符串 value
*   nodeOps：标签的一些DOM操作封装，获取，删除，增加，创建，移动等。
**/
const rendererOptions = extend({ patchProp, forcePatchProp }, nodeOps)

```
###### 3. 在createRenderer函数中执行baseCreateRenderer函数，返回执行该函数得到的结果
文件定位：runtime-core/src/renderer.ts
```
function createRenderer(options){
    return baseCreateRenderer(options)
}
```
###### 4. 执行baseCreateRenderer函数
该函数非常大，主要实现render函数（创建VNode），并且实现patch函数(对比新旧VNode进行更新操作)。该函数拥有返回值。{ render , hydrate , createApp}
```
function baseCreateRenderer(options,createHydrationFns){
    //code...省略千行代码

    return {
        render,
        hydrate,
        createApp: createAppAPI(render, hydrate)
    }
}
```
###### 5.执行createAppAPI函数，该函数只返回一个函数createApp.
文件定位：runtime-core/src/aipCreateApp.ts
```
function createAppApi(render,hydrate){
    return function createApp(rootComponent, rootProps){
        //code....省略创建应用实例app
        return app
    }
}
```

###### 结尾
至此我们回到第一步，我们执行ensureRenderer函数，让我们得到了createApp函数，然后执行它创建应用实例。

在createApp函数中我们创建一个app对象

上面拥有一些方法，如：component、directive、provide、mixin、use、mount、unmounted、config。

```
app={
    _uid: uid++,
    //根组件实例
    _component: rootComponent as ConcreteComponent,
    _props: rootProps,
    //根组件渲染的元素
    _container: null,
    
    _context:{
        app:null, //指向当前应用实例app,互相引用
        config:{ //当前Vue应用实例的配置项，具体可以从API中得知
            isNativeTag: NO,
            performance: false,
            globalProperties: {},
            optionMergeStrategies: {},
            errorHandler: undefined,
            warnHandler: undefined,
            compilerOptions: {}
        },
        //存放调用mixin混入的选项
        mixins: [],
        //存放该应用实例的全局组件
        components: {},
        //存放该应用实例的全局指令
        directives: {},
        //存放该应用实例提供的所有数据
        provides: Object.create(null),
        //存放缓存数据
        optionsCache: new WeakMap(),
        propsCache: new WeakMap(),
        emitsCache: new WeakMap()
    } ,
    _instance: null,
    version, //版本号
    //定义config的get和set行为，app应用的配置只能获取，不能设置
    get config(){return _context.config}
    set config(v){ 
        if (__DEV__) {
            warn(
            `app.config cannot be replaced. Modify individual options instead.`
            )
        }
    }
    use(){}//安装插件的api
    mixin(){}//混入API
    component(){}//注册或者检索全局组件，所谓全局是相对于这个应用的全局
    directive(){}//注册或者检索全局指令
    mount(){}//应用挂载到实际DOM中，
    unmount(){}//卸载运用实例的根组件
    provide(){}//给应用中所有的组件提供值

}
```