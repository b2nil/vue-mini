---
sidebarDepth: 0
---

# 定义页面

小程序中的每个页面都需要在对应的 js 文件中使用 `definePage` 函数进行定义。它是 `Page` 函数的超集，它额外接收一个 `setup` 函数。

```js
// page.js
import { definePage, reactive, computed } from '@vue-mini/wechat'

definePage({
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
})
```

如果 `setup` 返回一个对象，则对象的属性将会被合并到页面实例上，可以直接在页面模版中使用。

```xml
<!-- page.wxml -->
<button bindtap="increment">
  Count is: {{ state.count }}, double is: {{ state.double }}
</button>
```

注意 `setup` 返回的 ref 在模板中会自动解开，不需要写 `.value`。

## setup

- **调用时机**

`setup` 会在 `onLoad` 阶段被调用。返回都数据和方法也会在此时才会被合并到页面实例上，所以模版初次渲染时数据可能是 `undefined`。不过小程序模版对此做了兼容，所以不用担心会报错。

- **参数**

`setup` 函数的第一个参数与 `onLoad` 的参数相同。

```js
// page.js
import { definePage } from '@vue-mini/wechat'

definePage({
  setup(query) {
    // query 为打开当前页面路径中的参数
  }
})
```

第二个参数提供了一个上下文对象，从小程序页面 `this` 中选择性的暴露了一些 property。

```js
// page.js
import { definePage } from '@vue-mini/wechat'

definePage({
  setup(query, context) {
    context.is
    context.route
    context.getOpenerEventChannel
    // ...
  }
})
```

- **`this` 的用法**

**`this` 在 `setup()` 中不可用**。这是为了避免混乱。

## 生命周期

可以直接导入 `onXXX` 一族的函数来注册生命周期钩子。它们接收的参数和返回值与对应的生命周期一致，除 `onShareAppMessage` 和 `onAddToFavorites` 外每个 `onXXX` 函数都能被多次调用。

```js
// page.js
import { definePage, onShow, onHide, onUnload  } from '@vue-mini/wechat'

definePage({
  setup() {
    onShow(() => {
      console.log('show')
    })
    onHide(() => {
      console.log('hide')
    })
    onUnload(() => {
      console.log('unload')
    })
  }
})
```

这些生命周期钩子注册函数只能在 `setup()` 期间同步使用，其他场景下调用这些函数会抛出一个错误。

在 `setup()` 内同步创建的侦听器和计算状态会在页面销毁时自动删除。

- **onPageScroll**

监听页面滚动会引起小程序渲染层与逻辑层的通信。为避免定义空的 `onPageScroll` 监听造成不必要的性能损耗，需要使用 `definePage` 的第二个参数提前告知 Vue Mini 是否会调用 `onPageScroll()` 钩子。

```js
// page.js
import { definePage, onPageScroll  } from '@vue-mini/wechat'

definePage(
  {
    setup() {
      onPageScroll(({ scrollTop }) => {
        console.log('scrollTop:', scrollTop)
      })
    }
  },
  {
    listenPageScroll: true // 默认为 false
  }
)
```

如果已存在另外的 `onPageScroll` 选项，那么可以忽略此参数。

```js
// page.js
import { definePage, onPageScroll  } from '@vue-mini/wechat'

definePage({
  setup() {
    onPageScroll(({ scrollTop }) => {
      console.log('scrollTop (from setup):', scrollTop)
    })
  },
  onPageScroll({ scrollTop }) {
    console.log('scrollTop (from option):', scrollTop)
  }
})
```

如果以上两个条件都不满足，在 `setup()` 中调用 `onPageScroll()` 钩子会抛出一个错误。

- **onShareAppMessage**

由于 `onShareAppMessage` 会返回自定义转发内容，所以一个页面只能有一个 `onShareAppMessage` 监听。

```js
// page.js
import { definePage, onShareAppMessage  } from '@vue-mini/wechat'

definePage({
  setup() {
    // 仅第一次调用，且 `onShareAppMessage` 选项不存在才生效。
    onShareAppMessage(() => {
      return {
        title: '自定义标题',
        path: '/my/page/path',
        imageUrl: 'https://hosts.com/my-image.png'
      }
    })
  }
})
```

如果条件不满足，在 `setup()` 中调用 `onShareAppMessage()` 钩子会抛出一个错误。

- **onAddToFavorites**

由于 `onAddToFavorites` 会返回自定义收藏内容，所以一个页面只能有一个 `onAddToFavorites` 监听。

```js
// page.js
import { definePage, onAddToFavorites  } from '@vue-mini/wechat'

definePage({
  setup() {
    // 仅第一次调用，且 `onAddToFavorites` 选项不存在才生效。
    onAddToFavorites(() => {
      return {
        title: '自定义标题',
        query: 'a=xxx&b=xxx',
        imageUrl: 'https://hosts.com/my-image.png'
      }
    })
  }
})
```

如果条件不满足，在 `setup()` 中调用 `onAddToFavorites()` 钩子会抛出一个错误。

- **生命周期对应关系**

  - `onLoad` -> `setup`
  - `onShow` -> `onShow`
  - `onReady` -> `onReady`
  - `onHide` -> `onHide`
  - `onUnload` -> `onUnload`
  - `onPullDownRefresh` -> `onPullDownRefresh`
  - `onReachBottom` -> `onReachBottom`
  - `onShareAppMessage` -> `onShareAppMessage`
  - `onAddToFavorites` -> `onAddToFavorites`
  - `onPageScroll` -> `onPageScroll`
  - `onResize` -> `onResize`
  - `onTabItemTap` -> `onTabItemTap`

## 与原生语法混用

由于 `definePage()` 是 `Page()` 的超集，所以你也能使用原生语法。

```js
// page.js
import { definePage, ref } from '@vue-mini/wechat'

definePage({
  setup() {
    const count = ref(0)

    function increment() {
      count.value++
    }

    return {
      count,
      increment
    }
  },
  data: {
    number: 0
  },
  add() {
    this.setData({ number: this.data.number + 1 })
  }
})
```

如果名称相同，`setup()` 返回的数据或方法会覆盖原生语法声明的数据或方法。你应该避免出现这种情况。

请不要在其他选项中访问 `setup()` 返回的数据或方法，这将引起混乱。如果确实有此需求，应该将相关逻辑搬到 `setup()` 内。

## 简洁语法

如果不需要使用原生语法，也可以直接传递一个 `setup` 函数给 `definePage()`。

```js
// page.js
import { definePage, ref } from '@vue-mini/wechat'

definePage(() => {
  const count = ref(0)

  function increment() {
    count.value++
  }

  return {
    count,
    increment
  }
})
```