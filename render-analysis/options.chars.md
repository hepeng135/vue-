#### options.chars:处理空白节点或者纯文本节点


#### 参数详解
    @params text : 当前文本标签或者空白标签
    @params start  起点位置
    @params end 终点位置
```

options.chars=chars (text: string, start: number, end: number) {
      
    //Ie中textarea属性placeholder的bug,
    //Ie中textarea的placeholder的值会在textarea标签中显示
    if (isIE && currentParent.tag === 'textarea' &&currentParent.attrsMap.placeholder === text) {
        return
    }
    //获取当前最近的父级的子集
    const children = currentParent.children

    // 当前是否在pre标签中  或者  text是除了空格还有其他值。
    if (inPre || text.trim()) {
        //isTextTag  确定当前父级标签是否 script 或者 style。
        text = isTextTag(currentParent) ? text : decodeHTMLCached(text)

    }
     //这里还有一段关于是否压缩情况下的空格的一直处理，直接省略，只要是空格，统一处理成空 
    else {
        text = ''
    }
    //当有text的时候，直接作为文本标签去处理
    //parseText解析文本中的表达式,eg:<p>{{message1 | add}}111 {{message2}} 222</p>返回的结果
    /* {
            expression:"_s(_f("add")(message1))+"" message""+_s(message2)+"" 222"",  
            tokens:[{@bind:"_s(_f("add")(message1))"},"" 111"",{@bind:"_s(message2)"},"" 222""] 
    /*}
    
    if (text) {  
        let res
        let child: ?ASTNode
        if (!inVPre && text !== ' ' && (res = parseText(text, delimiters))) {
            child = {
                type: 2,
                expression: res.expression,
                tokens: res.tokens,
                text
            }
        } else if (text !== ' ' || !children.length || children[children.length - 1].text !== ' ') {
            child = {
                type: 3,
                text
            }
        }
    }
    //添加到当前标签的子集中
    if (child) {
        if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
            child.start = start
            child.end = end
        }
        children.push(child)
    }
}
}
```