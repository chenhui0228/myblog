---
title: vuejs-study-1
categories:
  - 前端技术
  - Vue
tags:
  - 前端
  - WVVM
  - Vue
date: 2017-08-15 09:18:54
---

----------

**转载申明**：本文转载博主[Xheldon](http://www.xheldon.com)的文章[Vue学习总结](http://www.xheldon.com/vue-learning-summary.html)，转载请注明出处！

----------


这篇文章从Vue 2.0官方文档的”实例”一节开始, 研究一些 Vue API 的使用方法, 以及 Vue 实现一些功能的原理, 此外还有自己的使用感受, 以及站在自己浅薄的角度分析 Vue 为什么要这么设计的不敬之举, 如有得罪还请海涵, 刚接触 Vue 不久, 不当之处烦请指出, 先行谢过。

> **注意**: 基础知识直接略过, 我只说我认为需要说的点。

#### Vue实例 ####

每个 Vue.js 的应用都是通过构造函数创建一个 Vue 的根实例启动的, 意思就是, 每个页面的数据都应该只由这一个实例维护, 原始数据的来源都应该只由根实例来发出和接收统一管理, 根实例再通过 props, 分发数据, 或者 events 来监听数据. 子组件只需要 watch/computed 数据变化, 及时更新即可。

文档中说了一句话叫: 所有的 Vue.js 组件其实都是被扩展的 Vue 实例, 这句话正确理解起来应该是, 你可以在组件上使用和实例一样的方法和钩子函数, 除了 data 。

组件中的 data, 必须是一个函数, 因为组件会被复用, 所以必须每次调用组件都生成一份数据。

数据代理(proxy), 指的是 Vue 实例会代理其 data 对象中的所有属性, 而实例属性 $data 则表示 data 属性本身, 以区别被代理的 data。

意思是, 如果一个 vm 的 data 属性为 {a: 'xheldon'}, 那么 vm.a 即为 'xheldon', 而 vm.$data 则为 {a: 'xheldon'}。

组件其实也是一个(被扩展的) Vue 实例, 下面是个简单的验证:

有一个 list.vue 组件(template 和 style 省略)：

```javascript
import Vue from 'vue'
export default{
  data(){
    console.log(this instanceof Vue);//true
    return {}
  }
  name: 'com-list'
}
```

#### props vs data ####

初始化组件的时候, prop 上的属性和 data 上的属性以及 computed 的方法, 都被绑定到 Vue 实例上了, 但是 porps 上的属性, 优先级比 data 同名属性要高,下面是验证:

```javascript
export default{
  data(){
    console.log('first:',this);//list 的实例
    return {
      a: 'a',//属性 a 代表的值挂在 data 上, 但是被下面的 prop 属性同名覆盖(查看上面控制台输出的内容即可)
      d: 'd'
    }
  },
  props: ['a'],//和 data 中的 a 属性同名, 因此来自父级的数据将 data 中的同名属性 a 上的数据覆盖.(注:父级是根组件, 挂载在一个实例上)
  name: 'com-list'
}
```

结果出现警告(不是报错, 不影响渲染):

{% asset_img props-vs-data.png %}

而 computed 返回的函数名和 data 上的属性名可以重复, 并且不会有任何提示, 但是同名覆盖了之后, 因为在初始化的时候, 从控制台可以看到 _data 后于 _computedWatcher 来设置这个重复属性的 getter 和 setter(不知道是不是这个原因, 先放在这个地方以待我深入研究之后再修改这篇文章), 因此导致了被覆盖. 相关原理可以看[这篇介绍](https://juejin.im/entry/577639de165abd00547b0924)

它们都在初始化的时候绑定到了实例 属性 上, 同名的 computed 属性被覆盖了, 但是 Vue devtool 仍然正确显示了出来)

```javascript
export default{
  data(){
    console.log('first:',this);//list 的实例
    return {
      ab: 'a',//属性 a 代表的值挂在 data 上, 但是被下面的 prop 属性同名覆盖(查看上面控制台输出的内容即可)
    }
  },
  computed: {
    ab(){
      return 'computed a';//方法名跟 data 上的 a 属性同名, 因此console 出来的 this 不会出现它的值, 用花括号输出的时候也是输出的 data 上的同名属性
    },
    f(){
      return 'computed f'//方法名没有重复的, 因此 console 出来的 this 会有它的同名属性, 且值为 'computed f', 我们提供的 function 被作为该属性的 getter(计算属性默认只有 getter, 可以手动添加 setter)
    }
  },
  name: 'com-list'
}
```

{% asset_img computed-vs-data1.png %}

Vue devtool 正确显示了出来:

{% asset_img computed-vs-data3.png %}


