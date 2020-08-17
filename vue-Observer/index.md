## Vue源码剖析——vue给数据添加拦截器（以data选项为例）

### 什么是数据响应化
在vue中核心思想中有一点至关重要，我们在开发中也感觉特别好使，就是更改数据自动驱动对应的UI进行改变，这点我们称之为Vue数据响应化，
vue2.0+版本都是通过defineProperty定义getter和setter进行数据拦截，getter进行收集事件，setter用于触发事件。
（vue3.0+中改用了Proxy进行数据拦截，我们这里先只介绍2.0+版本的数据响应）。

vue中用defineProperty对getter、setter进行重写，用于数据依赖收集和触发，我们先简单看看这个Object上的defineProperty方法，
该方法更多的特性行自行前往MDN查看[defineProperty](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

```
let obj={name:'hepeng',age:'20'};

let keys=Object.keys(obj);
for(let i=0;i<keys.length;i++){
    let value=obj[key[i]];
    Object.defineProperty(obj,keys[i],{
        get:function(){
            console.log('obj getter')
            
            return value
        },

        set:function(newVal){
            console.log('obj setter');

            value=newVal;
        }
    })
}

```
上面的栗子看上去非常的简单，我们并没有做过多的处理，只是给当前对象添加了getter和setter，下面我们看看Vue是如何做。
Vue中在初始化_init原型方法方法中调用initState对传进来的选项props、methods、data进行数据初始化，我们这里主要介绍data
的数据初始化，data的数据初始化分为获取data、添加代理、添加拦截器。

```
function initData (vm: Component) {
    let data = vm.$options.data
    //获取data
    data = vm._data = typeof data === 'function'? getData(data, vm): data || {}
    const keys = Object.keys(data)
    const props = vm.$options.props
    const methods = vm.$options.methods
    let i = keys.length
    while (i--) {
        const key = keys[i]
        //检测methods中定义的方法名是否是否与data中用重复
        if (methods && hasOwn(methods, key)) {
            warn(
            `Method "${key}" has already been defined as a data property.`,
            vm
            )
        }
        //检测props定义的属性名是否与data中有重复
        if (props && hasOwn(props, key)) {
            process.env.NODE_ENV !== 'production' && warn(
            `The data property "${key}" is already declared as a prop. ` +
            `Use prop default value instead.`,
            vm
            )
        } else if (!isReserved(key)) {//因为vue的内部属性大多是以_开头，在这判断一下，
            //代理data
            proxy(vm, `_data`, key)
        }
    }
    //添加观察器
    observe(data, true /* asRootData */)
}  
```


### 获取data，
Vue组件在定义data选项时，我们通常都定义为在函数中返回一个json对象，解决组件在多次调用中访问的data都指向同一地址。

```
getData (data: Function, vm: Component): any {
    //禁用Dep的收集
    pushTarget()
    try {
        return data.call(vm, vm)
    } catch (e) {
        handleError(e, vm, `data()`)
        return {}
    } finally {
        popTarget()
    }
}

```
执行data.call(vm, vm)，获取data，并将其赋值到vm._data上。执行时将函数中的this替换成当前组件数量，并传入一个参数（当前实例））。


### 将data进行代理
调用proxy函数对vm._data上的属性进行代理到vm(当前组件实例上)，代理操作用的也是defineProperty方法，这样我们在组件中可以直接操作对应的属性，
通过getter和setter的数据劫持映射到vm._data中对应的属性上。

```
function proxy (target: Object, sourceKey: string, key: string) {
    sharedPropertyDefinition.get = function proxyGetter () {
        return this[sourceKey][key]
    }
    sharedPropertyDefinition.set = function proxySetter (val) {
        this[sourceKey][key] = val
    }
    Object.defineProperty(target, key, sharedPropertyDefinition)
}

```
这里需要明确，我们只代理data中最外层的属性，既Object.keys(vm._data)获取到的keys数组，不根据对象的深度进行深层次代理，假如存在一个
这样的数据about:{job:'weber'},我们只给about进行了代理，当我们去设置vm.about.job='pm'时,我们会先触发about属性的代理getter,从而继续触发vm._data
中about的getter，返回对应的val，赋值在触发vm._data.about.job的setter


### 给data添加拦截器

* 调用observe。

    在该函数中我们判断当前value(需要添加拦截器getter/setter的data)是否存在__ob__这个属性继而确定当前value是否需要进行Observer实例化，
    并返回这个实例。已经Observer实例化的对象都带有__ob__属性。
  
    ```
    function observe (value: any, asRootData: ?boolean): Observer | void {
        //判断是否对象  或者 是Vnode的实例
        if (!isObject(value) || value instanceof VNode) {
            return
        }
        let ob: Observer | void
        //判断当前是否拥有__ob__属性，并是Observer的实例
        if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
            ob = value.__ob__
        } else if (shouldObserve && !isServerRendering() && (Array.isArray(value) || isPlainObject(value)) && 
            Object.isExtensible(value) && !value._isVue) {
            ob = new Observer(value)
        }
        if (asRootData && ob) {
            ob.vmCount++
        }
        return ob
    }
    ```
    Observe的构造函数,实例化Dep,添加__ob__属性到实例上并指向这个实例,通过判断当前对象是否数组调用不同的方法进行数据拦截
    ```
    export class Observer {
        constructor (value: any) {
            this.value = value
            //实例化Dep
            this.dep = new Dep()
            this.vmCount = 0
            //给这个属性添加__ob__属性，指向这个Observer实例
            def(value, '__ob__', this)
            if (Array.isArray(value)) {
                if (hasProto) {
                    protoAugment(value, arrayMethods)
                } else {
                    copyAugment(value, arrayMethods, arrayKeys)
                }
                this.observeArray(value)
            } else {
                this.walk(value)
            }
        }
        //当是json时添加拦截器 
        walk (obj: Object) {
            const keys = Object.keys(obj)
            for (let i = 0; i < keys.length; i++) {
                defineReactive(obj, keys[i])
            }
        }
        //当是数据时添加拦截器
        observeArray (items: Array<any>) {
            for (let i = 0, l = items.length; i < l; i++) {
                observe(items[i])
            }
        }
    }
    ```
* json对象添加拦截器

    ```
    export function defineReactive ( obj: Object,key: string,val: any,customSetter?: ?Function,shallow?: boolean) {
        const dep = new Dep()
        
        let childOb = !shallow && observe(val);
        
        Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
        const value = getter ? getter.call(obj) : val;
        if (Dep.target) {
        dep.depend()
        if (childOb) {
        childOb.dep.depend()
        if (Array.isArray(value)) {
        dependArray(value)
        }
        }
        }
        return value
        },
        set: function reactiveSetter (newVal) {
        const value = getter ? getter.call(obj) : val
        /* eslint-disable no-self-compare */
        if (newVal === value || (newVal !== newVal && value !== value)) {
        return
        }
        
        val = newVal
        
        childOb = !shallow && observe(newVal)
        dep.notify()
        }
        })
     }
    ```
 
  