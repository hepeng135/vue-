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










