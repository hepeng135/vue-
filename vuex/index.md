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

