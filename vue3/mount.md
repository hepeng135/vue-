#### mount官方介绍

mount:将应用实例的根组件挂载到提供的DOM元素上

通过这段表述，我们知道这将经历4个生命周期阶段：beforeCreate、created、beforeMount、mounted。同时还会执行Vue3中新增加的组合API：setup钩子函数，下面我们将根据这几个周期钩子函数去了解mount方法的源码实现


#### mount方法扩展
和Vue2一样，在源码中Vue3一样对mount方法进行了扩展，这个扩展是基于当前平台假如是浏览器端时的扩展。

在我们调用createApp创建应用实例后，我们紧接着解构出原有的mount方法，并且创建扩展的mount方法
```
//代码位置：runtime-dom/src/index.md
export const createApp=(...args)=>{
    //创建应用实例
    const app = ensureRenderer().createApp(...args)
    const { mount } = app;
    //扩展mount方法
    app.mount=function(containerOrSelector){
        //code...
    }
}
```

#### 扩展的mount方法详情

```
//伪代码，我们只展示代码中的关键部位
app.mount=function(containerOrSelector){
    //获取当前根组件的渲染DOM
    const container = normalizeContainer(containerOrSelector)
    //如果没有给定元素则不执行，与Vue2完全不一样
    if (!container) return
    //获取当前根组件定义项，
    const component = app._component
    //当前根组件不是函数式组件且不带有render和template选项
    if (!isFunction(component) && !component.render && !component.template) {
        component.template = container.innerHTML
    }
    // clear content before mounting
    //根组件渲染前清除根组件里面的内容
    container.innerHTML = ''
    //调用之前我们缓存下来mount方法
    const proxy = mount(container, false, container instanceof SVGElement)
    if (container instanceof Element) {
        container.removeAttribute('v-cloak')
        container.setAttribute('data-v-app', '')
    }
    return proxy
}

```
扩展的mount方法主要作用就是获取html模板，然后紧接着就会调用之前我们解构出的mount方法，并传入我们获取的模板。

#### 真正的mount方法
这个方法我们在源码解析createApp方法时提到过，我们现在直接看源码

```

```