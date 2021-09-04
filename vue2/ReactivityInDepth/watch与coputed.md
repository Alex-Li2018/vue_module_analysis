# computed与watch的区别

在render函数执行的时候会去获取data里的值这样也会触发get属性,把data里的依赖

watch 添加依赖的模式为: 
```js
watch: {
    user(newVal) {
        console.log(newVal);
    }
}
```

会执行this.getter()这个方法就是去获取当前监控的那个属性,这样会触发这个属性的get,接着就会触发Dep.depend方法

这段代码就实现了依赖会在Watcher实例和Dep实例中相互收集
```js
// Dep.target表示当前的watcher
Dep.prototype.depend = function depend () {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
};

// 接着addDep方法又会反向执行addSub方法,让watcher装上Dep
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
// this.subs里面就会装上wacther
Dep.prototype.addSub = function addSub (sub) {
    this.subs.push(sub);
};
```

在Object.defineProperty中dep是一个闭包的形式,所以不会出现data属性的dep依赖找不到的问题

# computed的逻辑
在initSate函数里进行initComputed

```js
// 第一步生一个computedWatcher(惰性watcher)
if (!isSSR) {
    // create internal watcher for the computed property.
    watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
    );
}

// 定义一个computed
defineComputed(vm, key, userDef);
```

defineComputed函数的目的

```js
Object.defineProperty(vm, key, {
    get() {
        var watcher = this._computedWatchers && this._computedWatchers[key];
        if (watcher) {
            if (watcher.dirty) {
                watcher.evaluate();
            }
            if (Dep.target) {
                watcher.depend();
            }
            return watcher.value
        }
    },
    set() {

    }
})
```

做的事情: 将computed里的属性代理到vm上,将get属性重写为computedGetter,其中当获取computed的时候,会先执行watcher.evaluate(), 然后有依赖的时候, 会执行watcher.depend()

流程如下:

+ 1. 当render function或methods中用到了computed属性会触发computed的get属性
+ 2. 执行watcher.evaluate();目的是求值
```js
Watcher.prototype.evaluate = function evaluate () {
    this.value = this.get();
    this.dirty = false;
};
```
+ 3. 触发this.get()将computed的属性方法执行一次,此时computed依赖的data就会把这个computed Watcher放到自己的依赖队列里
+ 4. watcher.depend()目的是让computed的依赖里添加上render的依赖