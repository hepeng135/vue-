## Vue源码剖析——vue中数据变化侦测

Vue.js最独特的特性之一就是响应式系统，数据模型仅仅是普通的javascript对象，当修改他们的时候，视图就会进行更新。下面我就
来看下vue是怎么实现这个功能的。

对于上面的响应式系统我们需要先讨论几个问题，首先数据模型进行改变的时候，我们怎么去捕获到这个状态，当状态可以捕获后我们如何知道去更新谁，更新哪个DOM。

### Vue中Object数据如何追踪变化

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
        setter:function(newVal){
            if(newVal===val) return
            val=newVal;
            
        }
    })
}

//创建一个Dep对象
Dep.target=null;
class Dep {
    constructor(){
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
    //触发执行依赖
    notify(){
        const subs=this.subs.slice();
        for(let i=0;i<subs.length;i++){
            subs[i].update()
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
        this.getter=expOrFn

        this.cb=cb;

        this.get();
    }

    get(){
        Dep.target=this;
        let value=this.getter.call(this,vm);
        Dep.target=null;
        return value
    }

    update(){
        //更新队列
        queueWatcher(this)
    }
}
```
上述代码并没有考虑用户自己创建watcher时的情况，只实现了，触发getter去收集，然后数据属性更新后调用update进行更新。
当模板编辑器生成render函数后，我们就需要去实例化watcher，代码如下

```
function mountComponent(vm,el,hydrating){
    let updateComponent = () => {
        vm._update(vm._render(), hydrating)
    }
    new Watcher(vm, updateComponent)
}
```

上述的代码中updateComponent用于创建或者更新Vnode并转换为真正的DOM元素。当实例化Watcher时，我们会调用get，首先给Dep.target赋值为当前
watcher实例，然后调用updateComponent，在生成vnode的过程中需要访问数据属性，这样就触发了getter，然后调用dep.depend去收集当前这个数据属性的
依赖。上面收集依赖的代码我们已经非常简化了，并没有去考虑很多问题，例如在收集的过程中，一个数据属性可能收集到很多相同的依赖watcher。想了解Vue完整
源码可以点击这些传送门 [Vue中Dep源码](./Dep.md)，Vue中Watcher源码。

现在我们实现变化侦测的功能，但是我们只限于数据中的一个属性，对于复杂的Object还是很无力，这个时候我们就需要循环递归，侦测Object上所有的属性，我们
新建一个Observer类
```
class Observer {
    constructor(value){
        this.value=value;
        if(!Array.isArray(value)){
            this.walk(value)
        }
    }

    walk(obj){
        let keys=Object.keys(obj);
        for(let i=0;i<keys.length;i++){
            defineReactive(obj,keys[i],obj[keys[i]])
        }
    }
}
defineReactive中需要增加的代码
function defineReactive(obj,key,val){
    if(typeof val ==='object'){
        new Observer(val)
    }
    ....code
}

```

上面是对Object类型数据的侦测，那么假如是数组的话我们要怎么办呢，怎么知道数据里元素的变化，已经当前数组的变化。

### Vue中Array数据如何追踪变化



#### 数组的变化侦测
关于数组的变化侦测，我们可以通过拦截数据的原型方法进行实现，在数组的原型方法中能改变数组自身内容的方法有7个，pop,push,shift,unshift,splice,sort,reverse,

```
let arrayProto=Array.prototype;
let arrMethods=Object.create(arrayProto);

[pop,push,shift,unshift,splice,sort,reverse].forEach(function(methodName){
    let original=arrayProto[methodName];
    Object.definePrototype(arrMethods,methodName,{
        enumerable:true,
        configurable:true,
        value:function(...args){
            //在这里我们可以侦测到当前数组发生了变化，进而通知依赖。
            original.apply(this,args);
        }
    })
})

```
数组的拦截我们已经做好了，现在需要考虑我们在哪进行拦截，如何触发依赖，最关键的一点是，我们如何收集依赖，先说说收集依赖这个事，比如我们需要访问
一个数组，那么首先我们需要访问数组对应的属性key，那么这个时候我们就触发了getter，那就可以在这进行收集，在这里我们需要能够访问到数组对应的Dep，对于
触发我们需要在拦截方法中访问对应的Dep，怎么访问呢，由于这两个地方都可以访问到数组，那么我们可以把Dep实例挂载到value上作为一个属性，代码如下

```
function observe(value){
    if(typeof value !=='object') return
    let ob;
    if(Object.prototype.hasOwnProperty.call(value,'__ob__') && value.__ob__.instanceof Observer){
        ob=value.__ob__;
    }else{
        ob=new Observer(value);
    }
    return ob
}

class Observer{
    constructor(value){
        this.value=value;
        this.dep=new Dep();
        Object.defineProperty(value,'__ob__',{
            enumerable:true,
            configurable:true,
            value:this
        });
        
        if(Array.isArray(value)){
            value.__proto__=arrMethods;
            this.observerArray(value)
        }else if(Object.prototype.toString.call(value)==='Object'){
            this.walk(value)
        }
    }
    
    walk(value){
        let keys=Object.keys(obj);
        for(let i=0;i<keys.length;i++){
            defineReactive(obj,keys[i],obj[keys[i]])
        }
    }
    
    observeArray(value){
        for(let i=0;i<value.length;i++){
            observe(value[i]);
        }
    }
}
function defineReactive(obj,key,val){
    let dep=new Dep();
    let childOb=observe(val);//获取对应数组的dep
    Object.defineProperty(obj,key,{
        enumerable:true,
        configurable:true,
        value:{
            get:function(){
                dep.depend();
                if(childOb){
                    childOb.dep.depend();
                }
                return val
            },

            set:function(newVal){
                if(newVal===val) return
                val=newVal;
                dep.notify();
            }
        }
    })
}
```
上述代码，我们增加了一个observe函数，用来检测当前value是否拥有__ob__属性，没有的话就调用Observer进行实例化并添加__ob__属性，我们在Observer中构造函数
中添加__ob__属性指向当前这个实例，从而可以访问到对应的dep，当访问数据 {list:[1,2,3,4]}时，我们属性访问list这个属性key，触发对应的getter，我们通过childOb可以访问
到这个数组value对应的dep，进而进行依赖收集。

同时上述代码也增加了给目标数组添加拦截，因为我们不能污染全局的数组方法，我们对需要添加拦截的数组进行添加arrMethods方法,挂载再__proto__上。

当数组里面的值都是主数据类型(Number,Boolean,String等)时，我们进行修改只能用splice方法进行（其他操作用上面对应的7种方法），这样才能触发依赖，不能直接
arr[index]=xx 这样进行修改，如果数组里面对象时，这个时候我们需要对对象中某个值进行修改要怎么办了，直接访问修改即可，eg：list:[{name:'hepeng'}] 访问：
list[0][name]='hepeng1',上述代码中，我们并没有对这种情况进行收集依赖，下面代码将解决这个问题，我们只需要在get中添加代码即可

```
function defineProperty(obj,key,val){
    let dep=new Dep();
    let childOb=observe(val);
    Object.defineProperty(obj,key,{
        enumerable:true,
        configurable:true,
        value:{
            get:function(){
                dep.depend();
                if(childOb){
                    childOb.dep.depend();
                    if(Array.isArray(val)){
                        dependArray(val)
                    }
                }
                return val
            }
        }
    })
}
function dependArray(val){
    for(let i=0;i<val.length;i++){
        let e=val[i];
        e && e.__ob__ && e.__ob__.depend();
        if(Array.isArray(e)){
            dependArray(e);
        }
    }
}
```
上述代码解决了数组中是Object类型数据时无法收集依赖的问题，我们新建一个dependArray函数，通过循环访问每一项属性__ob__，然后调用depend()进行收集依赖
，如果这一项数数组，将递归调用dependArray。同时该函数的第一次调用将放再getter中，如果当前属性对应的value是引用类型并且有childOb时并且是数组时则调用
dependArray。

上述所有的代码中，已经对数组类型的数据进行了依赖收集，那么我们怎么进行触发呢，当value为object类型时，我们设置新值的时候直接在setter里面去触发，当value为数组时
我们需要在拦截的方法中进行触发 ，代码如下
```
const arrayProto=Array.prototype;
let arrMethods=Object.create(arrayProto);
['pop','push','shift','unshift','sort','reverse','splice'].forEach(function(methodName){
    let original=arrayProto[methodName]
    Object.defineProperty(arrMethods,methodName,{
        enumerable:true,
        configurable:true,
        value:function(...args){
            let dep=this.__ob__.dep;
            original.apply(this,args);
            dep.notify();
        }
    })
})

```

现在我们可以通过list.splice(0,0,1)去操作数组，从而调用依赖函数进行视图UI更新，但是假如我们添加的是引用类型数据怎么办，再次修改这个引用类型中的值，依赖
并没有执行，因为这个新添加的数据并没有进行数据变化侦测。数组存在这种情况，object数据也存在这种情况 

```
eg:如下所示，上述代码在对数组或者对象进行添加或者修改，新值将不会进行数据变化侦测，
 data(){
    return {
        list:[1,2,3,4],
        name:'hepeng'
    }
}
this.list.push({name:'he',age:'22'})
this.name={name:'he',uid:'111'}

```
解决上述问题，我们需要在数据进行赋值时，将新数据进行observe，确保也是响应式，我们只需要在setter和数组拦截的方法中去调用observe就好

```
//对于Object
function defineReactive(obj,key,val){
    //code...
    set:function(newVal){
        //code..、
        //新添加的
        childOb=observe(newVal);
        return newVal
    }
    
}
//对于数组
let arrayProto=Array.prototype;
let arrMethods=Object.create(arrayProto);
['pop','push','shift','unshift','sort','reverse','splice'].forEach(function(methodName){
    let original=arrayProto[methodName];
    Object.defineProerty(arrMethods,methodName,{
        enumerable:true,
        configurable:true,
        value:function(...args){
            let result=original.apply(this,args)
            let ob=this.__ob__;
            let inserted;
            switch(methodName){
                case push:
                case unshift:
                    inserted=args;
                    break
                case splice:
                    inserted=args.length>=3 ? args.slice(2) : ''
                    break
            }
            inserted && ob.observeArray(inserted);
            ob.dep.notify();  
            return result
        }
    })   
})


```

通过上述的总结，我们将得到下面的代码

```
let arrayProto=Array.prototype;
let arrMethods=Object.create(arrayProto);
['pop','push','shift','unshfit','sort','reverse','splice'].forEach(function(methodName){
    let original=arrayProto[methodName];
    Object.defineProperty(arrMethods,methodName,{
        enumrable:true,
        configurable:true,
        value:function(...args){
            let result=original.apply(this,args);
            let ob=this.__ob__;
            let inserted;
            switch(methodName){
                case 'push':
                case 'unshift':
                    inserted=args;
                case 'splice' :
                    inserted=args.slice(2);
            }
            inserted && ob.observeArray(inserted);
            ob.dep.notify();
            return result
        }
    })
})


function observe(value){
    if(typeof value !=='object') return;
    let ob;
    if(value.hasOwnProperty('__ob__') && value.__ob__ instanceOf Observer){
        ob=value.ob
    }else{
        ob=new Observer(value)
    }
    return ob;   
}
function defineReactive(obj,key,val){
    let dep=new Dep();
    let childOb=observe(val);

    Object.defineProperty(obj,key,{
        get:function(){
            dep.depend();
            if(childOb){
                childOb.dep.depend();
                if(Array.isArray(val)){
                    dependArray(val);
                }
            }
            retturn val
        },
        set:function(newVal){
            if(newVal===val) return
            val=newVal; 
            childOb=observe(newVal);
            dep.notify();    
        }
    })  
}
class Observer{
    constructor(val){
        this.value=val;
        this.dep=new Dep();
        Object.deinfProperty(val,'__ob__',{
            enumerable:true,
            configurable:true,
            value:this
        });

        if(Array.isArray(val)){
            val.__proto__=arrMethods;
            this.observerArray(val)
        }else{
            this.walk(val)
        }
        
    }
    walk(val){
        let keys=Object.keys(val);
        for(let i=0;i<key.length;i++){
            defineReactive(val,key[i],val[key[i]]);
        }
    }
    observeArray(val){
        for(let i=0;i<val.length;i++){
            observe(val[i]);
        }
    }
}
function dependArray(val){
    for(let i=0;i<val.length;i++){
        let e=val[i];
        e && e.__ob__ && e.__ob__.depend();
        if(Array.isArray(e)){
            dependArray(e);
        }
    }
}
Dep.target=null
class Dep{
    constructor(){
        this.subs=[];
    }
    depend(){
        if(Dep.target){
            this.subs.push(Dep.target)
        }
    },
    notify(){
        let subs=this.subs.slice();
        for(let i=0;i<subs.length;i++){
            subs[i].update()
        }
    }
}
class Watcher{
    constructor(expOrFn,cb,options){
        this.getter=expOrFn;
        this.cb=cb;
    }
    get(){
        Dep.target=this;
        this.gette();
        Dep.target=null
    }
    update(){
        //更新操作
    }
}
```

