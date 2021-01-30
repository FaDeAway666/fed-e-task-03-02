# 一、简答题

## 1、请简述 Vue 首次渲染的过程。

Vue完整版的入口是entry-runtime-with-compiler.js，首次渲染的过程主要是执行Vue._init()

- 执行new Vue()，进行Vue初始化，注册静态成员和实例成员
  - 调用initGlobalAPI注册静态成员，包括
    - 静态方法 $set/$delete/$nextTick/$observable
    - options
    - keep-alive组件
    - Vue.use()
    - Vue.mixin()
    - Vue.extend()
    - Vue.directive() Vue.component() Vue.filter()
  - 在instance/index.js中初始化实例成员
    - 初始化Vue实例vm，注册_init()方法，其中包括
      - 初始化生命周期相关变量$children/$parent/$root/$refs
      - vm事件监听初始化，父组件绑定当前组件上的事件
      - vm render初始化 $slots/$scopedSlots/_c/$createElement/$attrs/$listeners
      - 调用beforeCreate生命钩子
      - 将inject成员注入到vm上
      - 初始化vm的`_props`/methods/`_data`/computed/watch
      - 初始化provide
      - 调用created生命钩子
    - 注册vm的$data/$props/$delete/$watch
    - 初始化事件的相关方法：$on/$emit/$once/$off
    - 初始化生命周期相关的混入方法_update/$forceUpdate/$destroy
    - 混入render $nextTick/_render
- 执行this._init()，合并options，进行vm生命周期、事件、render、inject、state、provide等方面的初始化，执行beforeCreate和created钩子函数
- 调用$mount()，挂载dom，这个$mount在入口文件处被重写了
  - 判断options中是否存在render，如果没有传递render
    - 将模板编译成render函数，如果没有template，则会将el作为template编译成render
    - 调用compileToFunctions()生成render渲染函数
    - options.render = render
  - 如果传递了render则调用runtime/index.js中的$mount，这个函数会调用mountComponent函数，重新获取el
    - mountComponent(this,el)
      - 位于src\core\instance\lifecycle.js
      - 判断是否有render选项，如果没有但是传入了模板，且当前是开发环境，会发送警告
      - 触发beforeMount
      - 定义updateComponent
        - 内部调用vm._render()渲染，渲染虚拟DOM
        - 内部调用vm._update()更新，将虚拟DOM转换成真实DOM，并挂载到el
      - 创建Watcher实例
        - 传入updateComponent
        - 调用get()方法
          - 创建完watcher会调用一次get方法
          - 调用updateComponent()
          - 调用vm._render()创建VNode
            - 调用render.call(vm._renderProxy, vm.$createElement)
            - 这个render是实例化时options里面的render()，或者是编译template后生成的render()
            - 返回VNode
          - 调用vm._update(vnode...)渲染DOM
            - 调用`vm.__patch__`(vm.$el, vnode)挂载真实DOM
            - 记录vm.$el
      - 触发mounted
      - return vm

## 2、请简述 Vue 响应式原理。

1. 从src\core\instance\init.js中的initState方法开始，调用initData方法，为data属性执行observe方法，将data属性变成响应式
2. observe方法执行过程
   1. 判断传入的值是否是对象，是否有`__ob__`属性，如果不是对象或者存在`__ob__`属性，直接返回
   2. 给这个值创建observer对象，将这个对象赋值给值的`__ob__`属性
   3. 进行数组的响应式处理（重写Array中所有需要更新数组的方法，对增加的数据进行依赖收集，最后派发更新）
   4. 进行对象的响应式处理，调用walk方法（遍历对象的所有属性，调用defineReactive方法）
3. defineReactive方法执行过程
   1. 为每一个属性创建dep对象
   2. 如果当前属性值是对象，调用observe方法对这个值进行响应式处理
   3. 定义getter
      1. dep.depend()收集依赖
      2. 返回属性值
   4. 定义setter（保存新值/如果新值是对象，调用observe/dep.notify()派发更新）
4. 依赖收集
   1. 在watcher对象的get的方法中调用pushTarget记录Dep.target属性，证明这是一个需要收集依赖的成员
   2. 访问data中的成员时收集依赖也就是defineReactive的getter中进行依赖收集
   3. 将属性对应的watcher对象添加到dep的subs数组中
   4. 给childOb收集依赖，目的是子对象添加和删除成员时发送通知
5. Watcher
   1. dep.notify()在调用watcher对象的update方法更新视图
   2. queueWatcher()判断watcher是否被处理，如果没有的话添加到queue中，并调用flushSchedulerQueue()
   3. flushSchedulerQueue
      1. 触发beforeUpdate钩子
      2. 调用watcher.run()：run()-->get()-->getter()-->updateComponent
      3. 清空上一次的依赖
      4. 触发actived钩子
      5. 触发updated钩子

## 3、请简述虚拟 DOM 中 Key 的作用和好处。

key主要用在updateChildren函数当中，如果不设置key，在比较新旧vnode的过程中执行sameVnode时，即使节点的内容发生了更改，但节点本身却是相同的，可以通过sameVnode的判断，此时会执行DOM操作。而如果设置了key，则不会通过sameVnode的判断，而会进行其他子节点的比较，这样就减少了DOM操作

所以设置key的好处就是能更好地比较新旧vnode之间的差异，减少不必要的DOM操作

## 4、请简述 Vue 中模板编译的过程。

1. compileToFunctions(template,...) 是编译的入口
   1. 先从缓存中加载编译好的render函数
   2. 如果缓存中没有render函数，则调用compile(template, options)
2. compile(template, options)
   1. 作用是合并传入的options和平台的options
   2. 调用baseCompile(template.trim(), finalOptions)
3. baseCompile(template.trim(), finalOptions)
   1. 调用parse()(来自于html-parse.js)，将template转换成AST
   2. 调用optimize()，标记AST 中的静态根节点，一旦检测到静态根节点，将AST的static属性设置为true，则不需要再每次重新渲染的时候重新生成节点，在patch阶段，也会跳过静态根节点（静态根节点：标签中除了文本内容以外，还需要包含其他标签）
   3. 调用generate()，生成字符串形式的js代码
4. 回到compileToFunctions(template,...)
   1. 调用createFunction()，将字符串形式的js代码转换为函数
   2. 这时候render和staticRenderFns初始化完毕，挂载到Vue实例的options对应的属性中

