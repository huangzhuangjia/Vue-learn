# Vue 源码分析
## 目录结构
```
├── compiler //模板解析的相关文件
│   ├── codegen //根据ast生成render函数
│   ├── directives //通用生成render函数之前需要处理的指令
│   └── parser //模板解析
|
├── core //核心代码
│   ├── components //全局组件
│   ├── global-api //全局方法，也就是添加在Vue对象上的方法，比如Vue.use,Vue.extend,,Vue.mixin等
│   ├── instance // Vue实例相关内容，包括实例方法，生命周期，事件等
│   ├── observer //双向数据绑定
│   ├── util //工具方法
│   └── vdom //虚拟dom相关，如diff算法
|
├── platforms //平台相关的内容
│   ├── web //web端独有文件
│   │   ├── compiler //编译阶段需要处理的指令和模块
│   │   ├── runtime //运行阶段需要处理的组件、指令和模块
│   │   ├── server //服务端渲染相关
│   │   └── util //工具库
│   └── weex //weex端独有文件
│       ├── compiler
│       ├── runtime
│       └── util
├── server //服务端渲染相关
│   ├── bundle-renderer
│   ├── template-renderer
│   └── webpack-plugin
├── sfc
└── shared //公共的工具方法
```