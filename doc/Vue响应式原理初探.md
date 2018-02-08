# 初步

最近一段时间在阅读Vue源码，从它的核心原理入手，开始了源码的学习，而其核心原理就是其数据的响应式，讲到Vue的响应式原理，我们可以从它的兼容性说起，Vue不支持IE8以下版本的浏览器，因为Vue是基于 [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 来实现数据响应的，而 Object.defineProperty 是 ES5 中一个无法 shim 的特性，这也就是为什么 Vue 不支持 IE8 以及更低版本浏览器的原因；Vue通过Object.defineProperty的 **getter/setter** 对收集的依赖项进行监听,在属性被访问和修改时通知变化,进而更新视图数据；

受现代JavaScript 的限制 (以及废弃 **Object.observe**)，Vue不能检测到对象属性的添加或删除。由于 Vue 会在初始化实例时对属性执行 **getter/setter** 转化过程，所以属性必须在 **data** 对象上存在才能让Vue转换它，这样才能让它是响应的。

![vue响应式](https://cn.vuejs.org/images/data.png)

我们这里是根据Vue2.3源码进行分析,Vue数据响应式变化主要涉及 **Observer**, **Watcher** , **Dep** 这三个主要的类；因此要弄清Vue响应式变化需要明白这个三个类之间是如何运作联系的；以及它们的原理，负责的逻辑操作。那么我们从一个简单的Vue实例的代码来分析Vue的响应式原理
```js
var vue = new Vue({
    el: "#app",
    data: {
        name: 'Junga'
    },
    created () {
        this.helloWorld()
    },
    methods: {
        helloWorld: function() {
            console.log('my name is' + this.name)
        }
    }
    ...
})
```
# Vue初始化实例
根据Vue的[生命周期](https://cn.vuejs.org/v2/guide/instance.html#实例生命周期钩子)我们知道，Vue首先会进行init初始化操作；源码在[src/core/instance/init.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/instance/init.js)中

```js
/*初始化生命周期*/
initLifecycle(vm)
/*初始化事件*/
initEvents(vm)Object.defineProperty 
/*初始化render*/
initRender(vm)
/*调用beforeCreate钩子函数并且触发beforeCreate钩子事件*/
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
/*初始化props、methods、data、computed与watch*/
initState(vm)
initProvide(vm) // resolve provide after data/props
/*调用created钩子函数并且触发created钩子事件*/
callHook(vm, 'created')
```
以上代码可以看到 **initState(vm)** 是用来初始化props,methods,data,computed和watch;

[src/core/instance/state.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/instance/state.js)

```js
/*初始化props、methods、data、computed与watch*/
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  /*初始化props*/
  if (opts.props) initProps(vm, opts.props)
  /*初始化方法*/
  if (opts.methods) initMethods(vm, opts.methods)
  /*初始化data*/
  if (opts.data) {
    initData(vm)
  } else {
    /*该组件没有data的时候绑定一个空对象*/
    observe(vm._data = {}, true /* asRootData */)
  }
  /*初始化computed*/
  if (opts.computed) initComputed(vm, opts.computed)
  /*初始化watchers*/
  if (opts.watch) initWatch(vm, opts.watch)
}
...

/*初始化data*/
function initData (vm: Component) {

  /*得到data数据*/
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}defi
  ...
  //遍历data中的数据
  while (i--) {

    /*保证data中的key不与props中的key重复，props优先，如果有冲突会产生warning*/
    if (props && hasOwn(props, keys[i])) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${keys[i]}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(keys[i])) {
      /*判断是否是保留字段*/

      /*这里是我们前面讲过的代理，将data上面的属性代理到了vm实例上*/
      proxy(vm, `_data`, keys[i])
    }
  }
  // observe data
  /*这里通过observe实例化Observe对象，开始对数据进行绑定，asRootData用来根数据，用来计算实例化根数据的个数，下面会进行递归observe进行对深层对象的绑定。则asRootData为非true*/
  observe(data, true /* asRootData */)
}

```
## 1、initData

现在我们重点分析下**initData**，这里主要做了两件事，一是将_data上面的数据代理到vm上，二是通过执行 **observe(data, true /* asRootData */)**将所有data变成可观察的，即对data定义的每个属性进行getter/setter操作，这里就是Vue实现响应式的基础；**observe**的实现如下 [src/core/observer/index.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/observer/index.js)

```js
 /*尝试创建一个Observer实例（__ob__），如果成功创建Observer实例则返回新的Observer实例，如果已有Observer实例则返回现有的Observer实例。*/
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return
  }
  let ob: Observer | void
  /*这里用__ob__这个属性来判断是否已经有Observer实例，如果没有Observer实例则会新建一个Observer实例并赋值给__ob__这个属性，如果已有Observer实例则直接返回该Observer实例，这里可以看Observer实例化的代码def(value, '__ob__', this)*/
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    /*这里的判断是为了确保value是单纯的对象，而不是函数或者是Regexp等情况。而且该对象在shouldConvert的时候才会进行Observer。这是一个标识位，避免重复对value进行Observer
    */
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
     /*如果是根数据则计数，后面Observer中的observe的asRootData非true*/
    ob.vmCount++
  }
  return ob
}
```
这里 **new Observer(value)** 就是实现响应式的核心方法之一了，通过它将data转变可以成观察的，而这里正是我们开头说的，用了 **Object.defineProperty** 实现了data的 **getter/setter** 操作，通过 **Watcher** 来观察数据的变化，进而更新到视图中。

## 2、Observer

Observer类是将每个目标对象（即data）的键值转换成getter/setter形式，用于进行依赖收集以及调度更新。

[src/core/observer/index.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/observer/index.js)

```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    /* 将Observer实例绑定到data的__ob__属性上面去，之前说过observe的时候会先检测是否已经有__ob__对象存放Observer实例了，def方法定义可以参考/src/core/util/lang.js*/
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      /*如果是数组，将修改后可以截获响应的数组方法替换掉该数组的原型中的原生方法，达到监听数组数据变化响应的效果。这里如果当前浏览器支持__proto__属性，则直接覆盖当前数组对象原型上的原生数组方法，如果不支持该属性，则直接覆盖数组对象的原型。*/
      const augment = hasProto
        ? protoAugment  /*直接覆盖原型的方法来修改目标对象*/
        : copyAugment   /*定义（覆盖）目标对象或数组的某一个方法*/
      augment(value, arrayMethods, arrayKeys)

      /*如果是数组则需要遍历数组的每一个成员进行observe*/
      this.observeArray(value)
    } else {
      /*如果是对象则直接walk进行绑定*/
      this.walk(value)
    },

    walk (obj: Object) {
      const keys = Object.keys(obj)
      /*walk方法会遍历对象的每一个属性进行defineReactive绑定*/
      for (let i = 0; i < keys.length; i++) {
        defineReactive(obj, keys[i], obj[keys[i]])
      }
    }
  }
```
1. 首先将Observer实例绑定到data的__ob__属性上面去，防止重复绑定；
2. 若data为数组，先实现对应的[变异方法](https://cn.vuejs.org/v2/guide/list.html#变异方法)（这里变异方法是指Vue重写了数组的7种原生方法，这里不做赘述，后续再说明），再将数组的每个成员进行observe，使之成响应式数据；
3. 否则执行walk()方法，遍历data所有的数据，进行getter/setter绑定，这里的核心方法就是 **defineReative(obj, keys[i], obj[keys[i]])**

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  /*在闭包中定义一个dep对象*/
  const dep = new Dep()
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  /*如果之前该对象已经预设了getter以及setter函数则将其取出来，新定义的getter/setter中会将其执行，保证不会覆盖之前已经定义的getter/setter。*/
  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set

  /*对象的子对象递归进行observe并返回子节点的Observer对象*/
  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      /*如果原本对象拥有getter方法则执行*/
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        /*进行依赖收集*/
        dep.depend()
        if (childOb) {
          /*子对象进行依赖收集，其实就是将同一个watcher观察者实例放进了两个depend中，一个是正在本身闭包中的depend，另一个是子元素的depend*/
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          /*是数组则需要对每一个成员都进行依赖收集，如果数组的成员还是数组，则递归。*/
          dependArray(value)
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      /*通过getter方法获取当前值，与新值进行比较，一致则不需要执行下面的操作*/
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        /*如果原本对象拥有setter方法则执行setter*/
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      /*新的值需要重新进行observe，保证数据响应式*/
      childOb = observe(newVal)
      /*dep对象通知所有的观察者*/
      dep.notify()
    }
  })
}
```
其中getter方法：
1. 先为每个data声明一个**Dep**实例对象，被用于getter时执行dep.depend()进行收集相关的依赖;
2. 根据Dep.target来判断是否收集依赖，还是普通取值。Dep.target是在什么时候，如何收集的后面再说明，先简单了解它的作用，

那么问题来了，我们为啥要收集相关依赖呢？

```js
new Vue({
    template: 
        `<div>
            <span>text1:</span> {{text1}}
            <span>text2:</span> {{text2}}
        <div>`,
    data: {
        text1: 'text1',
        text2: 'text2',
        text3: 'text3'
    }
});
```
我们可以从以上代码看出，data中text3并没有被模板实际用到，为了提高代码执行效率，我们没有必要对其进行响应式处理，因此，依赖收集简单点理解就是收集只在实际页面中用到的data数据，然后打上标记，这里就是标记为Dep.target。

在setter方法中:
1. 获取新的值并且进行observe，保证数据响应式；
2. 通过dep对象通知所有观察者去更新数据，从而达到响应式效果。

在Observer类中，我们可以看到在getter时，dep会收集相关依赖，即收集依赖的watcher，然后在setter操作时候通过dep去通知watcher,此时watcher就执行变化，我们用一张图描述这三者之间的关系：
![关系图](https://github.com/huangzhuangjia/huangzhuangjia.github.io/blob/master/img/%E7%A4%BA%E6%84%8F%E5%9B%BE.png?raw=true)

从图我们可以简单理解：Dep可以看做是书店，Watcher就是书店订阅者，而Observer就是书店的书，订阅者在书店订阅书籍，就可以添加订阅者信息，一旦有新书就会通过书店给订阅者发送消息。
## 3、Watcher
Watcher是一个观察者对象。依赖收集以后Watcher对象会被保存在Dep的subs中，数据变动的时候Dep会通知Watcher实例，然后由Watcher实例回调cb进行视图的更新。

[src/core/observer/watcher.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/observer/watcher.js)
```js
export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ) {
    this.vm = vm
    /*_watchers存放订阅者实例*/
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    /*把表达式expOrFn解析成getter*/
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = function () {}
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
   /*获得getter的值并且重新进行依赖收集*/
  get () {
    /*将自身watcher观察者实例设置给Dep.target，用以依赖收集。*/
    pushTarget(this)
    let value
    const vm = this.vm

    /*执行了getter操作，看似执行了渲染操作，其实是执行了依赖收集。
      在将Dep.target设置为自生观察者实例以后，执行getter操作。
      譬如说现在的的data中可能有a、b、c三个数据，getter渲染需要依赖a跟c，
      那么在执行getter的时候就会触发a跟c两个数据的getter函数，
      在getter函数中即可判断Dep.target是否存在然后完成依赖收集，
      将该观察者对象放入闭包中的Dep的subs中去。*/
    if (this.user) {
      try {
        value = this.getter.call(vm, vm)
      } catch (e) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      }
    } else {
      value = this.getter.call(vm, vm)
    }
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    /*如果存在deep，则触发每个深层对象的依赖，追踪其变化*/
    if (this.deep) {
      /*递归每一个对象或者数组，触发它们的getter，使得对象或数组的每一个成员都被依赖收集，形成一个“深（deep）”依赖关系*/
      traverse(value)
    }

    /*将观察者实例从target栈中取出并设置给Dep.target*/
    popTarget()
    this.cleanupDeps()
    return value
  }

  /**
   * Add a dependency to this directive.
   */
   /*添加一个依赖关系到Deps集合中*/
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
   /*清理依赖收集*/
  cleanupDeps () {
    /*移除所有观察者对象*/
    ...
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
   /*
      调度者接口，当依赖发生改变的时候进行回调。
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      /*同步则执行run直接渲染视图*/
      this.run()
    } else {
      /*异步推送到观察者队列中，下一个tick时调用。*/
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
   /*
      调度者工作接口，将被调度者回调。
    */
  run () {
    if (this.active) {
      /* get操作在获取value本身也会执行getter从而调用update更新视图 */
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        /*
            即便值相同，拥有Deep属性的观察者以及在对象／数组上的观察者应该被触发更新，因为它们的值可能发生改变。
        */
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        /*设置新的值*/
        this.value = value

        /*触发回调*/
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
   /*获取观察者的值*/
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
   /*收集该watcher的所有deps依赖*/
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
   /*将自身从所有依赖收集订阅列表删除*/
  teardown () {
   ...
  }
}
```

## 4、Dep
被Observer的data在触发 **getter** 时，Dep就会收集依赖的Watcher，其实Dep就像刚才说的是一个书店，可以接受多个订阅者的订阅，当有新书时即在data变动时，就会通过Dep给Watcher发通知进行更新。

[src/core/observer/dep.js](https://github.com/huangzhuangjia/Vue-learn/blob/master/core/observer/dep.js)

```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  /*添加一个观察者对象*/
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  /*移除一个观察者对象*/
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  /*依赖收集，当存在Dep.target的时候添加观察者对象*/
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  /*通知所有订阅者*/
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```
# 总结
其实在Vue中初始化渲染时，视图上绑定的数据就会实例化一个Watcher，依赖收集就是是通过属性的 getter 函数完成的，文章一开始讲到的 Observer、Watcher、Dep 都与依赖收集相关。其中 Observer 与 Dep 是一对一的关系， Dep 与 Watcher 是多对多的关系，Dep 则是 Observer 和 Watcher 之间的纽带。依赖收集完成后，当属性变化会执行被Observer对象的 dep.notify 方法，这个方法会遍历订阅者（Watcher）列表向其发送消息，Watcher 会执行 run 方法去更新视图，我们再来看一张图总结一下：
![关系图](https://github.com/huangzhuangjia/Vue-learn/blob/master/doc/img/vue-reactive.jpg?raw=true)

1. 在 **Vue** 中模板编译过程中的指令或者数据绑定都会实例化一个 **Watcher** 实例，实例化过程中会触发 **get()** 将自身指向 **Dep.target**;
2. data在 **Observer** 时执行 **getter** 会触发 **dep.depend()** 进行依赖收集;依赖收集的结果：1、data在 **Observer** 时闭包的dep实例的subs添加观察它的 **Watcher** 实例；2. **Watcher** 的deps中添加观察对象 **Observer** 时的闭包dep；
3. 当data中被 **Observer** 的某个对象值变化后，触发subs中观察它的watcher执行 **update()** 方法，最后实际上是调用watcher的回调函数cb，进而更新视图。

# 参考

* [Vue源码](https://github.com/vuejs/vue)
* [Vue文档](https://cn.vuejs.org)
* [Vue源码学习](https://github.com/answershuto/learnVue)