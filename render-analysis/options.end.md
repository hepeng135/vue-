#### end:end钩子函数，处理结束标签，维护stack数组，更新currentParent,并调用closeElement函数


```
function end (tag, start, end) {
    //获取需要闭合的标签
    const element = stack[stack.length - 1]
    // 从stack中删除这个标签
    stack.length -= 1
    //更新currentParent
    currentParent = stack[stack.length - 1]
    if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        element.end = end//更新标签结束的位置
    }
    closeElement(element)
}
```

#### closeElement  闭合标签时处理一些事物

#### 函数的作用 
    1：processElement函数解析标签的 v-slot slot-scope scope属性，以及是slot标签时解析name，component标签或者组件时解析 is inline-template属性  
    2：处理带有v-else和v-else-if的标签，带有这些属性的标签不会被添加到父级中children，而是添加同级上一个带有v-if标签的ifConditions 中
    3:确定父级自子集关系（带有elesif 或者 else属性的el不会去和currentParent确定子父级关系）

```
function closeElement (element) {
    //移除children中的空白标签
    trimEndingWhitespace(element)
    //inVPre：如果当前没有拥有v-pre指令。
    //element.processed：是否处理过 class style ref key  slot component 等特殊的属性
    if (!inVPre && !element.processed) {
        element = processElement(element, options)
    }
    // 当前stack数组为空并且当前的element和root不是同一个元素时
    if (!stack.length && element !== root) {
        //当root和最后一个element是并存关系时，必须是if和else的关系
        if (root.if && (element.elseif || element.else)) {
            addIfCondition(root, {
                exp: element.elseif,
                block: element
            })
        }
    }
    //如果当前标签有父级  并且 不是script或者style标签
    if (currentParent && !element.forbidden) {
        //如果当前标签拥有elseif  或者 else 属性
        if (element.elseif || element.else) {
            //寻找到当前上一个兄弟标签，并增加对应的条件,表示满足这个条件则显示这个
            //prevSublingElement.ifConditions.push({exp:expression,block:element})
            processIfConditions(element, currentParent)
        } else {
            //如果当前标签带有作用域插槽
            if (element.slotScope) {
                //获取插槽的名字，默认为default
                const name = element.slotTarget || '"default"';
                //想父级的currentParent.scopedSlot新增一个 以slotName为key的对象，值为el。
                ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
            }
            //el和currentParent互相正式添加父子关系
            currentParent.children.push(element)
            element.parent = currentParent
        }
    }

    // final children cleanup
    // filter out scoped slots
    //过滤el.children，清除el.children中带有slotScope的标签
    element.children = element.children.filter(c => !(c: any).slotScope)
    //清除el.children下的空白节点
    trimEndingWhitespace(element)

    // 更新全局变量  inVPre  inPre；
    if (element.pre) {
        inVPre = false
    }
    if (platformIsPreTag(element.tag)) {
        inPre = false
    }
    // apply post-transforms
    for (let i = 0; i < postTransforms.length; i++) {
        postTransforms[i](element, options)
    }
}


```
