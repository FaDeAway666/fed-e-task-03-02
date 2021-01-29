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

## 3、请简述虚拟 DOM 中 Key 的作用和好处。

## 4、请简述 Vue 中模板编译的过程。

## 