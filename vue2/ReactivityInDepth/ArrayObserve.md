## vue对数组的处理

### 首先data初始化

```js
function initState(vm) {

    // 获取vm上的$options对象，也就是options配置对象
    const opts = vm.$options
    if (opts.props) {
        initProps(vm)
    }
    if (opts.methods) {
        initMethods(vm)
    }
    if (opts.data) {
        // 如有有options里有data，则初始化data
        initData(vm)
    }
    if (opts.computed) {
        initComputed(vm)
    }
    if (opts.watch) {
        initWatch(vm)
    }
}

// 初始化data的函数
function initData(vm) {
    // 获取options对象里的data
    let data = vm.$options.data

    // 判断data是否为函数，是函数就执行（注意this指向vm），否则就直接赋值给vm上的_data
    // 这里建议data应为一个函数，return 一个 {}，这样做的好处是防止组件的变量污染
    data = vm._data = typeof data === 'function' ? data.call(vm) : data || {}

    // 为data上的每个数据都进行代理
    // 这样做的好处就是，this.data.a可以直接this.a就可以访问了
    for (let key in data) {
        proxy(vm, '_data', key)
    }


    // 对data里的数据进行响应式处理
    // 重头戏
    observe(data)
}

// 数据代理
function proxy(object, sourceData, key) {
    Object.defineProperty(object, key, {
        // 比如本来需要this.data.a才能获取到a的数据
        // 这么做之后，this.a就可以获取到a的数据了
        get() {
            return object[sourceData][key]
        },
        // 比如本来需要this.data.a = 1才能修改a的数据
        // 这么做之后，this.a = 1就能修改a的数据了
        set(newVal) {
            object[sourceData][key] = newVal
        }
    })
}
```

可以看到vue对数据的处理是: 先initData函数,在这个函数中的proxy函数的作用是访问data数据的时候,可以直接代理到this上,而observe是在对数据做监听

### observe函数
```js
function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
可以看到 __ob__属性是判断这个数据是否可以响应式的标记;同时vue会判断这个数据是否是object,不是则不会被响应式
```js
function isObject (obj: mixed): boolean %checks {
  return obj !== null && typeof obj === 'object'
}
```

### Observer类
```js
const { arrayMethods } = require('./array')

class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // hasProto 这里是处理ie的兼容性问题
      if (hasProto) {
        // 这个方法是在改写数组的__proto__属性
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

function defineReactive(data, key, val) {
    // 新增,递归属性
    if (typeof val === 'object') {
        new Observer(val)
    }
    let dep = new Dep();
    let childOb = observe(val);
    Object.defineProperty(data, key, {
        configurable: true,
        enumerable: true,
        get() {
            dep.depend();
            // 传入的字又是一个对象需要响应式
            if (childOb) {
                childOb.dep.depend();
            }
            return val;
        },
        set(newVal) {
            if (val === newVal) return;
            // 收集依赖
            val = newVal;
            // 通知视图更新
            dep.notify();
        }
    })
}
```
- 对于对象的操作,递归对象的属性(防止对象的属性也是对象的问题)
- 对对象的每一个属性都装上get/set
- 对于数组
  - 1. 改写数组的__proto__为arrayMethods
  - 2. 对数组的每一个元素再进行深度的属性响应式
- 数组的arrayMethods
```js
/**
 * 处理ie下没有__proto__的问题,目的是改写数组的__proto__
 */
function protoAugment (target, src) {
  target.__proto__ = src
}

function copyAugment (target, src, keys) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```
```js
/* 数组的依赖在Getter中存,触发在拦截器里

数组的push,pop,unshift,shift,splice,sort,reverse
时不会触发对象的get/set,所以定义
一个拦截器覆盖Array.prototype之后每当使用Array原型上的
方法操作数组,其实都是执行的拦截器里面的方法

直接对数组的赋值是会触发set的进而通知到各自的依赖
*/
const arrayProto = Array.prototype;
const arrayMethods = Object.create(arrayProto);

['push', 'pop', 'unshift', 'shift', 'splice', 'sort', 'reverse']
.forEach(function(method) {
    const original = arrayProto[method];
    Object.defineProperty(arrayMethods, method, {
        value: function mutator (...args) {
            const result = original.apply(this, args);
            const ob = this.__ob__;
            // 新增数组的元素也需要响应式
            let inserted
            switch(method) {
                case 'push':
                case 'unshift':
                    inserted = args;
                    break;
                case 'splice':
                    inserted = args.splice(2)
                    break;
            }
            // 对新增的数据做响应式
            if (inserted) ob.observeArray(inserted)
            // 直接通知更新
            ob.dep.notify();
            return result;
        },
        writable: true,
        enumerable: false,
        configurable: true
    })
});
```