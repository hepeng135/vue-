parseEndTag函数，解析结束标签，调用end钩子函数

#### 函数作用
    1：用匹配出来的结束标签在stack中找到对应的开始标签的位置pos，成对标签形成。
    2：倒序循环stack数组，pos为限制条件，然后调用end钩子函数
    3：将这个标签从stack中删除
#### 参数列表
    @params tagName: 当前标签名
    @params start:标签开始的index
    @params end：标签结束的index

```
function parseEndTag (tagName, start, end) {
    let pos, lowerCasedTagName
    if (start == null) start = index
    if (end == null) end = index
    
    if (tagName) {
        //统一将标签名转为小写
        lowerCasedTagName = tagName.toLowerCase()
        //从stack数组的尾部开始寻找与该结束标签对应的开始标签，并给出具体位置
        for (pos = stack.length - 1; pos >= 0; pos--) {
            if (stack[pos].lowerCasedTag === lowerCasedTagName) {
                break
            }
        }
    } else {
        // 如果tagName，则默认为0
        pos = 0
    }
    
    if (pos >= 0) {
        //  stack数组尾部开始循环，限制条件为pos，然后调用end钩子函数
        for (let i = stack.length - 1; i >= pos; i--) {
            if (options.end) {
                options.end(stack[i].tag, start, end)
            }
        }
        stack.length = pos//将已经处理完的成对标签进行删除
        lastTag = pos && stack[pos - 1].tag  //更新当前的lastTag
    } else if (lowerCasedTagName === 'br') { //处理br标签  br标签可以这么写 </br>
        if (options.start) { //则直接调用start钩子函数
            options.start(tagName, [], true, start, end)
        }
    } else if (lowerCasedTagName === 'p') {//处理p标签,当前p标签在模板中没有开始标签，eg:dadad</p>,则手动调用start，end
        if (options.start) {
            options.start(tagName, [], false, start, end)
        }
        if (options.end) {
            options.end(tagName, start, end)
        }
    }
}

```