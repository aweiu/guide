# Regular 在微信小程序中的使用

随着最近一段时间微信小程序的大火，很多公司开始重视起了微信小程序上面的开发投入，甚至出现了很多类似小程序的平台，比如快应用、百度小程序、支付宝小程序等等

（注：下文提到的小程序皆指微信小程序）

## 理论基础

为什么 Regular 代码可以跑在小程序平台上？

让我们先来看下微信小程序的整体架构

```
+--------------+    +---------------+
|     View     |    |  App Service  |
+----^---+-----+    +-----^---------+
     |   |                |   |
data |   | event    event |   | data
     |   |                |   |
+----+---v----------------+---v-----+
|              JSBridge             |
+-----------------------------------+
```

View 对应视图层，App Service 对应我们的逻辑层

View 和 AppService 分别跑在两个线程中，通过 JSBridge 进行数据和事件的通信

### 实现思路

举个例子（伪代码）：

```html
<component
  bindtap="eventHandler"
  data-cid="component-id"
  data-eid="event-id"
>
  Hello {{ _holders[ 'some-id' ] }}
</component>
```

```js
const Component = Regular.extend( {
  config() {
    this.data.name = 'world'
  },
  onClick() {
    console.log( 'clicked' )
  }
} )

const vm = new Component()
```

1. 数据填充

  我们只要通过一定规则找到这些坑位的 `some-id` 标识，就可以将 `name` 填充到模板的对应位置上

2. 事件处理

  当 tap 触发的时候会执行 eventHandler，先在 eventHandler 中通过 `component-id` 找到对应的 vm（ Regular 实例 ），再通过 `event-id` 找到对应的真实事件处理函数 onClick

### 为什么设计 holders，而不传递真正的数据呢？

先来看下 holders 是长什么样子的

```js
{
  "0":  ...,
  "1": ...,
  "2-0": ...,
  "2-1": ...
}
```

是一种扁平的结构

虽然看 holders 感觉平平无奇，但其实这是 mpregular 中 `最重要的一个设计`

- 全量的复杂表达式支持

  借助 holders，我们完成了 Regular 所有复杂表达式的支持。由于逻辑层提前将所有数据计算并剥离出来，放到了一个扁平的数据结构中，__所以我们不再需要 小程序视图层 提供这些表达式特性的支持，完全摆脱了小程序本身的模板语法限制__，现在的 View 只会被用来做纯粹的渲染

- 更高效的视图渲染

  根据我们的实践，__当调用小程序的 setData 时，如果 View 层接收的数据量过大，会导致渲染时的明显卡顿__，原因可能是 View 层 diff 的数据量过大导致，在传递数据过程中有一个大列表带了很多冗余的数据过去，而这点通过警告开发者不要这么做，是很难保证的，后端经常会传输一些冗余字段到前端，比如一个数组中真正用于渲染的其实只是部分字段，如果每次都是人工代码干预的方式，过滤模板真正用到的字段，代码将会很难维护，针对这块，mpregular 内部会在 setData 前数据全部抽离出来并展平，每个数据坑位都能通过某个 id 拿到自己数据，View 的 diff 也只需要在一层上进行，就能做到更高效的更新和更好的渲染表现，在框架层面解决了这个问题，用食堂大叔的话说就是__吃多少拿多少__

### 总结

通过静态分析 Regular 模板，提前将 __数据坑位查找规则__ 和 __事件处理函数查找规则__ 在构建阶段生成到 wxml 中，到了运行时阶段，小程序的视图层和逻辑层就可以无缝衔接起来了

## 和业界框架的对比

| 特性        | 小程序    |  mpvue  |   mpregular  |
| --------   | -----:   | :----: | :----: |
| 模板语法规范        | 原生小程序     |    Vue 规范   |  Regular 规范  |
| 组件化支持        | 小程序组件规范     |    Vue组件   | Regular组件   |
| computed 计算属性  |    x   |   o    |   o |
| model 双向绑定        |   x   |  o     |  o  |
| slot 插槽        |    x   |    x   |  o  |
| filter 过滤器        |   x    |   x    |  o  |
| html 富文本        |     x  |     x  |  o  |
| 复杂表达式插值        |   x    |    x   |  o  |
| 小程序分包        |   o    |    x   |  o  |

> 目前 mpvue 的 slot 支持还存在一定问题

> 上述表格的结果统计于 2018-8-17，如果数据过时，欢迎在 guide 仓库的 issue 中指出

## 感谢

mpvue 的实现思路给了 mpregular 很多的灵感，在此表示诚挚的感谢！