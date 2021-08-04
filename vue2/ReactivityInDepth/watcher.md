## watcher类分析

- 渲染Watcher：变量修改时，负责通知HTML里的重新渲染
- computed Watcher：变量修改时，负责通知computed里依赖此变量的computed属性变量的修改
- user Watcher：变量修改时，负责通知watch属性里所对应的变量函数的执行

## computed Watcher

- 如果有computed属性,那么initComputed
```js
function initComputed(vm, computed) {
  const watchers = vm._computedWatchers = Object.create(null)

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get

    // create internal watcher for the computed property.
    watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions: { lazy: true }
    )

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
  }
}
```
做的事情: 1. 创建computed 的watcher 2. 将computed里的属性代理到vm上(this.computedProperties)

- defineComputed内部的事情
```js
function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }

  Object.defineProperty(target, key, sharedPropertyDefinition)
}

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
做的事情: 将computed里的属性代理到vm上,将get属性重写为computedGetter,其中当获取computed的时候,会先执行watcher.evaluate(), 然后有依赖的时候, 会执行watcher.depend()

- evaluate函数
```js
 /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
    Watcher.prototype.evaluate = function evaluate () {
        this.value = this.get();
        this.dirty = false;
    };

    Watcher.prototype.get = function get () {
        pushTarget(this);
        var value;
        var vm = this.vm;
        try {
            value = this.getter.call(vm, vm);
        } catch (e) {
            if (this.user) {
                handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
            } else {
                throw e
            }
        } finally {
            // "touch" every property so they are all tracked as
            // dependencies for deep watching
            if (this.deep) {
                traverse(value);
            }
            popTarget();
            this.cleanupDeps();
        }
        return value
    };
```

执行流程: this.get()函数 => this.getter()函数;this.getter()函数是去执行computed里的函数或者get方法,如果get方法里有data中的属性,此时函数会进入defineReactive$$1

- defineReactive$$1
```js
function defineReactive$$1 (
    obj,
    key,
    val,
    customSetter,
    shallow
) {
    var dep = new Dep();

    var property = Object.getOwnPropertyDescriptor(obj, key);
    if (property && property.configurable === false) {
      return
    }

    // cater for pre-defined getter/setters
    var getter = property && property.get;
    var setter = property && property.set;
    if ((!getter || setter) && arguments.length === 2) {
      val = obj[key];
    }

    var childOb = !shallow && observe(val);
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
            var value = getter ? getter.call(obj) : val;
            if (Dep.target) {
                dep.depend();
                if (childOb) {
                    childOb.dep.depend();
                    if (Array.isArray(value)) {
                        dependArray(value);
                    }
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
            if (customSetter) {
                customSetter();
            }
            // #7981: for accessor properties without setter
            if (getter && !setter) { return }
            if (setter) {
                setter.call(obj, newVal);
            } else {
                val = newVal;
            }
            childOb = !shallow && observe(newVal);
            dep.notify();
        }
    });
}
```

进入get方法中会执行dep.depend()
```js
Dep.prototype.depend = function depend () {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
};
```
此时Dep.target就是这个computed watcher再调用addDep方法
```js
Watcher.prototype.addDep = function addDep (dep) {
    var id = dep.id;
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id);
      this.newDeps.push(dep);
      if (!this.depIds.has(id)) {
        dep.addSub(this);
      }
    }
};
```
这个dep是data属性里的Dep,这样就把data的属性上的Dep管家增加了computed Watcher

### evaluate函数总结

+ 1. 会去获取computed属性里的值,这样会触发computed里的方法或get属性,如果这个方法依赖data里的属性
+ 2. 会触发data属性的get方法, dep.depend()这个方法进而触发watcher.addDep()方法; watcher会管理这个Dep,data的属性会把computed Watcher添加到自己的依赖

+ 3. 执行computed watcher的get函数获取值,将dirty属性改为false,到此evaluate方法执行完

- watcher.depend()函数
```js
Watcher.prototype.depend = function depend () {
    var i = this.deps.length;
    while (i--) {
      this.deps[i].depend();
    }
};
```
## user Watcher

这里只说newVal 与 oldVal 的处理,如果你是user Watcher
```js
run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          const info = `callback for watcher "${this.expression}"`
          invokeWithErrorHandling(this.cb, this.vm, [value, oldValue], this.vm, info)
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }
```
意思是执行this.get()获取当前的值,将老值与新值一起传给callback函数