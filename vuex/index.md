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
如上述代码所示，在入口我们先导入主要Store，install，然后就是一些辅助方法，然后在将其用两种方法导出。下面我们先主要
分析一下Store和install

#### 安装Store插件的Install
在Vue安装插件需要调用Vue.use(plugin)来安装对应插件，在Vue中use的实现其实就是调用当前插件身上的install方法进行安装，
但是在Store对象上并没有install方法，那是怎么实现的，在store.js文件中我们需要注意如下代码。
```
//store.js文件中的部分代码
import applyMixin from './mixin'
export class Store {
    constructor (options = {}) {
    
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        //code. ...
    }
}
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
```
