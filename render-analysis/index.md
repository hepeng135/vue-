## vue源码剖析——模板编辑生成渲染函数

### 什么是render函数
我们在编写.vue文件时一般情况下都需要在template中编写html模板，render函数就是将通过将html模板解析转成而成的函数，
也就是模板的渲染函数。由模板生成render函数时需要经历以下几个过程。

1. 解析器，将Html模板解析成ASTElement
2. 优化器，遍历生成的ASTElement，为每个节点添加静态标记
3. 生成器，将ASTElement生成render函数（渲染函数）


### 解析器
将模板解析成ASTElement,模板解析器中我们可以细分为  文本解析、过滤器解析、html解析。
文本解析器：用来处理文本节点，解析出文本节点中表达式，也可以是纯文本节点。
过滤器解析：将表单式中中的过滤器提取出来。
html解析器：解析标签，以及标签上所有的属性。
在解析过程中，先用正则去匹配确认当前是开始标签或者接受标签或者文本标签，根据不同的结果去调用对应的钩子函数 start(解析开始标签，获取标签上的所有属性),end(处理标签上的所有属性),charts(处理文本节点或者空节点),然后生成对应的ASTElement节点，ASTElement节点由标签上所有的信息组成的json对象，一个ASTElement表示一个节点，对象上的每个属性表示对应的节点信息，如下

```
{
    type:[Number]标签类型   1：元素
    tag:[String] 标签名称
      
    attrList:[Array]  属性集合,[{name:attrName,value:attrValue,start:开始位置,end:结束位置}]
    attrsMap:[Object] 属性集合，与attrList对应 {attrName:attrValue}
    rawAttrsMap:[Object] 属性集合 与attrList对应  {attrName:{name:attrName,value:attrValue,start:开始位置,end:结束位置}}
    
    dynamicAttrs : [attrJson] 属性名是绑定的集合
    attrs:[] 属性名不是绑定的集合    

    eg：<div>   <:为开始位置 ，>：为结束位置
    start:[Number],当前标签开始的位置  
    end:[Number],当前标签结束的位置

    parent:[Object] 当前标签的父级
    children:[Array] 当前标签的子级标签

} 
```
上面仅仅显示ASTElement中比较常见的属性信息，关于节点上的指令属性，绑定属性，特殊属性的表现可以查看[ASTElement详解](./ASTElement详解.md)
#### html解析器
html解析器的主要流程如下图所示

![](../image/parseHTML.png)


如上图所示，在parse函数中调用parseHTML函数时我们提供了四个钩子函数，start(处理开始标签)、 end(处理结束标签)、chars（处理文本标签）、comment（处理注释），这边我们
只需要特别关注start，end,chars这三个钩子函数

```
   parseHTML(template,{
        start(){}
        end(){}
        chars(){}
        comment()
   }) 

```
通过上述的钩子函数，在正则匹配到不同标签位置时调用不同的函数，然后组成ASTElement,下面我们来具体看一下这几个钩子函数，如何形成一个闭环

* 当是开始标签时的钩子函数，会利用参数tag，attrs，unary创建一个ASTElement,并从attrs中取出v-pre,v-for ,v-if, v-once,进行处理，并将对应
的属性挂载到ASTElement上，将ASTElement push 进stack集合，同时确定当前模板的root级，以及默认当前的父级为element

```
    let inVPre = false //默认不拥有v-pre指令
    let currentParent
    let root
    let stack
    export function createASTElement (
      tag: string,
      attrs: Array<ASTAttr>,
      parent: ASTElement | void
    ): ASTElement {
      return {
        type: 1,
        tag,
        attrsList: attrs,
        attrsMap: makeAttrsMap(attrs),
        rawAttrsMap: {},
        parent,
        children: []
      }
    }

    参数列表
    @params tag:标签名
    @params attrs:标签属性集合
    @params unary:当前是否单个可闭合标签
    @params start:正则符合开始的地方
    @params end:正则符合结束的地方
    start(tag, attrs, unary, start, end){
        //创建一个ASTElement
        let element: ASTElement = createASTElement(tag, attrs, currentParent);
        //检测当前标签是否有v-pre属性，并想element上添加pre属性，表示当前标签以及子集不需要处理上面的表达式
        processPre(element)
        if (element.pre) { 
            inVPre = true
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
            currentParent = element
            stack.push(element)
        } else {
            closeElement(element)
        }
    }

```

* 当是结束标签时的钩子函数，确定当前标签的直属父级currentParent，将当前标签作为currentParent的children,从stack集合中获取当前结束的是哪个element，默认就是stack的最后一个元素，然后从stack删除这个element，在确定该元素
父级，既此时stack的最后一个元素，并调用closeElement函数去处理v-if系列（v-else-if 、v-else）指令 、作用域插槽的（v-slot、scope、slot-scope、slot）指令或属性、组件系列（is、inline-template、ref）属性、自定义系列（指令，属性，事件）

```
    end (tag, start, end) {
        const element = stack[stack.length - 1]
        stack.length -= 1
        currentParent = stack[stack.length - 1]
        closeElement(element)
    },
```

* 当是文本标签时

```

```    






