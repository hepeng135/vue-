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
```
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