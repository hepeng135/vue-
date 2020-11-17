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
    if (oldVnode === vnode) {
        return
    }
    if (isDef(vnode.elm) && isDef(ownerArray)) {
        vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm
    //如果新旧VNode都是静态的，同时它们的key相同（代表同一节点），并且新的VNode是clone或者是标记了
      v-once，那么只需要替换elm以及componentInstance即可。
    if (isTrue(vnode.isStatic) &&
        isTrue(oldVnode.isStatic) &&
        vnode.key === oldVnode.key &&
        (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
        ) {
            vnode.componentInstance = oldVnode.componentInstance
            return
    }
    /*如果存在data.hook.prepatch则要先执行*/
    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
        i(oldVnode, vnode)
    }
    const oldCh = oldVnode.children
    const ch = vnode.children
    /*执行属性、事件、样式等等更新操作*/
    if (isDef(data) && isPatchable(vnode)) {
        for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
        if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    
    //如果这个vnode没有文本节点
    if (isUndef(vnode.text)) {
        //如果新节点和旧节点都拥有子节点，则调用updateChildren
        if (isDef(oldCh) && isDef(ch)) { 
            if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
        } else if (isDef(ch)) {//如果只有新节点拥有子节点时，
            //如果新节点有子节点但是旧节点没有，但是旧节点拥有文本节点时，先清空 然后在添加
            if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
            addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
        } else if (isDef(oldCh)) {//当新节点没有子节点而老节点有子节点的时候，则移除所有ele的子节点
            removeVnodes(oldCh, 0, oldCh.length - 1)
        } else if (isDef(oldVnode.text)) {
            //*当新老节点都无子节点的时候，只是文本的替换，因为这个逻辑中新节点text不存在，所以清除ele文本
            nodeOps.setTextContent(elm, '')
        }
    } else if (oldVnode.text !== vnode.text) {
        /*当新老节点text不一样时，直接替换这段文本*/
        nodeOps.setTextContent(elm, vnode.text)
    }   
    //调用postpatch钩子
    if (isDef(data)) {
        if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
}

```
通过源码我们知道patchVnode就是在做更新操作，包括：属性更新，文本更新，子节点更新，规则如下
1. 如果新旧VNode都是静态的，同时它们的key相同（代表同一节点），并且新的VNode是clone或者是标记了
v-once，那么只需要替换elm以及componentInstance即可。
2. 新老节点均有children子节点，则对子节点进行diff操作，调用updateChildren，这个updateChildren也是
diff的核心。
3. 如果老节点没有子节点而新节点存在子节点，先清空老节点DOM的文本内容，然后为当前DOM节点加入子节
点。
4. 当新节点没有子节点而老节点有子节点的时候，则移除该DOM节点的所有子节点。
5. 当新老节点都无子节点的时候，只是文本的替换。

属性更新和文本节点更新非常简单，简单的DOM操作就可以完成。但是子节点更新，我们需要通过新旧子节点的对比，然后才能做相应
的更新操作，这样我就需要在新子节点循环中去循环旧节点才能完成新旧节点的对比，但是在实际开发中，我们一般只是简单修改列表中
某项数据的内容，或者在列表中追加一些新的节点、或者列表中的节点移动位置，我们完全不需要每个都去对比，在Vue中对于这一块采用
了优化策略，我们来看下。

```
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]  //旧节点第一位
    let oldEndVnode = oldCh[oldEndIdx]  //旧节点结束位
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]  //新节点的第一位
    let newEndVnode = newCh[newEndIdx]  //新节点的结束位
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    const canMove = !removeOnly

    //如果旧节点的开始index小于等于旧节点的结束index  并且  新节点的开始index  小于新节点的结束index
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {//如果没有旧节点的第一位，则将oldStartIdx+1，获取第二位
        oldStartVnode = oldCh[++oldStartIdx]
      } else if (isUndef(oldEndVnode)) {//如果没有旧节点的最后一位，则oldEndIdx-1，获取倒数第二位
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode,newEndVnode )) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {

        //createKeyToOldIdx：新建一个map对象，key为oldStartIdx至oldEndIdx 中所有oldVnode的key，value为对应的index
        //map={key:index}
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)

        //如果newStartVnode存在key并且这个key在oldKeyToIdx中存在，则返回对应的index。
        //否则循环这个oldCh集合，起点：oldStartIdx，终点：oldEndIdx，每一个与newStartVnode进行对比，当有相同的时候，返回这个oldVnode的index
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // 对应newStartVnode在oldVnode集合中没有找到对应相同node的index时， 表示是全新的node，直接创建，
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          //对应newStartVnode在oldVnode集合中找到了对应相同node的index时，
          vnodeToMove = oldCh[idxInOld] //在oldVnode集合获取这个node
          if (sameVnode(vnodeToMove, newStartVnode)) {//相同的话直接调用patchVnode，更新这个node
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            //然后将这个老节点设置成undefined
            oldCh[idxInOld] = undefined
            //移动对应的node到当前oldStartVnode的前面
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            //相同的key但是元素名等其他的不一样时，直创建，然后插入
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    //全部对比完成，当oldStartIndex大于oldEndIndex时，说明老节点变比完毕，新节点比老节点多，将多出来的节点创建加入到真实dom中
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {//当newStartIndex大于newEndIndex时，表示新节点对比完毕，剩下的老节点需要一个个的从真实的dom中移除
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }

```
上述代码取出新子节点集合的‘新前’、‘新后’。旧节点集合的‘旧前’、‘旧后’进行交叉对比，会出现以下四种情况
注：根据‘新前’、‘新后’、‘旧前’、‘旧后’我们也得出他们的一些信息 newStartIndex(新前位置)、newEndIndex(新后位置)、oldStartIndex(旧前位置)、oldEndIndex(旧后位置)

1. 新前和旧前进行比较，如果相同执行以下操作。
    ```
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]

    ```
   调用patchVnode进行更新，然后更新索引值，产生新的新前和新后。
   
2. 新后和旧后进行比较，如果相同执行以下操作。
    ```
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]

    ```
    调用patchVnode进行更新，然后更新索引值，产生新的新后和旧后
    
3. 新后和旧前进行比较，如果相同执行以下操作
    ```
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
    ```
    调用patchVnode进行更新，因为这个时候已经不是相同位置了，我们还需要已经节点移动，节点更新完毕后，需要将节点移动到当前未处理
    旧节点最后一个的后面，既当前oldEndVnode的下一个兄弟节点的前面，如代码nodeOps.nextSibling(oldEndVnode.elm)取下一个兄弟节点，如果存在则用
    insertBefore否认则用appendChild。然后更新索引，产生新的新后和旧后
    
4. 新前和旧后进行比较，如果相同则执行以下操作
    ```
         patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
         canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
         oldEndVnode = oldCh[--oldEndIdx]
         newStartVnode = newCh[++newStartIdx]
    ```
   调用patchVnode进行更新，因为这个时候已经不是相同位置，同3中一样需要移动节点，但移动的位置是当前未处理旧节点最前面一个的前面，
   然后更新索引，产生新的新前和旧后
   

当前面的4中情况都没法满足的时候，我们就需要去遍历了，代码如下
```
    if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
    idxInOld = isDef(newStartVnode.key) ? oldKeyToIdx[newStartVnode.key] : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
    if (isUndef(idxInOld)) { 
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
    } else {
        vnodeToMove = oldCh[idxInOld] //在oldVnode集合获取这个node
        if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
    }
    newStartVnode = newCh[++newStartIdx]
```

如上述代码，当前4中情况都不满足的时候，我们遍历剩下的旧节点，createKeyToOldIdx函数，遍历剩下的旧节点，将节点的key取出来组成一个对象，对象的
key为节点的key，对象的value为节点的index，这个key组成的对象赋值给oldKeyToIdx，然后取newStartVnode的key去ldKeyToIdx看是否存在，既oldCh集合中是否有和
这个新开始节点拥有相同key的节点，如果没有，则认为newStartVnode是一个完全的新节点，我们需要创建，然后添加到视图中。如果存在，则将这个旧节点取出来，与newStartVnode对比
看是否是同一个节点，如果不是，则还是认为这是个新节点，如果不是则调用patchVnode进行更新，然后在更新位置，并将对应的旧节点设置为undefined，防止下次重复比较。
因为这次处理了newStartVnode,所有需要更新newStartIndex。


对应while的条件 ‘oldStartIndex<=oldEndIndex || newStartIndex<=newEndIndex’,当newCh集合的长度和oldCh集合的长度不一样时，我们总会有些节点是处理不到的。
当oldStartIndex<oldEndIndex这个条件不满足时，既oldStartIndex>oldEndIndex表示旧节点集合已经处理完毕，剩下的没有处理的新节点都是全新的节点，需要创建，然后插入。
当newStartIndex<=newEndIndex这个条件不满足时，既newStartIndex>oldStartIndex表示新节点结合已经处理完毕，剩下没有处理的旧节点都是需要remove的。
代码如下

```
    if (oldStartIdx > oldEndIdx) {
        refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
        addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {//当newStartIndex大于newEndIndex时，表示新节点对比完毕，剩下的老节点需要一个个的从真实的dom中移除
        removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
```
