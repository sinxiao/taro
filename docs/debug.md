---
title: Debug 指南
---

和所有框架一样，Taro 也可能存在 bug。当你认为你的代码没有问题，问题出在 Taro 时，可以按照本章内容进行调试。

当你在 Taro 进行 debug 时，请先确认一下流程均已完成：

1. ESLint 已经开启并且没有报错；
2. 大致过了一遍包括[最佳实践](https://nervjs.github.io/taro/docs/best-practice.html) 在内的文档，文档里没有对应问题的描述;
3. 搜索过相关的 issue，issue 没有提到相关解决方案；
4. 按项目使用的 Taro 版本往上查看 [changelog](https://github.com/NervJS/taro/blob/master/CHANGELOG.md)，changelog 中没有意见修复相关问题的提交

很多时候只要你把以上四个流程都走一遍，遇到的问题就会迎刃而解。而作为一个多端框架，Taro 有非常多的模块，当出现问题时 Taro 也需要分模块进行调试，接下来我们会举一些已经解决了的 bug 样例，阐述我们调试 bug 的思路：

## 小程序

### 没有任何报错，但显示的结果不如预期

这时候很可能是编译模板出现了错误。例如中 [#2285](https://github.com/NervJS/taro/issues/2258) 中，题主写了两个嵌套循环，在第二个循环中无法正确地访问到第一个循环声明的 `index` 变量：

```jsx
// 假设源码在 src/pages/index/index.js 中
rooms.map((room, index) => (
  <View key={room.id}>
    <View>房间</View>
    <View className="men">
      {room.checkInMen.map(man => (
        <View onClick={this.handleRemoveMan.bind(this, man.id, index)}>
          {man.name}
        </View>
      ))}
    </View>
  </View>
);
```

而编译出来的 `wxml` 将会是：

```html
<!-- 编译后代码代码至少会生成三个文件，分别是: -->
<!-- dist/pages/index/index.js，dist/pages/index/index.wxml，dist/pages/index/index.json -->
<view wx:for="{{loopArray0}}" wx:for-item="room" wx:for-index="index">
  <view>房间</view>
    <view class="men">
      <view  data-e-tap-a-b="{{index}}" bindtap="handleRemoveMan" wx:for="{{room.$anonymousCallee__0}}" wx:for-item="man" data-e-tap-so="this" data-e-tap-a-a="{{man.$original.id}}">{{man.$original.name}}
      </view>
    </view>
  </view>
</view>
```

观察编译前后文件，我们可以发现：由于第二个循环没有指定 `index` 变量名，taro 编译的循环也没有指定 `index` 变量名。但问题在于微信小程序当不指定 `index` 时，会隐式地注入一个名为 `index` 的变量名作为 `index`。因此这段代码在第二个循环中访问 `index`，实际上是当前循环的 `index`，而不是上级循环的 `index`。

当我们了解到问题所在之后，解决问题也很容易，只要在第二个循环显式地暴露循环的第二个变量即可，源代码可以修改为：

```jsx
rooms.map((room, index) => (
  <View key={room.id}>
    <View>房间</View>
    <View className="men">
      {room.checkInMen.map((man, _) => (
        <View onClick={this.handleRemoveMan.bind(this, man.id, index)}>
          {man.name}
        </View>
      ))}
    </View>
  </View>
);
```

### 运行时在小程序开发者工具报错

有时候我们会在运行时遇到这样错误：

![debug.png](https://i.loli.net/2019/02/27/5c765b5bc1915.png)

调试这样的问题也很简单，只需要点击调用栈从调用栈最上层的链接，点进去我们可以发现是这样的代码：

![debug2.png](https://i.loli.net/2019/02/28/5c77517c622e3.png)

这时我们可以发现这个错误的原因在于变量 `url` 在调用 `Object.assign()` 函数时找不到变量，我们可以再看一下源码：

```jsx
// 如果运行时报错文件路径是：dist/pages/test/test.js
// 那么就可以推算出源码在：src/pages/test/test.js
// 编译后的 js 文件已经经过 Babel 编译过，但函数基本上还是能一一对应的
// 除了 `render()` 函数会对应到 `_createData()` 函数，形如 `renderTest` 函数会对应到 `createTestData` 函数
render () {
  let dom = null
  if (this.props.visable) {
      const url = 'https://...'
      dom = <Image src={url} />
  }
  
  return <Container>
    {dom}
  </Container>
}
```

通过观察编译前后代码，我们可以发现源码没有任何问题，但 Taro 在此问题出现的版本没有处理好 if 表达式作用域内的变量，调用 `Object.assign()` 函数时 `url` 变量并不存在于 `render` 函数的作用域中。为了解决这个问题，我们可以修改源码，手动把 `url` 变量也放在 `render` 函数作用域中：

```jsx
render () {
  let dom = null
  let url = ''
  if (this.props.visable) {
      url = 'https://...'
      dom = <Image src={url} />
  }
  
  return <Container>
    {dom}
  </Container>
}
```

大部分运行时错误都可以通过小程序内置的 Chrome Devtools 找到报错的缘由，如果当前调用栈没有找到问题所在，可以往上逐层地去调试各个调用栈。Chrome DevTools 相关文档请查看：[Chrome 开发者工具](https://developers.google.com/web/tools/chrome-devtools/)

### 生命周期/路由/setState 出错

在 [#1814](https://github.com/NervJS/taro/issues/1814) 中提到了 `this.$router.path` (当前页面路由的路径) 有时无法访问。经过调研发现原因在于 Taro 把获取路径的函数放在了小程序的 `onLoad` 函数上，而不是每个组件都能调用到这个函数。而解决这个问题的方法也很简单，如果当前页面是组件可以直接通过 `this.$scope.route` 访问，更普适的方法则是通过 `getCurrentPages` 函数访问到当前页面的示例，然后访问实例的 `route` 或 `__route__` 访问到当前页面路由的路径。

通过这个例子，我们不难发现 Taro 的生命周期/路由 和 `setState` 在小程序端其实是包装成 React API 的一层语法糖，我们把这层包装称之为 Taro 运行时框架。几乎所有 Taro 提供的 API 和语法糖最终都是通过小程序本身提供的 API 实现的，也就是说当 Taro 运行时框架出现问题时，你基本都能使用小程序本身提供的 API 达到同等的需求，其中就包括但不限于：

1. 使用 `this.$scope.triggerEvent` 调用通过 props 传递的函数;
2. 通过 `this.$scope.selectComponent` 和 `wx.createSelectorQuery` 实现 `ref`;
3. 通过 `getCurrentPages` 等相关方法访问路由；
4. 修改编译后文件 `createComponent` 函数创建的对象

虽然使用小程序原生方法也能做很多同样的事，但当 Taro 运行时框架出现问题时，我们还是强烈建议开发者向 Taro 官方 [提交 issue](https://github.com/NervJS/taro/issues/new/choose)，有能力的开发者朋友也可以 [提交 PR](https://github.com/NervJS/taro/pulls)。一方面使用 Taro API 实现可以帮助你抹平多端差异，另一方面寻找甚至是修复 bug 也有助于加强你对 Taro 和小程序底层的理解。

## H5

### 运行时报 DOM 相关错误

在 [#1804](https://github.com/NervJS/taro/issues/1804) 中提到，只要使用了 `Block` 组件并且有一个变量控制它的显式时，就必定会报错：

```jsx
export default class Index extends Component {
  config = {
    navigationBarTitleText: '首页'
  };

  state = {
    num: 1
  };

  componentDidMount() {
    console.log('did');
    setTimeout(() => {
      this.setState({
        num: 0
      });
    }, 2000);
  }

  render() {
    const { num } = this.state;
    return (
      <View className="container">
        {num === 0 && <View>已卖完</View>}
        {num > 0 && (
          <Block>
            <View>正在销售</View>
            <View>立即购买</View>
          </Block>
        )}
        {/* {num > 0 && <View>正在销售</View>} */}
      </View>
    );
  }
}
```

这个时候我们可以把问题定位到 `Block` 组件中，我们可以查看 `@tarojs/components` 的 `Block` 组件源码：

  ```jsx
const Block = (props) =>  props.children
export default Block
```

也就是说当变量 `num > 0` 时，`Block` 组件的 `children` 会显示，而当 `Block` 组件的 `children` 是一个数组时，`View.container` 的 `children` 就变成 `[一个 View 组件, [一个数组]]`，渲染这样的数据结构需要 `React.Fragment` 的包裹才能渲染。而 Taro 目前还没有支持 `React.Fragment` 语法，所以这样的写法就报错了。解决这个问题也很简单，只需要修改 `Block` 组件，用一个元素包裹住 `children` 即可:

```jsx
const Block = (props) => <div>{props.children}</div>
```

当你遇到了相关问题时，我们准备了一个快速起步的沙盒工具，你可以直接在这个工具里编辑、调试、复现问题：

[![Edit Taro sandbox](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/50nzv622z4?fontsize=14)

## 其它资源

本文列举了一些 Taro 的已解决 bug 例子，阐述了在 Taro 中 debug 的思路，但在实际操作中如果你能更深入地了解 Taro 的实现原理，那无论是对使用 Taro 或是 debug 都会有很大的帮助。以下资源从各个方面都介绍了 Taro 的实现原理：

* 掘金小册：[Taro 多端开发实现原理与实战](https://juejin.im/book/5b73a131f265da28065fb1cd?referrer=5ba228f16fb9a05d3251492d)
* 博文：[Taro 诞生记](https://aotu.io/notes/2018/06/25/the-birth-of-taro/)
* 公开演讲: [使用 Taro 快速开发多端项目](https://share.weiyun.com/5nUKYfy)
* 公开演讲： [基于 Taro 的多端项⽬目实践](https://share.weiyun.com/5lZXV1G)
