## Vue源码剖析——vue中数据变化侦测

Vue.js最独特的特性之一就是响应式系统，数据模型仅仅是普通的javascript对象，当修改他们的时候，视图就会进行更新。下面我就
来看下vue是怎么实现这个功能的。

对于上面的响应式系统我们需要先讨论几个问题，首先数据模型进行改变的时候，我们怎么去捕获到这个状态，当状态可以捕获后我们如何知道去更新谁，更新哪个DOM。

### Vue中如何追踪变化（Object）

在javascript中有两种追踪变化的方式，使用Object.defineProperty和ES6的Proxy，由于ES6在浏览器中的兼容度并不理想，所以Vue2.+
使用的Object.defineProperty来实现，虽然Vue3.+已经用Proxy进行重写了，但是原理是相同的，不必去纠结。知道Object.defineProperty这个
方法，我们就能写出如下代码 

```
function defineReactive(obj,key,val){
    Object.defineProperty(obj,key,{
        enumerable: true,
        configurable: true,
        get:function(){
            return val
        },
        set:function(newVal){
            if(val===newVal) return
            val=newVal
        }
    })
}
```
#### 收集依赖

我们需要如何收集依赖，然后数据的属性变化的时候去通知那些曾经使用过这个数据的地方，在Vue1.0中收集的目标定位到了具体的DOM，
然后属性变化的时候去通知之前使用过这个数据的DOM进行更新，这个颗粒度就非常的细了，假如一个数据属性绑定绑定的依赖越多，
依赖追踪的内存开销就越大，因此在Vue2.+中我们把这个数据属性与DOM的映射关系上升到数据属性与组件，同时引入虚拟DOM，当数据属性发生
变化时，我们去更新组件，在使用虚拟DOM进行对比。

上次我们介绍了模板编译生成Render渲染函数，当调用Render函数进行首次渲染的时候，我们需要去读取属性，这个时候我们就触发了对应属性的
getter，那么我们就可以在getter中去收集依赖。

```
function defineReactive(obj,key,val){
    let dep=new Dep();
    Object.defineProperty(obj,key,{
        enumerable:true,
        configurable:true,
        getter:function(){
            //收集依赖的地方
            dep.depend()
            return val
        },
        setter:function(){}
    })
}

//创建一个Dep对象
Dep.target=null;
class Dep {
    constructor(){
        this.id = uid++
        this.subs = []
    },
    //添加
    addSub(sub){
        this.sub.push(sub)
    }
    //收集依赖
    depend(){
        if(Dep.target){
            this.addSub(Dep.target)   
        }
    }
}
```
我们已经知道在哪进行收集了，在上面的代码中我们收集的依赖是Dep.target,那么依赖是什么？当数据发送变化时，我们去通知谁，在Vue
中就是Watcher，一个组件对应一个Watcher，当然也可以是用户自己创建的watcher。

```
class Watcher{
    constructor(vm,expOrFn,cb){
        this.vm=vm;
        this.getter=Object.prototype.toString.call(expOrFn)==='Function' ? expOrFn : parsePath(expOrFn);

        this.cb=cb;
    }

    get(){
        Dep.target=this;
        let value=this.getter.call(this,vm);
        Dep.target=null;
        return value
    }
}

```
