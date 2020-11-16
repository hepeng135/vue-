#### patch是虚拟DOM最核心的部分，在Vue中负责将虚拟DOM转换成真正的DOM，patch也可以称为patching算法，通过他将有变化的DOM节点进行更新，而不是暴力的移除上一次的DOM，添加新的DOM。

先了解下虚拟DOM：虚拟DOM树是描述状态与DOM树之间的映射关系，将状态与DOM树之间的映射关系生成一个虚拟DOM树对象，在Vue中，通过模板(.vue)文件来
描述状态(js数据)与DOM树机构之间的映射关系，通过Vue的编译模板函数生成一个render函数，执行这个render我们就会得到一个虚拟DOM对象，然后就可以将
这个虚拟DOM树创建出来真正的DOM,并渲染到页面上。

在Vue中，我们通过编译模板得到render函数，然后为这个组件创建一个watcher，在watcher.getter()我们执行render函数，生成虚拟DOM的同时也可以收集依赖，
然后通过patch对比新旧的虚拟DOM将虚拟DOM渲染到页面上。

当是第一次渲染的时候不会拥有旧的虚拟DOM，这个时候不需要对比，直接创建，然后渲染。
```
 vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)

```
当不是第一次渲染时，这个时候存在新旧虚拟DOM，这个时候我们就需要对新旧虚拟DOM进行对比，然后对发生变化的DOM才进行处理，对于发生变化的
DOM，我们无非有这几种处理方式，新增、删除、更新


下面的这些操作都是在DOM树的同一结构层去对比，然后进行操作的。
#### 新增：
1. 当newVnode中拥有该节点，但是oldVnode中不拥有这个节点时，我们认为newVnode中的这个节点是新增加的。
2. 当newVnode中的节点和oldVnode中的节点完全不同时（tagName、tagNodeType不同）时，我们需要先删除然后在新增

#### 删除
1. 当newVnode中没有改节点，但是oldVnode中拥有这个节点，这个时候我们需要做删除操作。

#### 更新
1. 当newVnode和oldVnode中都拥有某个节点（tag，key等信息相同时），表示需要做更新操作

 对应上述的操作来说，新增和删除既直接执行appendChild、insertBefore、removeChild这些DOM操作就好，因为Vue中要兼容其他平台（weex等）做了封装，所以
 看起来有点麻烦，但是本质上都是这些DOM操作方法，对应更新操作，我们看下Vue中的patchNode函数，这个很函数时专本做标签更新的，包括属性更新、文本更新，子节点更新
 
```
function patchVnode(oldVnode,vnode,insertedVnodeQueue,ownerArray,index,removeOnly){
    
}

```