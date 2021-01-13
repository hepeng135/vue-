大家在用Vue开发项目的时候都会使用VueRouter这一核心插件来提供路由，下面我们就来了解一下VueRoute的源码实现。首先我们平时一个
开发一个Vue应用时，都会有一个入口文件，如下

```
import from Vue
import from VueRouter
//安装插件
Vue.use(VueRuter)
//定义路由下使用的组件
const Home={template:'<p>home</p>'};
const Foo={template:'<p>home</p>'};
const Bar={template:'<p>home</p>'};

//创建路由实例
const router=new VueRouter({
    mode:'history',
    base:'/',
    routes:[
        {path:'/',component:'Home'},
        {path:'/foo',component:'Foo'},
        {path:'/bar',component:'Bar'}
    ]
})

new Vue({
    router,
    template:`
        <div class="app">
            <h1>main</h1>
            <ul>
                <li><router-link to='/'></router-link></li>
                <li><router-link to='/foo'></router-link></li>
                <li><router-link to='/bar'></router-link></li>
            </ul>
            <router-view></router-view>
        </div>
    `    
}).$mount('.app');

```
如上述代码所示，在导入VueRouter后，我们调用Vue.use安装VueRouter插件，每个Vue插件对象都应该拥有install方法，如果没有
就会把该插件本身作为函数来调用，我们先来看看VueRouter的install方法具体是怎么定义的



## VueRouter的install方法
在src/index.js文件中，我们的导入install，并挂载到VueRouter上。
```
import { install } from './install'
import { inBrowser } from './util/dom'

//...
export default calss VueRouter {
    //...
}
//...
VueRouter.install = install
// 自定使用插件，防止用户忘记调用Vue.use
if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}
```
如上，Vue插件系列的一套常规操作，给VueRouter挂载install方法，下面我们将详细介绍install函数是怎么去定义的，主要做些什么

文件路径：src/install.js
```
//export一个Vue的引用，因为打包的时候不希望Vue作为一个依赖包打进来，但是有需要用到Vue对象的一些方法，就先声明一个
//_Vue的全局变量，在install中给其辅助为Vue，这样我们就可以其他地方也使用Vue对象的方法,同时也不用引入Vue的依赖包，当前必须在install方法执行后。
export let _Vue

export function install (Vue) {
    if (install.installed && _Vue === Vue) return
    install.installed = true
    
    _Vue = Vue
    //注册实例，将在组件实例化时的beforeCreate钩子函数中调用
    const registerInstance = (vm, callVal) => {
        let i = vm.$options._parentVnode
   
        if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
            i(vm, callVal)
        }
    }
    //给每个组件混入beforeCreate、destroyed钩子函数
    Vue.mixin({
        beforeCreate () {
            //当对象实例的$options上中否有router对象，只有当前是根组件时，才会有这个router属性
            if (isDef(this.$options.router)) {
                this._routerRoot = this  
                this._router = this.$options.router 
                //初始化当前_router
                this._router.init(this)
                //添加 _route这个响应式对象，指向当前的路由对象（当前路由的详细信息）
                Vue.util.defineReactive(this, '_route', this._router.history.current)
            } else { 
                //如果不是根组件，则添加_routerRoot，指向当前根组件数量（this.$parent._routerRoot）
                this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
            }
            registerInstance(this, this)
        },
        destroyed () {
            registerInstance(this)
        }
    })
    //Vue实例上添加$router属性，指向路由对象实例
    Object.defineProperty(Vue.prototype, '$router', {
        get () { return this._routerRoot._router }
    })
    //Vue实例上添加$route属性，指向当前地址栏地址对应路由的详情信息对象
    Object.defineProperty(Vue.prototype, '$route', {
        get () { return this._routerRoot._route }
    })
    //注册RouterView和RouterLink两个组件
    Vue.component('RouterView', View)
    Vue.component('RouterLink', Link)
    
    //自定义合并策略
    const strats = Vue.config.optionMergeStrategies
    //将组件内部三个路由钩子函数挂载到组件实例上，自定义的合并策略与Vue中处理created的钩子函数策略一样
    //既 当前组件实例化时，处理的三个钩子函数，会挂载到 componentInstance.options.beforeRouteEnter/beforeRouterUpdate/beforeRouterLeave=[] 对应的数组中。
    strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created   
}

```
上述代码帮我们做了以下几种事情，比如我们开发中常用的$route、$router、router-view、router-link.。
* 给每个组件混入beforeCreate构造函数，在该钩子函数中，为根组件添加_router(指向当前VueRouter实例)、_route(指向当前路由信息，响应式对象)，
并添加_routerRoot（指向当前根组件），由于组件实例化的顺序是先父后子的属性，在子组件上通过父级的_routerRoot属性访问到父级实例，
层层递增，每个子组件都用_routerRoot属性，进而可以访问到根组件上的_router和route，然后再在Vue.property上设置$router和$route属性，
$router指向：通过当前组件实例访问其自身的_routerRoot（根组件实例，然后访问根组件上的_router属性），$route指向：通过当前组件实例访问其自身的_routerRoot（根组件实例，然后访问根组件上的_route属性）
因为每个组件都继承自Vue，所有每个组件上我们都能访问到$router和$route

* 注册router-view和router-link组件,关于他们的代码细节，我们后面细说。

* 自定义混入组件独享的路由守卫，beforeRouteEnter、beforeRouteLeave、beforeRouteUpdate,遵循的混入规则和组件的created构造函数一样。

* 一个很重要的疑问，为什么_route属性是响应式的，这个问题后面我们解答，这个非常的重要的一环

* 注意 this._router.init(this) 具体起到什么作用？ 

根据全面的Vue应用的代码，接下来我们将要看VueRouter是如何实例化的，
## 实例化VueRouter
在Vue应用入口文件，首先要实例化一个VuerRouter,然后传入Vue实例的options中，我们先看下VueRouter的构造函数，src/index.js暴露出VueRouter类。
```
import { createMatcher } from './create-matcher'
export default class VueRouter {
    constructor (options: RouterOptions = {}) {
        this.app = null // 当前正在初始化的组件实例
        this.apps = []  // 每次初始化组件后触发beforeCrate钩子函数就会被添加到这个数组， 所有已经初始化过的组件实例
        this.options = options
        this.beforeHooks = []
        this.resolveHooks = []
        this.afterHooks = []
        // 创建 match 匹配函数 { match, addRoutes}
        this.matcher = createMatcher(options.routes || [], this)
        // 根据 mode 实例化具体的 History
        let mode = options.mode || 'hash'
        this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
        if (this.fallback) {
            mode = 'hash'
        }
        if (!inBrowser) {
            mode = 'abstract'
        }
        this.mode = mode
        /* 几种模式判断*/
        switch (mode) {
            case 'history':
                this.history = new HTML5History(this, options.base)
                break
            case 'hash':
                this.history = new HashHistory(this, options.base, this.fallback)
                break
            case 'abstract':
                this.history = new AbstractHistory(this, options.base)
                break
            default:
                if (process.env.NODE_ENV !== 'production') {
                    assert(false, `invalid mode: ${mode}`)
                }
            }
        }
    }
    //...
}
```
先初始化一些属性变量，其中最终要的一步是this.matcher = createMatcher(options.routes || [], this)，调用createMatcher函数，传入路由
对象信息和当前实例，主要返回匹配函数，代码如下
#### 处理路由的对象信息，并返回对应的match函数

```
export function createMatcher(routes,router){
    // 将路径处理成 pathList：pathArray eg:['','/foo','/boo']
    // pathMap [Object] :{'/':{path,regx,name,computed,meta,props},'/foo':{...}}
    // nameMap [Object] :{name:{path,regexp}}
    const { pathList, pathMap, nameMap } = createRouteMap(routes)
    
    function addRoutes(routes){//...}

    function match(raw,currentRoute,redirectedFrom){//...}

    function redirect(record,location){//...}

    function alias(record,location,matchAs){//...}

    function _createRoute(){//...}

    return {
        match,
        addRoutes
      }
}
```
在上述函数中，我们调用createRouteMap函数，将传入的路由对象处理成对应的map，在其中用到了path-to-regexp插件，将路径换行成对象正则表达式，
用于后期匹配，然后返回pathList，pathMap，nameMap 三个map。然后createMatcher函数返回match匹配函数。

## 实例化History
在VueRouter构造函数中，我们根据mode来判断当前选择哪种模式，我们常用hash、history这两种的一个，abstract为Node服务器所用。这里我们先只介绍
history这种模式。具体的文件都在src/history文件夹下。

首先 实例化HTML5History构造函数，并传入当前的VueRouter实例和options.base作为参数。
在HTML5History构造函数中我们很明显可以看出这个类继承自History。我们需要看看History
```
class History {
    this.router = router  // vueRouter实例
    this.base = normalizeBase(base)  // 处理默认值 默认为/。也可以用户这种/app/=>/app
    // start with a route object that stands for "nowhere"
    this.current = START // 以path为‘/’创建一个起始路由对象作为当前路由对象。
    this.pending = null
    this.ready = false
    this.readyCbs = []
    this.readyErrorCbs = []
    this.errorCbs = []
}
//...
```
根类History在构造函数中首先确定了当前的默认路由为'/'的路由对象，和当前base处理后的值,如下
eg:配置项设置base为/app，this.base则为 “/app”
this.current默认为当前路由对象既  {name: null,meta: {},path: "/",hash: "",query: {},params: {},fullPath: "/", matched: []}

回到HTML5History构造函数中，在构造函数中我们做了以下事情
```
export class HTML5History extends History {
    constructor (router: Router, base: ?string) {
        super(router, base)
    
        const expectScroll = router.options.scrollBehavior// 获取滚动信息
        const supportsScroll = supportsPushState && expectScroll  // 是否支持h5 history
    
        if (supportsScroll) {
            setupScroll()  // 记录当前页面进入时的位置
        }
    
        const initLocation = getLocation(this.base) // 获取当前页面除域名外的所有信息，同时也剔除base
        window.addEventListener('popstate', e => {  // 当点击链接进行跳转时
        const current = this.current
    
        const location = getLocation(this.base)
        if (this.current === START && location === initLocation) {
            return
        }
    
        this.transitionTo(location, route => {
            if (supportsScroll) {
                handleScroll(router, route, current, true)
            }
        })
    })
    go(n){...}
    push(location,onComplete,onAbort){...}
    replace(location,onComplete,onAbsort){...}
    ensureURL(push){...}
    getCurrentLocation(){...}
}
```
1. 首先获取路由的滚动配置函数scrollBehavior，并判断当前浏览器是否兼容H5History API。

2. 在兼容H5 History API的浏览器中调用setupScroll函数，调用History API 的replace替换当前history中的状态，{key:当前的时间戳},'',当前的url（去掉协议+域名），
并且监听popstate事件，当历史记录发生变化时存储当前页面的window.pageXOffset和window.pageYOffset到positionStore对象中{x,y}，并更新对应的key
```
export function setupScroll () {

    // 获取当前的页面协议+域名
    const protocolAndPath = window.location.protocol + '//' + window.location.host
    // 去掉协议+域名
    const absolutePath = window.location.href.replace(protocolAndPath, '')
    // 替换当前history栈中的状态
    window.history.replaceState({ key: getStateKey() }, '', absolutePath)
    window.addEventListener('popstate', e => { // 绑定popstate事件，当该事件发生时
        saveScrollPosition() // 存储当前的scrollX scrollY 到对象 positionStore上 {x,y}
        if (e.state && e.state.key) {//获取当前的history对象是否存在state，然后更新对应的key值，用于下次存储。
            setStateKey(e.state.key)//
        }
    })
}
```

3. 调用getLocation函数，获取当前页面的路径 （剔除base），然后监听popstate事件，
```
    const initLocation = getLocation(this.base) // 获取当前页面除域名外的所有信息，同时也剔除base
    window.addEventListener('popstate', e => {  // 当点击链接进行跳转时
        const current = this.current
    
  
        const location = getLocation(this.base)
        if (this.current === START && location === initLocation) {
            return
        }
    
        this.transitionTo(location, route => {
            if (supportsScroll) {
                handleScroll(router, route, current, true)
            }
        })
    })
```

## 实例化Vue
在应用的入口文件实例完了VueRouter后，这个时候我们就需要实例化Vue，实例Vue很简单，只是我们传多呢一个router(路由对象实例而已)，创建Vue实例后
我们将编译根组件，然后创建根组件，依次类推，在这个过程中每个组件都会触发beforeCreate钩子函数，这样我们就会执行当前我们在install中混入的beforeCreat
钩子函数，

```
exports function install(Vue){
    //...
    Vue.mixin({
        beforeCreate(){
            if (isDef(this.$options.router)) {
                    this._routerRoot = this  // this._routerRoot重新赋值为当前组件this
                    this._router = this.$options.router  // 指向当前vueRouter实例
                    this._router.init(this) // 路由初始化
                    // 当前组件实例上创建_route的响应式对象，
                    Vue.util.defineReactive(this, '_route', this._router.history.current)
                } else { // 当是子组件时，在根组件上添加_routerRoot属性，指向根组件的实例
                    this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
                }
                registerInstance(this, this)
        }
    })
   //...
}
```
上面介绍过这个钩子函数的一些作用，这里我们将详细介绍这其中的两个关键，this._router.init(this)和定义响应式_route对象。

#### _router.init(this)
在Vue实例化后，vm.$options.router指向VueRouter实例，这个时候调用VueRouter的原型方法init。
```
class VueRouter {
    //...
    init(app){
        this.apps.push(app) // app 当前组件的实例对象
        // 当前组件销毁时同时也清除apps中的组件实例引用。否则会导致内存溢出
        // https://github.com/vuejs/vue-router/issues/2639
        app.$once('hook:destroyed', () => {
            const index = this.apps.indexOf(app)
            if (index > -1) this.apps.splice(index, 1)
            if (this.app === app) this.app = this.apps[0] || null
        })
        if (this.app) {
            return
        }
        
        this.app = app
        
        const history = this.history // 获取当前history实例
        // history.getCurrentLocation()  获取当前页面的路径，path+location.search+location.hash  eg:/home?a=b
        if (history instanceof HTML5History) {
            history.transitionTo(history.getCurrentLocation())
        } else if (history instanceof HashHistory) {
            const setupHashListener = () => {
                history.setupListeners()
            }
            history.transitionTo(
                history.getCurrentLocation(),
                setupHashListener,
                setupHashListener
            )
        }
        
        history.listen(route => {
            this.apps.forEach((app) => {
                app._route = route
            })
        })
    }
    
    //...
}

```
这段代码，前面在处理一个bug，当前组件销毁时，VueRouter实例属性apps中还保持该组件的实例对象，没有及时销毁，导致内存溢出的危险
后面的主要是调用history.getCurrentLocation()获取当前页面对应的url，然后做完参数传给history.transitionTo。这个时候了解history.transitionTo就
显的非常重要

```
//参数列表 
//location：我们将跳往的目标地址

transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    //   调用 match 得到匹配的 route 对象  location：目标地址  this.current : 当前route信息
    const route = this.router.match(location, this.current)
    // 确认过渡
    this.confirmTransition(route, () => {
        this.updateRoute(route)
        onComplete && onComplete(route)
        this.ensureURL()
    
        // fire ready cbs once
        if (!this.ready) {
            this.ready = true
            this.readyCbs.forEach(cb => { cb(route) })
        }
    }, err => {
        if (onAbort) {
            onAbort(err)
        }
        if (err && !this.ready) {
            this.ready = true
            this.readyErrorCbs.forEach(cb => { cb(err) })
        }
    })
}
```
上述代码 调用router实例的match方法而在这个方法中我们其实调用的其实是this.matcher.match函数，记得我们在实例化Router时，在构造函数中创建了matcher属性，他的值
是调用createMatcher返回的，返回两个方法match 和  addRoutes，这个时候我们调用的其实是这个match函数，在这个函数中我们循环pathList，找到对应的pathMap项，然后返回一个和当前目标
location相对于的路由对象 {name: "Home",meta: {}, path: "/",hash: "",query: {} ,params: {},fullPath: "/" ,matched: [对应的pathMap项]}。

在这里我们通过目标路径，确认我们即将要去路由的所有路由对象信息（params,query,component等），接下来就是确认路径，然后渲染该路径对应的组件，接下来我们来看看
确认过渡这个函数 confirmTransition

```
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
    const current = this.current
    const abort = err => {
        if (isError(err)) {
            if (this.errorCbs.length) {
                this.errorCbs.forEach(cb => {
                    cb(err)
                })
            } else {
                warn(false, 'uncaught error during route navigation:')
                console.error(err)
            }
        }
        onAbort && onAbort(err)
    }
    // 如果当前的路由信息与将去往的路由信息相同，则直接返回
    if (
        isSameRoute(route, current) &&
        route.matched.length === current.matched.length
    ) {
        this.ensureURL()  // 如果当前地址栏url  与 current不一样  ，则修改为current， 只修改url  不涉及组件更新和页面跳转
        return abort()
    }
    // 每个路由对象都带有matched数组，一个路由对应的可能对应多个matched， 如 /foo/a 则对应两个，因为 /foo对应一个正则  /foo/bar对应一个正则
    // resolveQueue 通过对比目标的matched和现在的matched，确定需要更新哪些组件，eg: 目标：/foo/a  现在：/foo  既我们现在只需要更新foo对应的组件
    // 哪些需要更新  哪些不需要更新 updated:不需要更新的路由配置对象集合（组件） activated:需要重新创建的路由配置对象集合   deactivated：需要销毁的路由配置对象集合
    // eg /foo/a  跳往/foo/b   则 /foo对象的组件不需要更新（保留）  /foo/a 对应的组件需要销毁  /foo/b 对应的组件需要创建
    const {
        updated,
        deactivated,
        activated
    } = resolveQueue(this.current.matched, route.matched)
    // 生命周期队列，用于调用路由的各种钩子函数
    // 1：deactivated 组件调用beforeRouterLeave   2:全局beforeEach守卫   3：
    const queue: Array<?NavigationGuard> = [].concat(
        // 返回要销毁组件集合beforeRouterLeave钩子函数集合，其实返回的是bindGuard函数，
        // bindGuard函数中调用beforeRouterLeave钩子函数，this指向当前组件的实例，参数为调用bindGuard函数的参数
        extractLeaveGuards(deactivated), // 调用beforeRouterLeave钩子
        // global before hooks
        this.router.beforeHooks,  // 全局beforeEach守卫
        // in-component update hooks
        // 返回要更新组件的beforeRouterUpdate钩子函数集合，和上面的组件内独享钩子函数一样的处理方式
        extractUpdateHooks(updated),   // 调用 beforeRouterUpdate钩子函数
        // in-config enter guards
        //  beforeEnter：路独享的钩子函数，直接map这个activated（将要新建的路由配置）获取每个组件独享的beforeEnter
        activated.map(m => m.beforeEnter), // 调用beforeEnter钩子函数
        // async components
        resolveAsyncComponents(activated)  // 异步组件
    )
    // 当前要去往的路由配置项
    this.pending = route
    // hook:当前的路由钩子函数  next:回调runQueue中的step函数
    const iterator = (hook: NavigationGuard, next) => {
        if (this.pending !== route) {
            return abort()
        }
        try {
            // 调用钩子函数，  塞入函数  to:route  from:current  next
            hook(route, current, (to: any) => {
                // 当next都参数为false时，中断导航，调用ensureURL将浏览器地址恢复至current
                if (to === false || isError(to)) {
                    // next(false) -> abort navigation, ensure current URL
                    this.ensureURL(true)
                    abort(to)
                } else if ( typeof to === 'string' || (typeof to === 'object' && ( typeof to.path === 'string' || typeof to.name === 'string'))) {
                    // 跳转到另一个地址。
                    // next('/') or next({ path: '/' }) -> redirect
                    abort()
                    if (typeof to === 'object' && to.replace) {
                        this.replace(to)
                    } else {
                        this.push(to)
                    }
                } else { // 确认导航
                    // confirm transition and pass on the value
                    next(to)
                }
            })
        } catch (e) {
            abort(e)
        }
    }
    // queue 钩子函数队列
    runQueue(queue, iterator, () => {
        const postEnterCbs = []
        const isValid = () => this.current === route
        // wait until async components are resolved before
        // extracting in-component enter guards
        // 获取组件的beforeRouteEnter组件
        const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
        const queue = enterGuards.concat(this.router.resolveHooks)
        runQueue(queue, iterator, () => {
            if (this.pending !== route) {
                return abort()
            }
            this.pending = null
            onComplete(route)
            if (this.router.app) {
                this.router.app.$nextTick(() => {
                    postEnterCbs.forEach(cb => {
                        cb()
                    })
                })
            }
        })
    })
}
```
上述函数中，通过resolveQueue函数获取当前需要销毁组件(matched对象中含有对应的组件)deactivated，不需要更新的组件updated,需要重新创建的组件activated，
然后将beforeRouterLeave，beforeEach,beforeRouteUpdate,beforeEnter对应的路由钩子函数取出，然后进行执行，并且用resolveAsyncComponents函数去处理异步组件
，再次调用beforeRouteEnter钩子函数，当beforeRouteEnter钩子调用完毕后，我们进行当前组件，已经当前组件父集合的_route属性，该属性在install方法中被我们作为
响应式对象添加到了每个组件实例上，当他改变时，每个组件下子组件router-view都会重新进行渲染，加载当前路由对应的新组件。


