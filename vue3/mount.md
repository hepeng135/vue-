#### mount官方介绍

mount:将应用实例的根组件挂载到提供的DOM元素上

通过这段表述，我们知道这将经历4个生命周期阶段：beforeCreate、created、beforeMount、mounted。同时还会执行Vue3中新增加的组合API：setup钩子函数，下面我们将根据这几个周期钩子函数去了解mount方法的源码实现


#### mount方法扩展
和Vue2一样，在源码中Vue3一样对mount方法进行了扩展，这个扩展是基于当前平台假如是浏览器端时的扩展。

在我们调用createApp创建应用实例后，我们紧接着解构出原有的mount方法，并且创建扩展的mount方法
```
//代码位置：runtime-dom/src/index.md
export const createApp=(...args)=>{
    //创建应用实例
    const app = ensureRenderer().createApp(...args)
    const { mount } = app;
    //扩展mount方法
    app.mount=function(containerOrSelector){
        //code...
    }
}
```

#### 扩展的mount方法详情

```
//伪代码，我们只展示代码中的关键部位
app.mount=function(containerOrSelector){
    //获取当前根组件的渲染DOM
    const container = normalizeContainer(containerOrSelector)
    //如果没有给定元素则不执行，与Vue2完全不一样
    if (!container) return
    //获取当前根组件定义项，
    const component = app._component
    //当前根组件不是函数式组件且不带有render和template选项
    if (!isFunction(component) && !component.render && !component.template) {
        component.template = container.innerHTML
    }
    // clear content before mounting
    //根组件渲染前清除根组件里面的内容
    container.innerHTML = ''
    //调用之前我们缓存下来mount方法
    const proxy = mount(container, false, container instanceof SVGElement)
    if (container instanceof Element) {
        container.removeAttribute('v-cloak')
        container.setAttribute('data-v-app', '')
    }
    return proxy
}

```
扩展的mount方法主要作用就是获取html模板，然后紧接着就会调用之前我们解构出的mount方法，并传入我们获取的模板。

#### 真正的mount方法
这个方法我们在源码解析createApp方法时提到过，我们现在直接看源码

```
export function createAppAPI(render,hydrate){
    return function createApp(rootComponent, rootProps = null){
        const context = createAppContext()
        const installedPlugins = new Set()

        let isMounted = false
        const app=context.app={
            //code...
            mount(rootContainer,isHydrate,isSVG){
                if(!isMounted){
                    //初始化创建vnode对象
                    const vnode = createVNode(
                        rootComponent as ConcreteComponent,
                        rootProps
                    )
                    //在根节点的vnode上添加appContext对象，指向当前应用实例的上下文对象
                    vnode.appContext = context
                    render(vnode, rootContainer, isSVG)
                    isMounted = true
                    app._container = rootContainer
                    rootContainer.__vue_app__ = app
                    return vnode.component.proxy
                }
            }
        }
         return app
    }
}

```
在createApp这个函数中我省略了很多代码，感兴趣的可以去看看上一篇createApp的介绍，在这个函数中最重要就是调用createVNode函数和render函数。

1. createVNode函数。
    * 在VNode函数中，我们更加传入组件项判断当前组件属于哪种类型，即VNode对象的shapeFlag属性，分为，1代表原生元素，2代表函数组件，4：有状态的组件，8：文本节点，16代表有子节点，32代表slot节点，64代表是内置teleport组件，128代表suspense组件，在我们平时在实例化创建根组件的时候，他的类型为4。
    * 创建一个VNode对象并从props中解构出为key和ref的属性，并赋值给VNode.key和VNode.ref,

2. render函数
    * render函数，执行这个函数会引起的一系列的函执行，在这期间我们将执行前面提到的五个钩子函数，完成了这个应用实例的根组件从初始化到挂载的全部过程，下面我们将重点介绍调用这个函数所做到哪些时。

#### createApp中调用render函数所做的事情

首先我们先看看render函数的这个代码。他定义在runtime-core/src/renderer.ts文件的baseCreateRenderer函数中，具体可以查看createApp详解。
```

function baseCreateRenderer(options,createHydrations){
    //code...
    //vnode:vnode对象，container:当前组件项的定义，isSVG：是否svg元素
    const render=(vnode,container,isSvg){
        //卸载这个应用实例
        if (vnode == null) {
            if (container._vnode) {
                unmount(container._vnode, null, null, true)
            }
        }else{
            patch(container._vnode || null, vnode, container, null, null, null, isSVG)
        }
        flushPostFlushCbs()
        container._vnode = vnode 
    }

    //code...
}
```
在render函数中我们将调用patch函数，至此我们将进入VNode的核，这里我们将只介绍组件的创建过程，并且我们以根组件的创建过程为例，即根组件的beforeCreate->mounted的过程。

因为是跟组件的创建过程，所有我们调用path的第一个参数会是null，isSVG也将会是false。

##### path函数中调用processComponent处理组件，同时在processComponent函数中，我们将会调用mountComponent
```
function baseCreateRenderer(options,createHydrations){
    //code...
    const patch=function(n1,n2,container,anchor = null,parentComponent = null,parentSuspense = null,isSVG = false,slotScopeIds = null,optimized = false){
        //code...
        const { type, ref, shapeFlag } = n2
        switch (type) {
            case Text:  //文本
                processText(n1, n2, container, anchor)
                break
            case Comment: //注释
                processCommentNode(n1, n2, container, anchor)
                break
            case Static: //静态
                if (n1 == null) {
                    mountStaticNode(n2, container, anchor, isSVG)
                } else if (__DEV__) {
                    patchStaticNode(n1, n2, container, isSVG)
                }
                break
            case Fragment: //顶级没有父元素包括的片段
                processFragment(n1,n2,container,anchor,parentComponent,
parentSuspense,isSVG,slotScopeIds,optimized)
                break 
            default:
                if (shapeFlag & ShapeFlags.ELEMENT) {//当前type对应的是dom元素时
                    processElement(n1,n2,container,anchor,parentComponent,parentSuspense,isSVG,slotScopeIds,optimized)
                } else if (shapeFlag & ShapeFlags.COMPONENT) {//当前type对应的是组件时
                    processComponent(n1,n2,container,anchor,parentComponent,parentSuspense,isSVG,slotScopeIds,optimized)
                } else if (shapeFlag & ShapeFlags.TELEPORT) {//当type为teleport组件时
                    type.process(n1,n2,container,anchor,parentComponent,parentSuspense,isSVG,slotScopeIds,optimized,internals)
                } else if (__FEATURE_SUSPENSE__ && shapeFlag & ShapeFlags.SUSPENSE) {//当是suspense
                    type.process(n1,n2,container,anchor,parentComponent,parentSuspense,isSVG,slotScopeIds,optimized,internals)
                } else if (__DEV__) {
                    warn('Invalid VNode type:', type, `(${typeof type})`)
                }
        }
        //设置ref组件引用
        if (ref != null && parentComponent) {
            setRef(ref, n1 && n1.ref, parentSuspense, n2 || n1, !n2)
        }
    }
    //code...
}
```
在patch函数中我们解构出之前项VNode上添加的shapeFlag属性，然后进行逐步判断，然后调用processComponent方法，在processComponent方法中由于我们是创建根组件实例对象，则会调用mountComponent方法

```
function baseCreateRenderer(options,createHydrations){
    //code...
    const patch=(n1,n2,container, anchor = null,parentComponent = null,parentSuspense = null,isSVG = false,slotScopeIds = null,optimized = false){
        n2.slotScopeIds = slotScopeIds
        if (n1 == null) { //如果没有旧节点
            if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) { //当新节点组件是keep-live时
                //code..
            } else {
                //处理组件
                mountComponent(n2,container,anchor,parentComponent,parentSuspense,isSVG,optimized)
            }
        } else {
            //更新组件
            updateComponent(n1, n2, optimized)
        }
    }

}
```

##### mountComponent函数，功能如其函数名，挂载组件，也是mount方法中的核心函数。

在这个函数中我们将调用createComponentInstance创建组件对象，调用setupComponent函数，处理一些组件选项的初始化，稍后将详情介绍，然后调用setupRenderEffect函数创建副作用，期间渲染组件，触发getter收集这个副作用。
```
function baseCreateRenderer(options,createHydrations){
    //code...
    const mountComponent=function(initialVNode,container,anchor,parentComponent,parentSuspense,isSVG,optimized){
        //首先是获取组件对象，没有的话则创建组件对象
        const compatMountInstance =__COMPAT__ && initialVNode.isCompatRoot && initialVNode.component;
        const instance= compatMountInstance || (initialVNode.component = createComponentInstance(
            initialVNode,
            parentComponent,
            parentSuspense
        ))
        //如果是keepLive组件时
        if (isKeepAlive(initialVNode)) {
            ;(instance.ctx as KeepAliveContext).renderer = internals
        }
        //准备开始处理当前组件的setup构造函数
        //在setup执行完毕以后，编译模板生成render函数，执行beforeCreate钩子函数
        //初始化provide  data数据观测   methods   watch  event computed
        //执行created钩子函数
        //依赖收集准备工作，即生成vNode之前执行patch之前调用beforeMount函数
        setupComponent(instance);
        //如果setup是个async函数，返回promise时
        if (__FEATURE_SUSPENSE__ && instance.asyncDep) {
            parentSuspense && parentSuspense.registerDep(instance, setupRenderEffect)

            // Give it a placeholder if this is not hydration
            // TODO handle self-defined fallback
            if (!initialVNode.el) {
                const placeholder = (instance.subTree = createVNode(Comment))
                processCommentNode(null, placeholder, container!, anchor)
            }
            return
        }

        setupRenderEffect(
            instance,
            initialVNode,
            container,
            anchor,
            parentSuspense,
            isSVG,
            optimized
        )
    }
}
```

###### createComponentInstance函数，初始化创建组件对象实例。
```
function createComponentInstance(vnode,parent,suspense){
    const type = vnode.type;
    //获取当前根组件的上下文。
    const appContext =(parent ? parent.appContext : vnode.appContext) || emptyAppContext
    //创建这个组件的对象实例
    var instance= {
        //一些属性，太多了  就不贴出来呢
    }
    if (__DEV__) {
        instance.ctx = createRenderContext(instance)
    } else {
        instance.ctx = { _: instance }
    }
    //当前组件实例的根节点实例
    instance.root = parent ? parent.root : instance
    //挂载当前组件上对的emit方法,更改第一个参数为当前组件对象
    instance.emit = emit.bind(null, instance)

    return instance
}
```
###### setupComponent函数，
在这个函数中将执行我们心里期待的一些功能，前面那么多铺垫都是为了接下来的事，接下来我们将准备开始处理当前组件的setup构造函数，在setup执行完毕以后，编译模板生成render函数，执行beforeCreate钩子函数，初始化组件的provide、data数据观测、methods、watch 、event 、computed 等组件选项，执行created钩子函数，依赖收集准备工作，即生成vNode之前执行patch之前调用beforeMount函数。

```
function setupComponent(instance,isSSR=false) {
    isInSSRComponentSetup = isSSR
    //从vnode解构出props和children
    const { props, children } = instance.vnode
    //确定是否状态组件
    const isStateful = isStatefulComponent(instance);
    //初始化组件上所有的属性   区分出 组件定义的prop  和 为定义的attrs
    initProps(instance, props, isStateful, isSSR)
    //初始化当前组价的slots
    initSlots(instance, children)
    const setupResult = isStateful ? setupStatefulComponent(instance, isSSR) : undefined
    isInSSRComponentSetup = false
    return setupResult
}
```
###### 在setupComponent函数中调用setupStatefulComponent函数
```
function setupStatefulComponent(instance,isSSR){
    const Component = instance.type;//或者组件选项

    instance.accessCache = Object.create(null)
    instance.proxy = markRaw(new Proxy(instance.ctx, PublicInstanceProxyHandlers));

    //解构出setup构造函数
    const { setup } = Component
    if (setup) {
      //确定setup的第二个参数 {attrs , slots , emit }
        const setupContext = (instance.setupContext = setup.length > 1 ? createSetupContext(instance) : null);
        //将shouldTrack变成为false
        //这将有利于我们在在执行setup钩子函数时，访问响应数据而出发对应的依赖收集。
        pauseTracking()
        //调用setup钩子函数
        const setupResult = callWithErrorHandling(
            setup,
            instance,
            ErrorCodes.SETUP_FUNCTION,
            [__DEV__ ? shallowReadonly(instance.props) : instance.props, setupContext]
        )
        resetTracking() //执行完毕后恢复shouldTrack状态
        //接下来处理setup函数返回的结果，因为可能是一个promise，因为我们可能将setup定义为一个async函数
        currentInstance = null
        if (isPromise(setupResult)) {
            const unsetInstance = () => {
                currentInstance = null
            }
            setupResult.then(unsetInstance, unsetInstance)
            if(isSSR){//如果在服务端
                //code...
            }else if(__FEATURE_SUSPENSE__){//suspense 处理 async setup
                //等待返回结果
                instance.asyncDep = setupResult
            }
        }else{
            //处理setup的返回结果 
            handleSetupResult(instance, setupResult, isSSR)
        }
    }else{
        //没有setup函数直接调用完成函数
        finishComponentSetup(instance, isSSR)
    }

}
```
上述代码我们可以看到我们调用setup钩子函数，当返回值是promise类型，然后将其配合suspense进行处理，假如返回值不是promise对象，则我们直接调用handleSetupResult处理返回的结果，假如没有setup钩子函数，则我们直接调用finishComponentSetup函数，表示setup已经完成。

###### 在setupStatefulComponent中调用handleSetupResult函数处理setup钩子的返回结果返回的

```
function handleSetupResult(instance,setupResult,isSSR){
    //当返回结果是个函数时，我们认为返回的是个render函数
    if(isFunction(setupResult)){
        //只显示浏览器端的代码
        instance.render = setupResult
    }else if (isObject(setupResult)){//当返回的是一个json对象时，则将结果进行响应化处理
        //将setup返回的结果setupResult进行代理到instance的setupState的属性上。
        //浅层次代理，并且代理中没有收集和触发，依赖于里面属性的代理
        instance.setupState = proxyRefs(setupResult)
    }
    /setup钩子函数处理完成，我们调用finishComponentSetup进行下一步
    finishComponentSetup(instance, isSSR)
}
```

###### 在handleSetupResult函数中处理setup函数返回的结果后，我们在调用finishComponentSetup进行下一步。

```
function finishComponentSetup(instance,isSSR,skipOptions){
    const Component = instance.type //获取当前组件的选项  
    if (__NODE_JS__ && isSSR) {
        //ssr端
    }else if(!instance.render){ //在浏览器端且当前组件实例并没有render函数时
        if(compile && !Component.render){ //有编译模块且组件并没有带有render函数
            //获取当前组件的html模板
            const template = Component.template;
            if (template) { 
                //整合编译html期间，compile的一些配置项，在最新版本的vue中有的配置项已经更新
                const { isCustomElement, compilerOptions } = instance.appContext.config
                const {
                    delimiters,
                    compilerOptions: componentCompilerOptions
                } = Component
                const finalCompilerOptions = extend(
                    extend({isCustomElement,delimiters},compilerOptions),
                    componentCompilerOptions
                )
                //编译html生成render函数。
                Component.render = compile(template, finalCompilerOptions)
            }
        }
    }
    //将render函数挂载到组件实例对象上
    instance.render = Component.render || NOOP;
    if (instance.render._rc) {
        //给instance.ctx对象 { _ : instance} 加上代理，返回的代理对象赋值给withProxy
        instance.withProxy = new Proxy(
            instance.ctx,
            RuntimeCompiledPublicInstanceProxyHandlers
        )
    }
    //初始化处理出setup除外的其他组件选项   如：props data  methods  等
    if (__FEATURE_OPTIONS_API__ && !(__COMPAT__ && skipOptions)) {
        //确定当前正在处理的组件实例对象
        currentInstance = instance
        //修改shouldTrack对象，在访问的时候不会触发get进行收集依赖
        pauseTracking()
        //处理组件选项
        applyOptions(instance)
        //重置之前设置的变量状态
        resetTracking()
        currentInstance = null
    }
}
```
在上述代码中我们将html模板编译成render函数，然后调用applyOptions函数去初始化组件选项。

###### 在finishComponentSetup函数中调用applyOptions函数处理组件选项
```
export function applyOptions(instance){
    //合并组件选项   如mixins extends
    const options = resolveMergedOptions(instance)
    //获取当前这个组件实例的代理
    const publicThis = instance.proxy
    const ctx = instance.ctx
    //执行beforeCreate钩子函数
    if (options.beforeCreate) {
        callHook(options.beforeCreate, instance, LifecycleHooks.BEFORE_CREATE)
    }
    //从当前组件选项中解构出组件的每一项
    const {
        // state
        data: dataOptions,
        computed: computedOptions,
        methods,
        watch: watchOptions,
        provide: provideOptions,
        inject: injectOptions,
        // lifecycle
        created,
        beforeMount,
        mounted,
        beforeUpdate,
        updated,
        activated,
        deactivated,
        beforeDestroy,
        beforeUnmount,
        destroyed,
        unmounted,
        render,
        renderTracked,
        renderTriggered,
        errorCaptured,
        serverPrefetch,
        // public API
        expose,
        inheritAttrs,
        // assets
        components,
        directives,
        filters
    } = options
    //处理inject项，并获取对应的注入的数据，先找父级的parent.provides  没有就找根组件实例上的instance.vnode.appContext.provides
    if (injectOptions) {
        resolveInjections(injectOptions, ctx, checkDuplicateProperties)
    }
    //初始化处理methods
    if(methods){
        for (const key in methods) {
            const methodHandler = (methods as MethodOptions)[key]
            if (isFunction(methodHandler)) {
                ctx[key] = methodHandler.bind(publicThis)
            }
        }
    }
    //处理组件选项data
    if (dataOptions){
        //执行data选项的函数，获取返回data
        const data = dataOptions.call(publicThis, publicThis);
        //对data数据进行响应化代理，然后将响应对象挂载到当前对象的data属性上
        instance.data = reactive(data)
    }
    //处理组件的computed属性
    if(computedOptions){
        for (const key in computedOptions) {
            const opt = (computedOptions as ComputedOptions)[key]
            
            //处理计算属性的get 和 set，单独定义一个函数，默认为该计算属性的get
            const get = isFunction(opt) ? opt.bind(publicThis, publicThis) : isFunction(opt.get) ? opt.get.bind(publicThis, publicThis) : NOOP;
            const set =!isFunction(opt) && isFunction(opt.set) ? opt.set.bind(publicThis)
            : NOOP;
            //创建一个惰性的副作用，后期我们会专门详解computed
            const c = computed({
                get,
                set
            })
            Object.defineProperty(ctx, key, {
                enumerable: true,
                configurable: true,
                get: () => c.value,
                set: v => (c.value = v)
            })
        }
    }
    //处理watch选项
    if(watchOptions){
        //创建watcher
        for (const key in watchOptions) {
            createWatcher(watchOptions[key], ctx, publicThis, key)
        }
    }
    //处理provide项,继承父组件的provides数据，然后在添加当前组件的provide选项到当前的provides属性上
    
    if(provideOptions){
        const provides = isFunction(provideOptions) ? provideOptions.call(publicThis) : provideOptions;
        Reflect.ownKeys(provides).forEach(key => {
            provide(key, provides[key])
        })
    }
    //初始化组件选项后，我们执行create构造函数
    if (created) {
        callHook(created, instance, LifecycleHooks.CREATED)
    }

    //注册生命周期钩子函数
    function registerLifecycleHook(
        register: Function,
        hook?: Function | Function[]
    ) {
        if (isArray(hook)) {
        hook.forEach(_hook => register(_hook.bind(publicThis)))
        } else if (hook) {
            //当前这个钩子函数
        register((hook as Function).bind(publicThis))
        }
    }
    //收集该组件周期钩子函数
    registerLifecycleHook(onBeforeMount, beforeMount)
    registerLifecycleHook(onMounted, mounted)
    registerLifecycleHook(onBeforeUpdate, beforeUpdate)
    registerLifecycleHook(onUpdated, updated)
    registerLifecycleHook(onActivated, activated)
    registerLifecycleHook(onDeactivated, deactivated)
    registerLifecycleHook(onErrorCaptured, errorCaptured)
    registerLifecycleHook(onRenderTracked, renderTracked)
    registerLifecycleHook(onRenderTriggered, renderTriggered)
    registerLifecycleHook(onBeforeUnmount, beforeUnmount)
    registerLifecycleHook(onUnmounted, unmounted)
    registerLifecycleHook(onServerPrefetch, serverPrefetch)

    //render选项，不会替换前期setup中返回的render函数
    if (render && instance.render === NOOP) {
        instance.render = render as InternalRenderFunction
    }
    //组件属性继承
    if (inheritAttrs != null) {
        instance.inheritAttrs = inheritAttrs
    }
    // 组件  指令
    if (components) instance.components = components as any
    if (directives) instance.directives = directives
}
```
至此setupComponent函数引起的一系列的函数调用到此结束，我们执行了setup组合API钩子函数，并且初始化编译html模板生成render函数，执行组件周期钩子函数beforeCreate，然后初始化组件项（inject,data,methods,provide,computed,watch,components,directives）,然后执行created钩子函数，接下来我们将开始创建整个组件的副作用，调用setupRenderEffect函数

##### 调用setupRenderEffect函数创建当前render函数对应的副作用函数
```
const setupRenderEffect=(instance,initialVNode,container,anchor,parentSuspense,isSVG,optimized){
    instance.update = effect(function componentEffect(){
        if (!instance.isMounted) {
            let vnodeHook;
            const { el, props } = initialVNode
            const { bm, m, parent } = instance

            //执行beforeMount钩子函数
            if (bm) {
                invokeArrayFns(bm)
            }
            //code...
            const subTree = (instance.subTree = renderComponentRoot(instance))
            patch(
                null,
                subTree,
                container,
                anchor,
                instance,
                parentSuspense,
                isSVG
            )
            initialVNode.el = subTree.el
            //执行mounted钩子函数
            if (m) {
                queuePostRenderEffect(m, parentSuspense)
            }
        }
    })

}

```
上述代码中，我们调用effect函数，并传入一个componentEffect函数作为参数，我们来看看effect是干嘛的。

###### effect函数
```
export function effect(fn,options){
    if (isEffect(fn)) { //当前这个函数是否副作用函数
        fn = fn.raw
    }
    //创建effect对象，然后没有延迟调用就立马调用
    const effect = createReactiveEffect(fn, options)
    if (!options.lazy) {//如果不是
        effect() //立即执行副作用
    }
    return effect
}
```
主要是利用传进来的fn函数，我们会为这个fn函数创建一个副作用对象，我们来看看createReactiveEffect函数

###### createReactiveEffect函数
```
function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
    const effect = function reactiveEffect() {
        if (!effect.active) {
            return fn()
        }
        //判断effectStack数组中是否拥有当前这个effect
        if (!effectStack.includes(effect)) {
            cleanup(effect)//清除当前副作用对象上的dep
            try {
                enableTracking()//启动跟踪，将shouldTrack设置成true,并添加到trackStack数组中
                effectStack.push(effect)//添加当前正在运行的effect
                activeEffect = effect//当前正在运行的函数
                return fn() //运行函数，在里面会访问响应式属性，然后触发getter,收集当前的effect
            } finally {
                //当前effect的已经收集完毕，我们需要清理
                effectStack.pop()
                resetTracking()
                activeEffect = effectStack[effectStack.length - 1]
            }
        }
    }
    effect.id = uid++
    effect.allowRecurse = !!options.allowRecurse
    effect._isEffect = true
    effect.active = true
    effect.raw = fn
    effect.deps = []
    effect.options = options
    return effect
}
```
我们在createReactiveEffect函数中创建一个副作用函数（effect），并为这个函数挂载一些属性，在这个副作用函数进行调用的时候，我们调用enableTracking函数更改是否收集依赖的状态，然后将当前副作用函数添加到当前任务队列中，并更新当前正在执行副作用函数，然后执行fn函数进行DOM渲染，期间触发数据代理的getter，从而收集副作用，完成以后，即我们在finally中进行一些状态修正。

回到setupRenderEffect函数中，我们调用effect函数，会根据传入的这个函数创建一个副作用对象，当这个副作用执行时，更新全局的shouldTrack、activeEffect状态，effectStack更新当前任务队列，做好依赖收集前的准备，然后我们会执行这个传入的函数，在这个函数中我们会先执行beforeMount钩子函数，然后调用renderComponentRoot函数根据当前组件实例对象上的render函数生成虚拟DOM，然后调用patch进行对比，因为是第一次组件生成，所以这里是直接将虚拟DOM 创建成真正的DOM，然后挂载到DOM文档中，在执行mounted钩子函数。
