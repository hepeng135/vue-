#### handleStartTag函数：对匹配出来的属性值进行数据格式处理，将attrItemArr转换成arrtItemJSON

#### 函数背景
    该函数被定义在parseHTML函数中，作为parseHTML的局部函数。
    在该函数中用到的parseHTML函数的局部变量：
    stack:存储属性值被处理好的标签信息。
    lastTag：更新为已经处理好的标签名称作为上一个标签。
    
#### 函数主要的作用：
    
    1：通过函数parseStartTag，我们将匹配开始标签和标签上的属性简单的匹配出来,
    {tagName:'标签名',attrs:[matchAttrArr1,matchAttrArr2],start:'标签开始的位置'，end:'标签结束的位置',unarySlash:'标签结束标识符一般为空或者/'}
    
    2：将这个匹配出来的数据作为参数传进handleStartTag中进行进一步处理
    
    3：通过for循环将每个attrs中的每个属性进行数据格式处理，处理成以下格式
        {name:attrName,value:attrVal, start:属性开始位置,end:属性结束位置}
    
    4:判断当前标签是否有结束标识，没有的话就添加到stack<Array>全局变量中,并将lastTag更新为当前标签
    
    5：调用option.start函数，对标签上的属性进行进一步区分处理
#### 参数具体类容举例
    {
        tagName:'div',
        attrs:[
            ["class="app"","class","=","app",undefined,undefined,index:0,start:4,end:16]
            [属性全称 , 属性名 , = , 属性值(没有过滤器时) , 属性值(拥有过滤器时) ， 其他情况下的属性值 ，匹配的位置 ， 属性开始的位置，属性结束的位置]
        ]，
        unarySlash：当前标签结束的符号，为空或者"/"
        start:"标签开始的位置"，
        end:"标签结束的位置"
    }

#### 函数内容

```
function handleStartTag (match) {
    const tagName = match.tagName //获取标签名
    const unarySlash = match.unarySlash //获取当前标签结束时的标识符，
    
    //expectHTML:确定是html字符串
    if (expectHTML) {
      //上一个处理完成的标签是否p标签，并且当前标签是块级元素,表示当前模板字符串中p标签中套用呢块元素
      // p便签中不能包括块元素，次数直接闭合p标签，然后在处理当前标签
      if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
        parseEndTag(lastTag)
      }
      //自动闭合标签，并且上一个标签和当前标签相同，需要测试
      if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
        parseEndTag(tagName)
      }
    }
    //isUnaryTag 检测当前标签是不是单个、不用成对出现的标签,如input hr br ||  符号转换成Boolean值
    const unary = isUnaryTag(tagName) || !!unarySlash
    //获取匹配出标签属性的数量
    const l = match.attrs.length
    const attrs = new Array(l)
    //循环处理每一个匹配出来的属性数组，进行进一步数据格式加工
    //通过for循环将属性数组处理成json格式
    /*{
        name:attrName,
        value:attrVal,
        start:属性开始位置
        end:属性结束位置
    }*/
    for (let i = 0; i < l; i++) {
      const args = match.attrs[i]
      const value = args[3] || args[4] || args[5] || ''
      const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
        ? options.shouldDecodeNewlinesForHref
        : options.shouldDecodeNewlines
      attrs[i] = {
        name: args[1],
        value: decodeAttr(value, shouldDecodeNewlines)
      }
      if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        attrs[i].start = args.start + args[0].match(/^\s*/).length
        attrs[i].end = args.end
      }
    }
    //如何当前标签没有闭合标志，则存储到stack数组中，
    if (!unary) {
      stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end })
      lastTag = tagName
    }
    //调用start函数进一步处理
    if (options.start) {
      options.start(tagName, attrs, unary, match.start, match.end)
    }
  }
```

       