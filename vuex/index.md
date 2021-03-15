Vue的核心插件Vuex，用来管理应用中的状态，下面我们将一起来看看Vuex的源码，并对一些常用的API进行讲解。

## 目录接口
Vuex的源码被托管在github上，我们首先通过git将代码clone下来，用心仪的IDE打开，文件目录如下。

![](./image/src.png)

如图所示，Vuex的主要代码都在src文件夹下，examples文件夹下放一些栗子，解析源码时，我们运行栗子进行断点调试进行跟进会更容易理解，首先npm install安装一下依赖包
在运行前我们需要修改一个地方，首先先找到目录中的package.json文件，注意scripts中的第一项 "dev": "node examples/server.js" ，我们运行
npm run dev 对应的就好走这行,既会自动 node examples/server.js 命令，node会启动example下的server.js,我们打开examples文件，看到一些栗子、
server.js 和 webpack.config.js文件，在server文件我们会启动一个node的本地服务器，在里面会用webpack进行打包，对应的webpack配置文件就是 webpack.config.js，
在这个配置文件中我们加上 devtool: 'source-map'，使得我们在打断点时可以打在源src文件上，不用在编译后的代码中进行调试。

## 源码解析

#### 从入口开始
```
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'
import createLogger from './plugins/logger'

export default {
    Store,
    install,
    version: '__VERSION__',
    mapState,
    mapMutations,
    mapGetters,
    mapActions,
    createNamespacedHelpers,
    createLogger
}

export {
    Store,
    install,
    mapState,
    mapMutations,
    mapGetters,
    mapActions,
    createNamespacedHelpers,
    createLogger
}

```
如上述代码所示，我们一目了然的看到Vuex对外暴露的API，在入口我们先导入主要Store，install，然后就是一些辅助方法，然后在将其用两种方法导出。下面我们先主要
分析一下Store和install

#### 安装Store插件的Install方法
在Vue安装插件需要调用Vue.use(plugin)来安装对应插件，在Vue中use的实现其实就是调用当前插件身上的install方法进行安装，
但是在Store对象上并没有install方法，那是怎么实现的，在store.js文件中我们需要注意如下代码。
```
//store.js文件中的部分代码
import applyMixin from './mixin'
//code...
let Vue
export class Store {
    constructor (options = {}) {
    
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        //code. ...
    }
}

```
在store.js中先设置一个全局变量Vue，在Store构造函数中判断当前是否拥有全局Vue，并且当前环境拥有全局引用Vue，也就是window上有Vue这个对象，
然后执行install，对全局变量Vue的判断主要保证install只执行一次，然后我们看看install方法
```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (__DEV__) {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  // applyMixin 在2.0版本中：每个组件混入beforeCreate构造函数，在每个组件实例上挂载$store属性，使得在每个组件实例中我们都能访问$store  既this.$store
  applyMixin(Vue)
}

//mixin.js文件
//省略版本控制部分
export default function(Vue){
    Vue.mixin({ beforeCreate: vuexInit });

    function vuexInit(){
        const options = this.$options
        // store injection
        if (options.store) {
            this.$store = typeof options.store === 'function'
                ? options.store()
                : options.store
        } else if (options.parent && options.parent.$store) {
            this.$store = options.parent.$store
        }
    }
}
```
install调用时我们比较一下当前的Vue和window.Vue,确保install只执行一次，在将window.Vue赋值给Vue,最后调用applyMixin，
在applyMixin函数中我们用Vue.mixin混入beforeCreate钩子函数，该钩子函数确保我们在Vue的任何组件上我们都可以通过thus.$store
来访问当前的Store实例。

#### 认识Store构造函数
我们在使用Vuex的时候，通常需要传入一个对象去实例化Store类，这个对象包含state、mutations、actions、getters、modules,那么在实例化的过程中
到底做过什么，让我们来来看看下面的代码
```
export class Store {
    constructor (options = {}) {
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        if (__DEV__) {
            assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
            assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
            assert(this instanceof Store, `store must be called with the new operator.`)
        }
        const {
            plugins = [],
            strict = false
        } = options

        this._committing = false
        this._actions = Object.create(null)
        this._actionSubscribers = []
        this._mutations = Object.create(null)
        this._wrappedGetters = Object.create(null)
        this._modules = new ModuleCollection(options)
        this._modulesNamespaceMap = Object.create(null)
        this._subscribers = []
        
        this._watcherVM = new Vue()
        this._makeLocalGettersCache = Object.create(null)
        
        // bind commit and dispatch to self
        const store = this
        const { dispatch, commit } = this
        this.dispatch = function boundDispatch (type, payload) {
            return dispatch.call(store, type, payload)
        }
        this.commit = function boundCommit (type, payload, options) {
            return commit.call(store, type, payload, options)
        }
        this.strict = strict
        const state = this._modules.root.state
       
        installModule(this, state, [], this._modules.root)
        resetStoreVM(this, state)
        plugins.forEach(plugin => plugin(this))
        const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
        if (useDevtools) {
            devtoolPlugin(this)
        }
    }
}
```
首先是我们之前说到的install函数的调动，然后在开发环境下Vuex进行了一些断言判断操作。

断言：必须保证当前Vue的存在，因为后面的时候我们会依赖他
```
assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
```
断言：必须保证当前浏览器环境必须兼容Promise，否则需要提供一个Promise polyfill
```
assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
```
断言：当前对象必须是Store的实例对象。既通过new Store创建的。
```
assert(this instanceof Store, `store must be called with the new operator.`)
```
然后我们再看看assert函数是怎么实现的,代码如下，实现非常简单，通过判断传进来的第一个参数（不为真），然后抛出第二个参数（错误描述）
```
export function assert (condition, msg) {
    if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
回到Store构造函数中，接下来将看到一些属性初始化代码
```
//记录当前state的改变是否通过commit
this._committing = false
//存放模板所有的action
this._actions = Object.create(null)
//存放通过API subscribeAction 添加的订阅store的action
this._actionSubscribers = []
//存放所有模板的mutation
this._mutations = Object.create(null)
//存放所有模板的getter
this._wrappedGetters = Object.create(null)
//处理Store中的模块
this._modules = new ModuleCollection(options)
//存放当前命名空间模块 eg: {a:module,b:module,''a/c':module}
this._modulesNamespaceMap = Object.create(null)
//存放所有通过API subscribe 添加的订阅store的mutation。
this._subscribers = []
// 创建vue实例
this._watcherVM = new Vue()
```
在属性初始化中，我们先中点介绍this._modules，他通过创建ModuleCollection类的实例进行初始化，传入的options和创建Store类的实例options
一样，接下来我们看看这个ModuleCollection类。

```
export default class modeModuleCollection {
    constructor(rawRootModule){
        this.register([], rawRootModule, false)
    }

    register(path, rawModule, runtime = true){
        if (__DEV__) {
            // 验证getter  mutations actions ,getter、mutations 必须是函数,actions可以是函数、对象(则：actions.handler 必须是函数
            assertRawModule(path, rawModule)
        }
        // 实例化Module类
        const newModule = new Module(rawModule, runtime)
        // 通过当前的path判断当前是根模块还是 子集模块
        if (path.length === 0) {
            this.root = newModule// 根模板
        } else { // 子模板
            // 获取父级模板
            const parent = this.get(path.slice(0, -1))
            // 添加子模板
            parent.addChild(path[path.length - 1], newModule)
        }
        
        // register nested modules // 子集模板
        // 循环子集模板，然后注册
        if (rawModule.modules) {
            forEachValue(rawModule.modules, (rawChildModule, key) => {
                this.register(path.concat(key), rawChildModule, runtime)
            })
        }
    }
}
```
在ModuleCollection类的构造函数中，我们直接调用它的原型方法register，那么我们就来看下这个register方法,首先做的也是断言验证，断言当前
对应的module中的getter、mutations必须是函数，actions可以是函数、对象(则：actions.handler 必须是函数）

```
  assertRawModule(path, rawModule)
```
在register中接下来是实例化Module，传入当前的module，如果当前的path，既module的key路径为空数组，则代表当前是根模板，在赋值到this.root上，
如果当前的path不为空的的话，则代表当前的模板存在子集，则调用this.get获取父级模板，然后父级模板实例调用其原型方法addChild添加这个子模板
```
// 实例化Module类
const newModule = new Module(rawModule, runtime)
// 通过当前的path判断当前是根模块还是 子集模块
if (path.length === 0) {
  this.root = newModule// 根模板
} else { // 子模板
  // 获取父级模板
  const parent = this.get(path.slice(0, -1))
  // 添加子模板
  parent.addChild(path[path.length - 1], newModule)
}
```
register方法是个递归函数，我们需要监测每个module的子级，然后实例，然后做添加操作，使其形成一个链式结构。我们通过判断当前
模板是否存在modules属性，既是否存在子级模板，如果存在则循环子模板，然后调用register。
```
// register nested modules // 子集模板
// 循环子集模板，然后注册
if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
    })
}
```

综上所属在register方法函数中，我们主要是将传进来的module模块进行实例化，然后根据其子父级的关系创建一个链式结构，并赋值到this.root上，下面
我们看看在register函数中调用的Module类的具体代码
```
export default calss Module {
    constructor (rawModule, runtime) {
        this.runtime = runtime  // 当前是初始化  还是运行时
        // 创建_children属性
        this._children = Object.create(null) // 创建一个纯对象
        //当前的module
        this._rawModule = rawModule  
        //获取出当前模块的state
        const rawState = rawModule.state // 存储当前的state ,未加工的state
        this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
    }
}
```
在Module类的构造函数中，主要设置了一个_children属性和获取当前module的state。回到register函数中，通过path去判断当前是否根模板，是的话直接挂
到this.root上，否则我们调用ModuleCollection类的原型方法get，获取当前module的上一个module，既当前模块的父模块，我们来看看这个get方法
```
class ModuleCollection {
    
    //code...
    get (path) {
         // reduce 进行循环，初始值为当前的根对象模板。循环的为当前模块的名称
         // 返回对应moduleKey的modules项
         console.log(path)
         return path.reduce((module, key) => {
            console.log(module)
            return module.getChild(key)
         }, this.root)
    }
    //code...
}
```
循环传进来的path，以this.root(根模块)做个起点，依次调用当前模板实例的原型方法getChild,this.root指向通过Module类创建的实例，那我们
现在看看Module类的原型方法getChild
```
class Module {
    //code...
    getChild (key) {
        return this._children[key]
    }
    //code...
}
```
getChild函数接收一个key作为参数，返回当前模板的子模块集合中的具体一个子模块。回到ModuleCollection类的register方法，调用get是传入的参数
为path.slice(0,-1),既去掉当前最后一个元素（当前模板的key）去取当前模板的父级模拟。然后在调用父模板的addChild方法添加子级模块为当前模块.
```
class Module {
    //code
    addChild (key , module){
        this._children[key]=module;
    }
    //code
}
```
回调最开始的Store类的构造函数中this._modules = new ModuleCollection(options)做的作用我们可以这样来理解，创建一个this.root执行根
模板(执行rootModule)，根模板拥有_children，里面放子模板，依次类推，形成一个链式结构。
```
    new Store({
        state,
        mutations,
        actions,
        modules:{
            a:ModuleA,
            b:ModuleB,
            c:moduleC
        }
    })
   //经过ModuleCollection实例化以后，
    this._modules.root={
        _rawModule:{state,mutations,actions},
        state:state,
        _children:{
            a:{},
            b:{},
            c:{}
        }
    }
```
接下来我们来看Store构造函数中对dispatch和commit的处理
```
const store = this
// 从当前实例对象上解析出dispatch commit
const { dispatch, commit } = this
// 重新定义dispatch、commit属性
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}

```
如上所示，从当前实例上解析出dispatch与commit，然后重新在当前实例上定义dispatch、commit方法，后续我们将介绍这两个方法的具体实现。
下面我们将接着介绍Store构造函数中剩下的代码
```
// strict mode当前Vue Store是否严格模式，默认为false
this.strict = strict
// 获取当前的rootState
const state = this._modules.root.state

// 注册commit actions getter
installModule(this, state, [], this._modules.root)

//初始化state和getter
resetStoreVM(this, state)

// apply plugins// 安装插件
plugins.forEach(plugin => plugin(this))
```
在我们处理完模板关系后，接下来就是处理模板中 state、commit、actions、getters，在上述代码中，我们首先





