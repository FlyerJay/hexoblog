---
title: Vue源码解析--如何实现数据变化的监听
date: 2017-06-02 16:18:58
tags: 'javascript'
---
### 前言

&#8195;&#8195;Vue对象属性data被定义为一个返回值必须为Object的函数，data初始化之后就不可以随意的增删属性。Vue中所有的可以用来驱动视图的属性被设计为响应式的（reactive），这些属性会被挂载一个Observer对象，Observer对象拥有强大的观察能力，reactive数据一旦发生变化就会被他捕捉然后通知视图应该做出怎样的响应。data便是最外层的被挂载了Observer的数据（也就是reactive化），如果data中有Object或者Array属性，它们也会被reactive化，以此类推。我们都知道要想实现数据变化的监听一般都会使用到设计模式中的订阅发布模式（观察者模式），Vue中的数据变化监听也是这样的一个模式，下面就一起来看看源码吧。

### Dep对象
&#8195;&#8195;在Vue的订阅发布模式中，首先进入我们视野的是Dep对象（depend：依赖），它的角色应该更接近于消息中心。Dep实例拥有一个全局唯一的标志（id）和一个订阅栈（subs）。
```js
    //Dep实例可以拥有多条不的同订阅
    //他维护一个订阅栈subs
    var uid$1 = 0;
    var Dep = function Dep () {
        this.id = uid$1++;
        this.subs = [];
    };
```
&#8195;&#8195;addSub、removeSub方法用来管理subs栈
```js
    //添加订阅
    Dep.prototype.addSub = function addSub (sub) {
        this.subs.push(sub);
    };

    /*
     *   remove$1(arr, item){
     *       if(arr.length){
     *           var index = arr.indexOf(item);
     *           if(index > -1){
     *               return arr.splice(index,1);
     *           }
     *       }
     *   }
     */
    //移除订阅
    Dep.prototype.removeSub = function removeSub (sub) {
        remove$1(this.subs, sub);
    };
```
&#8195;&#8195;depend方法中出现了一个target对象，根据上下文我发现它是一个watcher实例，watcher里也维护了一个类似subs的栈来管理dep（暂且叫他deps吧），调用这个方法的dep会被放入watcher的deps栈中。现在我们先不关心watcher到底是什么鬼，继续往下看。（看名字应该能猜得出来吧）
```js
    //添加追踪
    Dep.prototype.depend = function depend () {
        if (Dep.target) {
            Dep.target.addDep(this);
        }
    };
```
&#8195;&#8195;再来看看target在Dep上的定义，它是Dep的类属性，并且通过一个全局的targetStack对其进行管理，很明显通过Dep访问target在任何时候都是一个唯一确定的。
```js
    //这里的target是一个watcher实例
    Dep.target = null;
    var targetStack = [];

    function pushTarget (_target) {
        if (Dep.target) { targetStack.push(Dep.target); }
        Dep.target = _target;
    }

    function popTarget () {
        Dep.target = targetStack.pop();
    }
```
&#8195;&#8195;notify方法用来通知subs栈中的每一个订阅者数据变更
```js
    Dep.prototype.notify = function notify () {
        //slice方法是最简单的数组拷贝（记住js中对象赋值是引用赋值）
        var subs = this.subs.slice();
        for (var i = 0, l = subs.length; i < l; i++) {
            //每一个subs元素都是一个对象，所以subs[i] === this.subs[i]
            //为什么不直接用this.subs[i].update()
            //stablize the subscriber list first（这是作者给的注释）
            subs[i].update();
        }
    };
```
### Observer对象
&#8195;&#8195;Observer对象扮演观察者的角色，是这个模式中绝对的主角。创建Observer对象需要给一个value参数，value即是需要reactive化的数据。value被Observer包装之后会增加一个属性__ob__，它指向Observer本身（有点像隐式原型）。一个Observer对象包含3个属性：value自身的引用，dep实例，以及一个看起来好像是计数器的vmCount。value的值好像是没有什么要求的，如果value是一个数组那么就会执行一系列数组reactive化的方法。（这个稍后再来看，继续往下）
```js
    //把需要ob的data封装成一个Observer对象
    var Observer = function Observer (value) {
        this.value = value;
        //ob.value可以访问到他的值
        this.dep = new Dep();
        this.vmCount = 0;
        //当有根元素的时候才会给vmCount++
        def(value, '__ob__', this);
        //value.__ob__ = ob类似于__proto__的设计
        //数组和其他类型的处理方式不同
        if (Array.isArray(value)) {
            //var hasProto = '__proto__' in {};
            //由于__proto__并非标准属性，所以需要判断其是否可用
            //__proto__指向隐式原型[[prototype]]
            var augment = hasProto
            ? protoAugment
            : copyAugment;
            //把value的数组方法替换成之前定义好的那些响应式方法
            augment(value, arrayMethods, arrayKeys);
            this.observeArray(value);
        } else {
            this.walk(value);
        }
    };
```
&#8195;&#8195;如果value不是数组就会执行下面这个walk方法，这里其实value就是被当成了object处理，如果不是，那么什么也不会发生，value中所有的属性都会被defineReactive$$1方法处理。
```js
    Observer.prototype.walk = function walk (obj) {
        var keys = Object.keys(obj);
        for (var i = 0; i < keys.length; i++) {
            //把obj的普通属性转换成reactive属性（ob对象）
            defineReactive$$1(obj, keys[i], obj[keys[i]]);
        }
    };
```
&#8195;&#8195;defineReactive$$1接收四个参数，这里只用到了前三个，第一个参数value，后面两个分别是value属性的键和值。具体来看这个方法，第一行怎么又特么new了一个dep，不同于Ob对象具有dep属性，像number，string这种基本属性由于不是对象所以挂载不了Ob，而这里的dep就是用来追踪基本属性的。实际上执行defineReactive$$1相当于创建了一个闭包，利用闭包的特性，每一个基本属性都能访问到一个唯一的dep。
&#8195;&#8195;defineReactive$$1会尝试获取属性的set，get方法，一般得属性都是没有这两个方法的，除非有特别定义过，如果有get，set方法后面定义新的set，get会用到。
&#8195;&#8195;var childOb = observe(val);又出现了一个方法observe，它的作用是reactive化传入的参数。当val是一个对象时（数组也是对象），val也会被reactive化并把挂载的ob返回回来。
&#8195;&#8195;再然后就是给val属性设置get和set方法（如果你还不知道get，set方法的作用那就可以回去重修了），好像忽略了enumerable和configurable，这两个属性设置为true之后val就是可枚举可修改的了（默认的属性都是可枚举可修改的）。看看get方法的设置，每次获取val的值都会调用dep.depend往watcher中添加追踪，这里就有个疑问，每次都添加会不会重复添加很多dep，想想应该是不会的，前面有提到dep会有一个全局唯一的标志，添加的时候应该会判断一下（这里总感觉怪怪的，每次都添加会不会影响效率呢？），如果val是一个对象那就把reactive化之后的dep添加到watcher中。set方法中会调用dep.notify也就是我们想要的当属性发生变化时发出通知，dep中包含的订阅都会收到这个通知。如果set的属性是一个对象也会调用observe方法对它进行reactive化。
```js
        function defineReactive$$1 (
        obj,
        key,
        val,
        customSetter
    ) {
        
        var dep = new Dep();

        var property = Object.getOwnPropertyDescriptor(obj, key);
        if (property && property.configurable === false) {
            return
        }

        //没有特别定义过的对象都是没有默认的get和set方法的
        var getter = property && property.get;
        var setter = property && property.set;

        //如果val是一个对象就会被包装成ob对象
        var childOb = observe(val);
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: function reactiveGetter () {
                var value = getter ? getter.call(obj) : val;
                //当Dep.target存在时把所有的依赖收集起来,调用watcher的addDep方法
                //前面有提到target是一个watcher对象
                if (Dep.target) {
                    dep.depend();
                    if (childOb) {
                        childOb.dep.depend();
                    }
                    if (Array.isArray(value)) {
                        dependArray(value);
                    }
                }
                return value
            },
            set: function reactiveSetter (newVal) {
                var value = getter ? getter.call(obj) : val;
                //如果新的值和之前的值一样的话就不继续往下执行
                if (newVal === value || (newVal !== newVal && value !== value)) {
                    return
                }
                if ("development" !== 'production' && customSetter) {
                    customSetter();
                }
                //如果有set方法调用set方法
                if (setter) {
                    setter.call(obj, newVal);
                } else {
                    val = newVal;
                }
                //设置的新值如果是个对象的话就调用observe把他变成ob对象
                childOb = observe(newVal);
                //通知变更
                dep.notify();
            }
        });
    }
```
&#8195;&#8195;从get、set方法中我们大致可以看出一些vue进行对象reactive化的规律，我们不能通过value.data = data来设置一个新的属性，因为这样不会触发set的reactive化，取而代之的是使用value = obj（新的对象），当然在value对象下的属性有声明的时候可以直接通过value.data = data来赋值。但是我们发现对于数组好像不是那么回事，数组元素中增加一个值并不会触发set方法也就不会收到通知，我们不可能每次都给数组全量替换进行赋值。
```js
    function observe (value, asRootData) {
        if (!isObject(value)) {
            return
        }
        var ob;
        if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
            ob = value.__ob__;
        } else if (
            observerState.shouldConvert &&
            !isServerRendering() &&
            (Array.isArray(value) || isPlainObject(value)) &&
            Object.isExtensible(value) &&
            !value._isVue
        ) {
            ob = new Observer(value);
        }
        if (asRootData && ob) {
            ob.vmCount++;
        }
        return ob
    }
```
### 数组
&#8195;&#8195;对于数组的reactive化会比对象的稍微复杂一些，因为数组元素的增加删除不会触发set方法。Vue中采用了封装数组方法的方式来捕捉数组元素的变化，首先收集到所有会对数组产生影响的方法，除了 [] = [1,2]（这种方法会触发set）,push、pop、shift、unshift、splice、sort、reverse。数组方法还有很多，例如：concat、slice、map等，但是这些方法不会在数组上直接进行操作，也不会对数组产生副作用。
&#8195;&#8195;通过重写数组方法，在其中插入钩子，每次执行这些数组方法的时候都调用notify进行更新通知，且在有新数组插入是调用observeArray方法进行reactive化。
```js
    //arrayMethods继承自Array对象
    var arrayProto = Array.prototype;
    var arrayMethods = Object.create(arrayProto);
    [
        'push',
        'pop',
        'shift',
        'unshift',
        'splice',
        'sort',
        'reverse'
    ].forEach(function (method) {
        var original = arrayProto[method];
        /*
         *  function def (obj, key, val, enumerable) {
         *     Object.defineProperty(obj, key, {
         *          value: val,
         *          enumerable: !!enumerable,
         *          writable: true,
         *          configurable: true
         *      });
         *  }
         */
        def(arrayMethods, method, function mutator () {
            //把arguments赋值给arguments$1是为了提高速度
            //作者给了原文链接http://jsperf.com/closure-with-arguments
            //这里可以看到测试结果（长姿势）
            var arguments$1 = arguments;
            var i = arguments.length;
            var args = new Array(i);
            while (i--) {
                args[i] = arguments$1[i];
            }
            //这句代码应该是为了保证重新定义过的方法保持不变
            var result = original.apply(this, args);
            var ob = this.__ob__;
            var inserted;
            //push，unshift，splice这三个方法会往数组中添加值
            switch (method) {
                case 'push':
                    inserted = args;
                    break
                case 'unshift':
                    inserted = args;
                    break
                case 'splice':
                    inserted = args.slice(2);
                    break
            }
            if (inserted) { ob.observeArray(inserted); }
            ob.dep.notify();
            return result
        });
    });
```
```js
    var arrayKeys = Object.getOwnPropertyNames(arrayMethods);
    
    var observerState = {
        shouldConvert: true,
        isSettingProps: false
    };

    Observer.prototype.observeArray = function observeArray (items) {
        for (var i = 0, l = items.length; i < l; i++) {
            observe(items[i]);
        }
    };

    function protoAugment (target, src) {
        target.__proto__ = src;
    }

    function copyAugment (target, src, keys) {
        for (var i = 0, l = keys.length; i < l; i++) {
            var key = keys[i];
            def(target, key, src[key]);
        }
    }

    function dependArray (value) {
        //e = (void 0) === e = undifined
        for (var e = (void 0), i = 0, l = value.length; i < l; i++) {
            e = value[i];
            e && e.__ob__ && e.__ob__.dep.depend();
            if (Array.isArray(e)) {
                dependArray(e);
            }
        }
    }
```