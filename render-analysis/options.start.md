#### options.start:主要用来处理特殊标签（input）、当前属性名是一些自定义指令或者Vue自带指令(v-for、v-if、v-else-if、v-els、v-pre、v-once)


#### 函数背景
    定义在parse函数中，在parse函数中调用parseHTML时，作为参数传递进来。
    在该函数中用到的parse所在文件中的全局变量
    currentParent：当前标签的父级目标
    root：当前模板的根元素
    



#### 函数主要的作用
    1：创建一个完整的数据标签json
    2：处理特殊属性 v-pre v-for v-if v-once
    3：当前标签是否root根元素，既该组件所属html模板的最外层元素（非template）。
    4:确定当前的父级元素
####参数
    @params tag:标签名
    @params attrs:标签属性集合
    @params unary:当前是否单个可闭合标签
    @params start:正则符合开始的地方
    @params end:正则符合结束的地方

   
   
   
   
```
start (tag, attrs, unary, start, end) {
      
    //获取ns，假如currentParent上没有，就调用platformGetTagNamespace函数，该函数确定tag是svg  还是 math ,返回对应的值
    //ns可能为 svg  或者 math  两种特殊的标签
    const ns = (currentParent && currentParent.ns) || platformGetTagNamespace(tag)
    //处理ie下svg的兼容问题
    if (isIE && ns === 'svg') {
        attrs = guardIESVGBug(attrs)
    }
    //创建一个完整的标签数据json,元素标签 
    /*{
        type:1,标签类型，元素标签
        tag：tagName 标签名
        attrsList:[attr<Object>,...] 标签的属性集合
        attrMap:{attrName1:attrVal1,...} 标签的属性集合
        parent：标签的父级
        children:[]标签的子集集合
    }*/
    
    let element: ASTElement = createASTElement(tag, attrs, currentParent)
    //确定当前是否是svg或match标签，并给element添加上该属性
    if (ns) {
        element.ns = ns
    }
    //element上添加start，end，rawAttrsMap属性
    //  rawAttrsMap:{attrName:attr}
    if (process.env.NODE_ENV !== 'production') {
        if (options.outputSourceRange) {
            element.start = start
            element.end = end
            element.rawAttrsMap = element.attrsList.reduce((cumulated, attr) => {
                cumulated[attr.name] = attr
                return cumulated
            }, {})
        }
    }
    //判断当前标签是否style、script 并且不再服务端
    if (isForbiddenTag(element) && !isServerRendering()) {
        element.forbidden = true
    }
    
    //处理input。当不是input类标签时preTransforms函数返回undefined
    for (let i = 0; i < preTransforms.length; i++) {
        element = preTransforms[i](element, options) || element
    }
    //默认当前标签及其子级需要进行编译处理
    if (!inVPre) {
        //检测当前标签是否有v-pre属性，并想element上添加pre属性，表示当前标签以及子集不需要处理上面的表达式
        processPre(element)
        if (element.pre) { 
            inVPre = true
        }
    }
    //检测当前标签是否pre标签
    if (platformIsPreTag(element.tag)) {
        inPre = true
    }
    if (inVPre) {
        //当前拥有v-pre属性时，也是处理v-pre
        processRawAttrs(element)//处理pre标签
    } else if (!element.processed) {//处理标签上的v-for,v-if,v-once属性
        // structural directives
        processFor(element)
        processIf(element)
        processOnce(element)
    }
    
    if (!root) {//确定根元素
        root = element
    }
    //如果当前不是单个可闭合标签  
    if (!unary) {
        //确定当前父级
        currentParent = element
        stack.push(element)
    } else {
        closeElement(element)
    }
}

```