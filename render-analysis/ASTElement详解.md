
#### ASTElement [JSON] : 通过解析html模板生成标签信息json。

#### 绑定属性的val和一般属性val的区别
    绑定的值： 直接取出，进行过滤器解析，然后得出 xxxx （字符串）;
    普通的值：直接取出，用JSON.stringify处理下，得出  "xxxx" （带有双引号的字符串）


#### 元素标签
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

    staticClass:[String] 静态class的值。
    classBinding:[String] 动态绑定class的表达式, 
    当带有过滤器时  f("filter1")(expression,arg1,arg2,...)
    多个过滤器时 f("filter2")(f("filter1")(expression,arg1,arg2,...),arg1,arg2,....)    

    staticStyle:[String] 静态style的值,
    styleBinding:[String] 动态绑定styled的表达式, 可能存在过滤器

    forbidden:true| false  是否style或者script标签

    hasBindings：当前有自定义的绑定属性 （除一些vue自带的绑定属性，如class style is 等）

    //解析v-pre指令
    pre:true | false  当前元素已其子元素是否需要编译

    //value in data | (value,index) in data | (value,name,index) in data
    //processFor解析v-for指令所得  eg : (item,key,index) in message
    for: message
    alias:item
    iterator1：key
    iterator2: index
    
    //processIf函数解析v-if，v-else,v-else-if指令
    //在closeElement函数中，挂载v-else 和 v-esle-if的el都将放在if的这个el.ifConditions中
    //<p v-if="state==1"></p><p v-else-if="state==2"></p><p v-else==3></p>
    if:"state==1",
    ifConditions:[
        {exp:'state==1',block:if-P-json}
        {exp:'state==2',block:else-if-P-json}
    ]   {exp:'state==3',block:v-else-P-json} 
    else:true  当el带有v-else属性
    elseif：elseifExpression    

    //processOnce解析v-once
    once: true  当前标签拥有v-once
    
   
    //:key | key 绑定key或者普通key
    key:keyExpression  循环时用到的唯一key，可能存在过滤器
    
    //:ref | ref 绑定ref或者普通 ref
    ref:refExpression  可能存在过滤器

    //:is 
    component:componentName
    //具名插槽与插槽作用域
    scopedSlots:{
        slotName:{elJson,scopeSlot:作用域，slotTarget：slot的名字，slotTargetDynamic：slot的名字是否绑定}
    }

    //调用prop修饰符的自定义属性
    props:[{bidngsPropsJson,....}]

    //其他属性
    el.attrs:[{bindsAttrJson,...}]

    //绑定事件
    events:{eventName:{eventHandleFn,start,end,dynamic}}

    //自定义指令
   directives={[],[]}


    
    
}
```

#### type:2 带有表达式的文本节点
```
eg:<p>{{message1 | add}}111 {{message2}} 222</p
{
    type:2,
    start:开始位置
    end:结束位置
    expression:"_s(_f("add")(message1))+"" message""+_s(message2)+"" 222"", 
    tokens:[{@bind:"_s(_f("add")(message1))"},"" 111"",{@bind:"_s(message2)"},"" 222""] 
    text:"{{message1 | add}}111 {{message2}} 222"
}

```

#### type:3 只有文本或者空格的纯文本节点

