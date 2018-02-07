# 初步

讲到Vue的响应式原理，我们可以从它的兼容性说起，Vue不支持IE8以下版本的浏览器，原因就是Vue是基于 [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) 来实现数据响应的，而 Object.defineProperty 是 ES5 中一个无法 shim 的特性，这也就是为什么 Vue 不支持 IE8 以及更低版本浏览器的原因；Vue通过Object.defineProperty的**getter/setter**对收集的依赖项进行监听,在属性被访问和修改时通知变化,进而更新视图数据；

受现代JavaScript 的限制 (以及废弃 **Object.observe**)，Vue不能检测到对象属性的添加或删除。由于 Vue 会在初始化实例时对属性执行**getter/setter**转化过程，所以属性必须在**data**对象上存在才能让Vue转换它，这样才能让它是响应的。

![vue响应式](https://cn.vuejs.org/images/data.png)