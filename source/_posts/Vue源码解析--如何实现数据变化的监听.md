---
title: Vue源码解析--如何实现数据变化的监听
date: 2017-06-02 16:18:58
tags: 'javascript'
---
```js
    //Dep在观察者模式中扮演着消息中心的角色，他维护一个订阅栈subs;
    var Dep = function Dep () {
        this.id = uid$1++;
        this.subs = [];
    };

    Dep.prototype.addSub = function addSub (sub) {
        this.subs.push(sub);
    };

    Dep.prototype.removeSub = function removeSub (sub) {
        remove$1(this.subs, sub);
    };

    Dep.prototype.depend = function depend () {
        if (Dep.target) {
            Dep.target.addDep(this);
        }
    };

    Dep.prototype.notify = function notify () {
        var subs = this.subs.slice();
        for (var i = 0, l = subs.length; i < l; i++) {
            subs[i].update();
        }
    };

    Dep.target = null;
    var targetStack = [];

    function pushTarget (_target) {
        if (Dep.target) { targetStack.push(Dep.target); }
        Dep.target = _target;
    }

    function popTarget () {
        Dep.target = targetStack.pop();
    }

    //arrayMethods继承自Array对象
    var arrayProto = Array.prototype;
    var arrayMethods = Object.create(arrayProto);
```
```js
    [
        'push',
        'pop',
        'shift',
        'unshift',
        'splice',
        'sort',
        'reverse'
    ].forEach(function (method) {
        // cache original method
        var original = arrayProto[method];
        def(arrayMethods, method, function mutator () {
            var arguments$1 = arguments;

            // avoid leaking arguments:
            // http://jsperf.com/closure-with-arguments
            var i = arguments.length;
            var args = new Array(i);
            while (i--) {
                args[i] = arguments$1[i];
            }
            var result = original.apply(this, args);
            var ob = this.__ob__;
            var inserted;
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
###### 这里得返回去看下def方法
```js
    function def (obj, key, val, enumerable) {
        Object.defineProperty(obj, key, {
            value: val,
            enumerable: !!enumerable,
            writable: true,
            configurable: true
        });
    }
```
```js
    var arrayKeys = Object.getOwnPropertyNames(arrayMethods);
    var observerState = {
        shouldConvert: true,
        isSettingProps: false
    };

    var Observer = function Observer (value) {
        this.value = value;
        this.dep = new Dep();
        this.vmCount = 0;
        def(value, '__ob__', this);
        if (Array.isArray(value)) {
            var augment = hasProto
            ? protoAugment
            : copyAugment;
            augment(value, arrayMethods, arrayKeys);
            this.observeArray(value);
        } else {
            this.walk(value);
        }
    };

    Observer.prototype.walk = function walk (obj) {
        var keys = Object.keys(obj);
        for (var i = 0; i < keys.length; i++) {
            defineReactive$$1(obj, keys[i], obj[keys[i]]);
        }
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

        // cater for pre-defined getter/setters
        var getter = property && property.get;
        var setter = property && property.set;

        var childOb = observe(val);
        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get: function reactiveGetter () {
                var value = getter ? getter.call(obj) : val;
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
                /* eslint-disable no-self-compare */
                if (newVal === value || (newVal !== newVal && value !== value)) {
                    return
                }
                /* eslint-enable no-self-compare */
                if ("development" !== 'production' && customSetter) {
                    customSetter();
                }
                if (setter) {
                    setter.call(obj, newVal);
                } else {
                    val = newVal;
                }
                childOb = observe(newVal);
                dep.notify();
            }
        });
    }

    function set$1 (obj, key, val) {
        if (Array.isArray(obj)) {
            obj.length = Math.max(obj.length, key);
            obj.splice(key, 1, val);
            return val
        }
        if (hasOwn(obj, key)) {
            obj[key] = val;
            return
        }
        var ob = obj.__ob__;
        if (obj._isVue || (ob && ob.vmCount)) {
            "development" !== 'production' && warn(
            'Avoid adding reactive properties to a Vue instance or its root $data ' +
            'at runtime - declare it upfront in the data option.'
            );
            return
        }
        if (!ob) {
            obj[key] = val;
            return
        }
        defineReactive$$1(ob.value, key, val);
        ob.dep.notify();
        return val
    }

    function del (obj, key) {
        var ob = obj.__ob__;
        if (obj._isVue || (ob && ob.vmCount)) {
            "development" !== 'production' && warn(
            'Avoid deleting properties on a Vue instance or its root $data ' +
            '- just set it to null.'
            );
            return
        }
        if (!hasOwn(obj, key)) {
            return
        }
        delete obj[key];
        if (!ob) {
            return
        }
        ob.dep.notify();
    }

    function dependArray (value) {
        for (var e = (void 0), i = 0, l = value.length; i < l; i++) {
            e = value[i];
            e && e.__ob__ && e.__ob__.dep.depend();
            if (Array.isArray(e)) {
                dependArray(e);
            }
        }
    }
```