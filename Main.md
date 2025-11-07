# 微信小程序核心知识总结

## 目录
- [一、小程序框架原理](#一小程序框架原理)
- [二、小程序兼容问题](#二小程序兼容问题)
- [三、小程序性能优化](#三小程序性能优化)

---

## 一、小程序框架原理

### 1.1 双线程架构

小程序与传统web单线程架构不同，采用**双线程架构**：

- **渲染层**：使用 WebView 进行渲染，一个页面对应一个 WebView
- **逻辑层**：采用 JSCore 运行 JS 代码
- **通信机制**：两个线程通过 Native 层（微信客户端）进行转发通信

**核心特点**：
- 渲染层和逻辑层分离，所有数据传递都是线程间通信
- **一切都是异步的**
- 通过 `setData` 方法进行数据传递

```javascript
// 基本示例
Page({
  onLoad: function () {
    this.setData({ msg: 'Hello World' })
  }
})
```

### 1.2 为什么使用双线程架构？

**优势：**
1. **管控与安全**：防止脚本获取修改页面敏感内容或随意跳转
2. **性能更好**：渲染层和逻辑层并行不阻塞
3. **体验更流畅**：多个 WebView，页面切换更接近原生 App

**代价：**
- 任何数据传递都有延时
- 需要通过 Native 层转发，增加通信成本

### 1.3 快速渲染设计 - PageFrame

#### WebView 数量限制
- 微信小程序限制最多打开 **10 个页面**
- 达到限制后无法再打开新页面
- **开发建议**：避免路由嵌套太深

#### PageFrame 模板预加载机制

小程序启动时，除了渲染首页，还会预加载一个空的 WebView 模板（`pageframe.html/instanceframe.html`）：

**模板包含的核心 JS 资源：**
- `wxconfig.js`：小程序配置项（`__wxConfig`）
- `devtoolsconfig.js`：开发者配置（导航栏、状态栏高度等）
- `deviceinfo.js`：设备信息
- `WAWebview.js`：渲染层底层基础库
- `wxappcode.js`：页面的 json/wxml/wxss 编译结果

**快速启动原理：**
1. 首页启动后，后台服务缓存 `pageframe.html` 内容
2. 后续打开新页面时，直接走缓存
3. 只需将页面特定的 json/wxml/wxss 注入到预加载模板中

**首次打开新页面流程：**
```
启动空 WebView → 加载 pageframe.html → DOM Ready 
→ 通过 history.pushState 修改路径 → 注入页面代码 → 渲染
```

### 1.4 WXML 设计思路

#### Exparser 框架

小程序自行搭建了 **Exparser 组件框架**，不依赖浏览器的 WebComponents：

**特点：**
- 基于 Shadow DOM 模型（但不依赖浏览器原生支持）
- 可在纯 JS 环境中运行
- 高效轻量，性能表现好

**组件转换示例：**
```html
<!-- 源码 -->
<view class="container">
  <text>文本</text>
</view>

<!-- 转换后 -->
<wx-view exparser:info-component-id="2" class="container">
  <wx-text exparser:info-component-id="3">
    <span>文本</span>
  </wx-text>
</wx-view>
```

#### 原生组件

部分组件由客户端创建，不在 Exparser 渲染体系下：
- camera、canvas、input（focus时）、textarea、video、map
- live-player、live-pusher

**优势：**
1. 扩展 Web 能力（如更好的键盘控制）
2. 体验更好，减轻 WebView 渲染负担
3. 绕过 setData 和数据通信，性能更好

### 1.5 WXSS 设计思路

#### RPX 响应式单位

**rpx（responsive pixel）**：根据屏幕宽度自适应
- 规定屏幕宽为 **750rpx**
- iPhone6：750rpx = 375px = 750物理像素

**编译原理：**
```javascript
// 核心转换公式
px = rpx值 / 750 * 设备宽度
number = Math.floor(number + 0.0001) // 精度收拢
```

#### 编译流程

WXSS 会被编译成 JS 注入到 WebView：

```css
/* 源码 */
.test {
  height: calc(100rpx-2px);
  width: 200rpx;
}

/* 编译后结构化数据 */
[".", [1], "test{ height: calc(", [0, 100], "-2px); width: ", [0, 200], "; }\n"]
```

### 1.6 Virtual DOM 渲染流程

#### WXML 编译

WXML 被编译成 `$gwx` 函数，执行后生成 Virtual DOM 树：

```javascript
var decodeName = decodeURI("./pages/index/index.wxml")
var generateFunc = $gwx(decodeName)
// 执行 generateFunc() 返回 Virtual DOM 树
```

**数据驱动原理：**
1. WXML 转成 JS 对象（Virtual DOM）
2. setData 改变数据，产生新的 JS 对象
3. 对比前后差异（Diff）
4. 把差异应用到 DOM 树，更新 UI

### 1.7 事件系统设计

#### 事件绑定原理

WXML 中的事件绑定转换为 Virtual DOM 后只是键值对标记：

```javascript
// WAWebview.js 处理事件
if (n = e.match(/^(capture-)?(mut-)?(bind|catch):?(.+)$/)) {
  // 通过 addListener 绑定事件
  // 触发时通过 sendData 发送到逻辑层
}
```

**流程：**
1. 渲染层识别事件属性
2. 通过 `addListener` 绑定
3. 事件触发时组装 event 参数
4. 通过 `sendData` 发送到逻辑层
5. 逻辑层执行对应回调

#### 事件类型

- **bind**：事件冒泡
- **catch**：阻止事件冒泡
- **mut-bind**：互斥事件（触发后，同级其他 mut-bind 不触发）
- **capture-bind/capture-catch**：捕获阶段监听

### 1.8 通信系统设计

通过 **WeixinJSBridge** 实现跨线程通信：

- **iOS**：利用 WKWebView 的 `messageHandlers` 特性
- **Android**：向 WebView 的 window 对象注入原生方法

Native 层分别在视图层和逻辑层注入 `WeixinJSBridge`，实现双向通信。

### 1.9 生命周期设计

#### 页面生命周期

```javascript
Page({
  onLoad(query) {},    // 页面加载，只调用一次
  onShow() {},         // 页面显示/切入前台
  onReady() {},        // 首次渲染完成，只调用一次
  onHide() {},         // 页面隐藏/切入后台
  onUnload() {}        // 页面卸载
})
```

**与路由的关系：**
- `wx.navigateTo`：创建新 WebView，当前页面 → onHide
- `wx.redirectTo`：更新当前 WebView，当前页面 → onUnload
- `wx.navigateBack`：销毁 WebView，当前页面 → onUnload

#### 初始化流程

1. 渲染层和逻辑层各自初始化
2. 渲染层初始化完成后发送信号给逻辑层
3. 逻辑层收到信号后发送初始数据到渲染层
4. 渲染层接收数据，开始首次渲染

### 1.10 路由设计

#### 路由栈

Native 层维护路由栈，统一控制 WebView 的创建和销毁：

**流程：**
1. 逻辑层/视图层触发路由行为
2. 发送到 Native 层
3. Native 通过 PageFrame 快速创建 WebView
4. 推入路由栈
5. 通知逻辑层创建页面组件，开启生命周期

---

## 二、小程序兼容问题

### 2.1 橡皮筋回弹

iOS 系统下，滚动到顶部或底部会出现回弹效果，影响用户体验。

**解决方案：**
- 使用 `scroll-view` 组件代替 `view`
- 禁用页面滚动：`"disableScroll": true`

### 2.2 滚动穿透

弹窗打开时，背景页面仍可滚动。

**解决方案：**
```javascript
// 方案1: 动态控制页面滚动
wx.showModal({
  success: () => {
    wx.pageScrollTo({ scrollTop: 0, duration: 0 })
  }
})

// 方案2: 使用 catchtouchmove 阻止冒泡
<view catchtouchmove="preventMove"></view>
```

### 2.3 WebView 缓存

小程序 WebView 组件会缓存页面，导致内容不更新。

**解决方案：**
- URL 添加时间戳：`url + '?t=' + Date.now()`
- 服务端设置 Cache-Control 响应头

### 2.4 安全区域适配

iPhone X 等全面屏机型需要适配安全区域。

**解决方案：**
```javascript
// 获取安全区域信息
const { safeArea, screenHeight } = wx.getSystemInfoSync()
const bottomSafeHeight = screenHeight - safeArea.bottom

// CSS 适配
.bottom-bar {
  padding-bottom: env(safe-area-inset-bottom);
}
```

```html
<!-- WebView 内适配 -->
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
```

### 2.5 scroll-view 自带节流

`scroll-view` 的 scroll 事件自带节流，`scrollTop` 值可能不准确。

**解决方案：**
- 使用防抖处理
- 关键数据通过 `scrolltoupper/scrolltolower` 获取

### 2.6 页面跳转层级限制

页面跳转层级最大为 **10 层**。

**解决方案：**
- 合理使用 `wx.redirectTo` 替代 `wx.navigateTo`
- 重要页面使用 Tab 页
- 层级深的页面考虑重构

### 2.7 WebView 加载字体过小

小程序 web-view 加载网页时字体显示异常小。

**解决方案：**
```html
<meta name="viewport" content="width=device-width,initial-scale=1,minimum-scale=1,
  maximum-scale=1,user-scalable=no,viewport-fit=cover">
```

### 2.8 使用 root-portal 实现全局弹窗

类似 Vue 的 Teleport，将组件渲染到页面根节点。

```vue
<template>
  <!-- #ifdef H5 -->
  <teleport to="body">
  <!-- #endif -->
  <!-- #ifdef MP-WEIXIN || MP-ALIPAY -->
  <root-portal>
  <!-- #endif -->
    <slot />
  <!-- #ifdef MP-WEIXIN || MP-ALIPAY -->
  </root-portal>
  <!-- #endif -->
  <!-- #ifdef H5 -->
  </teleport>
  <!-- #endif -->
</template>
```

**原理：**
```javascript
class RootPortalComponent {
  attached() {
    const content = this.getSlotContent()
    this.detachFromParent()           // 从当前组件树剥离
    const pageRoot = this.getPageRootNode()
    pageRoot.appendChild(content)      // 插入到页面根节点
    this.maintainDataBinding()         // 保持数据绑定
  }
}
```

---

## 三、小程序性能优化

### 3.1 小程序启动流程

```
资源准备 → 代码注入 → 首屏渲染
```

#### 资源准备阶段
1. **小程序信息准备**：头像、昵称、版本、配置、权限等（缓存）
2. **环境预加载**：根据使用场景预加载运行环境（不一定命中）
3. **代码包准备**：从 CDN 下载代码包（缓存 + 增量更新）

**微信的优化手段：**
- 代码包压缩
- 增量更新
- 优先使用 QUIC 和 HTTP/2
- 预先建立连接
- 代码包复用（MD5 签名）

#### 代码注入阶段
- 读取配置和代码，注入 JS 引擎
- 使用 V8 的 **Code Caching** 缓存编译结果

### 3.2 冷启动优化（4大方法）

#### 方法1：控制代码包体积 ⭐⭐⭐

**优化手段：**

1. **使用分包加载**
   ```json
   {
     "pages": ["pages/index/index"],
     "subpackages": [{
       "root": "pages-sub",
       "pages": ["detail/detail"]
     }]
   }
   ```

2. **删除/置空非必要的包**
   - 删除错误打入 `common/vendor.js` 的包
   - 使用 `null-loader` 置空处理
   - 移出非首屏包到具体业务

3. **资源文件放 CDN**
   - 字体、图片统一使用外部链接
   - 避免 base64 内联大图

4. **npm 包和组件分包异步化**
   - 非首屏 npm 包转移到子包
   - 业务组件分包异步加载

**效果：** 代码包体积减少 ~50%，点击启动耗时从 554ms → 332ms

#### 方法2：代码注入优化 ⭐⭐⭐

**1) 启用按需注入**
```json
{
  "lazyCodeLoading": "requiredComponents"
}
```

**2) 启用用时注入（占位组件）**
```json
{
  "usingComponents": {
    "heavy-component": "/components/heavy"
  },
  "componentPlaceholder": {
    "heavy-component": "view"
  }
}
```

**3) 优化主线逻辑**
- 减少阻塞启动的 API 调用（如 Sync 结尾的 API）
- 优先使用异步 API：`getSystemInfoAsync`
- 简化首屏逻辑，延迟到首屏后执行

#### 方法3：数据预拉取 ⭐⭐⭐

##### 数据预拉取（Pre-fetch）

**机制说明：**
在小程序启动前，通过微信后台提前向第三方服务器拉取业务数据。当代码包加载完时可以更快地渲染页面，减少用户等待时间。

**使用场景：**
- 冷启动时预拉取首页数据
- 提前获取用户信息、配置信息等

**配置步骤：**

1. **后台配置**：在小程序管理后台设置预拉取域名

2. **代码中使用**：
```javascript
// app.js
App({
  onLaunch() {
    // 获取预拉取的数据
    wx.getBackgroundFetchData({
      fetchType: 'pre',
      success(res) {
        console.log('预拉取数据:', res.fetchedData)
        // 可以将数据存储到全局或缓存
        wx.setStorageSync('preFetchData', res.fetchedData)
      }
    })
  }
})
```

3. **设置预拉取数据**：
```javascript
// 在适当时机设置 token，告诉微信后台去哪个接口拉取数据
wx.setBackgroundFetchToken({
  token: 'your-fetch-token'
})
```

##### 周期性更新（Periodic Update）

**机制说明：**
在用户未打开小程序的情况下，也能从服务器提前拉取数据。当用户打开小程序时可以更快地渲染页面，增强弱网条件下的可用性。

**特点：**
- 即使用户没有打开小程序，也会定期更新数据
- 适合新闻、资讯类小程序
- 更新频率由微信控制（不低于 12 小时）

**配置步骤：**

1. **后台配置**：开启周期性更新能力

2. **代码中使用**：
```javascript
// app.js
App({
  onLaunch() {
    // 获取周期性更新的数据
    wx.getBackgroundFetchData({
      fetchType: 'periodic',
      success(res) {
        console.log('周期性更新数据:', res.fetchedData)
        // 更新本地缓存
        this.updateLocalData(res.fetchedData)
      }
    })
  }
})

// 设置周期性更新的 token
wx.setBackgroundFetchToken({
  token: 'periodic-update-token'
})
```

**数据预拉取 vs 周期性更新对比：**

| 特性 | 数据预拉取 | 周期性更新 |
|------|----------|----------|
| 触发时机 | 小程序启动时 | 定期后台更新 |
| 是否需要打开小程序 | 是 | 否 |
| 使用场景 | 冷启动优化 | 内容预加载 |
| fetchType | 'pre' | 'periodic' |

**优化策略：**
1. 合并关键接口（如登录 + 数据接口）
2. 使用数据预拉取提前请求时机
3. 控制预拉取数据量在首屏范围内
4. 结合缓存策略，避免重复请求

**效果：** 减少串行接口调用时间，提升首屏加载速度

#### 方法4：首屏渲染优化 ⭐⭐⭐

**1) 启用初始渲染缓存**
```json
{
  "initialRenderingCache": "static"
}
```

**原理：** 视图层不等待逻辑层初始化，直接展示缓存的初始 data

**2) 控制渲染优先级**

**优先级划分：**
- **P0**：纯静态内容，一次渲染成功
- **P1**：首屏必要数据（预拉取 + 控制数量）
- **P2**：其他功能，延迟加载

```javascript
// 延迟加载示例
Page({
  onReady() {
    setTimeout(() => {
      this.loadSecondaryContent()
    }, 300)
  }
})
```

**3) 提前发起数据请求**
```javascript
// 跳转时提前请求下一页数据
wx.navigateTo({
  url: '/pages/detail/detail',
  events: {
    acceptDataFromOpener: (data) => {}
  },
  success: (res) => {
    // 立即发起请求
    fetchDetailData().then(data => {
      res.eventChannel.emit('acceptDataFromOpener', data)
    })
  }
})
```

**4) 控制预加载时机**
```json
{
  "handleWebviewPreload": "manual"
}
```

**5) 骨架屏**
在数据加载时展示页面结构，提升感知性能。

**优化效果：** 冷启动耗时降低 **70%+**

### 3.3 运行时性能优化

#### 3.3.1 合理使用 setData ⭐⭐⭐

**问题：**
- setData 是跨线程通信，有延迟
- 频繁调用会阻塞渲染

**优化策略：**

**1) 控制频率**
```javascript
// ❌ 不好的做法
for (let i = 0; i < 100; i++) {
  this.setData({ count: i })
}

// ✅ 好的做法
let count = 0
for (let i = 0; i < 100; i++) {
  count = i
}
this.setData({ count })
```

**2) 控制范围**
```javascript
// ❌ 更新整个数组
this.setData({ list: newList })

// ✅ 只更新变化的项
this.setData({
  [`list[${index}].name`]: newName
})
```

**3) 控制内容**
```javascript
// ❌ 传递大量数据
this.setData({ 
  userInfo: { /* 100+ 字段 */ }
})

// ✅ 只传递需要的数据
this.setData({
  'userInfo.name': name,
  'userInfo.avatar': avatar
})
```

#### 3.3.2 页面渲染优化

**1) 适当监听 scroll 事件**
```javascript
// 使用节流
onPageScroll: throttle(function(e) {
  // 处理滚动
}, 100)
```

**2) 控制 WXML 节点数量和层级**
- 单个页面节点数不超过 **16000**
- 避免过深的层级嵌套
- data 层级不要过深（影响遍历性能）

**3) 使用 IntersectionObserver 监听曝光**
```javascript
const observer = wx.createIntersectionObserver()
observer.observe('.item', (res) => {
  if (res.intersectionRatio > 0) {
    // 元素可见，加载内容
  }
})
```

#### 3.3.3 页面切换优化

**1) 避免在 onHide/onUnload 执行耗时操作**
```javascript
// ❌ 阻塞页面切换
onUnload() {
  heavyOperation() // 耗时操作
}

// ✅ 必要时使用异步
onUnload() {
  setTimeout(() => {
    heavyOperation()
  }, 0)
}
```

**2) 提前发起数据请求**（见上文）

**3) 控制预加载时机**
- 默认：onReady 后 200ms 预加载下一页
- 可配置 `handleWebviewPreload` 手动控制

#### 3.3.4 分包预下载机制 ⭐⭐⭐

##### 背景与问题

尽管分包加载能够提升启动速度，但当用户跳转到分包内页面时，仍需等待分包下载，这可能导致页面切换的延迟。为解决这一问题，小程序引入了**分包预下载机制**。

##### 机制说明

分包预下载机制允许小程序在后台预先下载分包，确保用户在首次进入分包页面时无需等待下载，从而提升页面切换的流畅性。

##### 预下载原理

**工作流程：**
1. 分包预下载利用小程序框架的预加载能力，在用户打开小程序时提前加载分包资源
2. 当用户需要访问对应分包页面时，可以更快地展示内容，减少加载等待时间
3. 预下载机制通常在主包的某些关键页面或事件触发时开始执行
4. 提前下载分包所需的资源文件，如 JS、CSS、图片等

##### 实现步骤

**1. 识别关键页面**
- 确定用户经常访问的重要页面
- 这些页面通常被定义为关键页面
- 在这些页面触发时启动预下载机制

**2. 触发预下载**
- 在合适的时机（如用户打开小程序或进入关键页面时）
- 调用相应的预下载函数或方法
- 开始下载分包资源

**3. 资源加载**
- 下载完成后，将分包资源缓存至本地
- 用户访问分包页面时直接使用已下载的资源
- 避免重新下载

##### 配置方式

在 `app.json` 中通过 `preloadRule` 配置预下载规则：

```json
{
  "pages": ["pages/index"],
  "subpackages": [
    {
      "root": "important",
      "pages": ["index"]
    },
    {
      "root": "sub1",
      "pages": ["index"]
    },
    {
      "name": "hello",
      "root": "path/to",
      "pages": ["index"]
    },
    {
      "root": "sub3",
      "pages": ["index"]
    },
    {
      "root": "indep",
      "pages": ["index"],
      "independent": true
    }
  ],
  "preloadRule": {
    "pages/index": {
      "network": "all",
      "packages": ["important"]
    },
    "sub1/index": {
      "packages": ["hello", "sub3"]
    },
    "sub3/index": {
      "packages": ["path/to"]
    },
    "indep/index": {
      "packages": ["__APP__"]
    }
  }
}
```

**配置说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| packages | StringArray | 是 | 进入页面后预下载分包的 root 或 name |
| network | String | 否 | 在指定网络下预下载，可选值：all（不限网络）、wifi（仅wifi下） |

**特殊值：**
- `"__APP__"`: 表示主包
- 用于独立分包中预下载主包

##### 限制与注意事项

1. **同一个分包中的页面不能互相预下载**
2. **预下载时机**：在进入某个页面时触发
3. **网络控制**：可以限制仅在 wifi 下预下载，避免消耗用户流量
4. **容量限制**：注意控制预下载的分包总大小

##### 实际项目场景

**1. 图片密集页面**
```json
{
  "preloadRule": {
    "pages/index/index": {
      "network": "wifi",
      "packages": ["gallery", "product-detail"]
    }
  }
}
```
- 适用场景：相册、商品展示页
- 通过预下载图片资源提高用户体验
- 建议仅在 wifi 下预下载，避免消耗流量

**2. 复杂交互页面**
```json
{
  "preloadRule": {
    "pages/home/home": {
      "network": "all",
      "packages": ["map-module", "video-player"]
    }
  }
}
```
- 适用场景：地图、视频播放页
- 提前加载相关组件和数据
- 加快页面展示速度

**3. 业务流程优化**
```json
{
  "preloadRule": {
    "pages/login/login": {
      "network": "all",
      "packages": ["user-center", "order-list"]
    }
  }
}
```
- 用户登录后可能访问的页面
- 提前预下载，提升后续页面打开速度

##### 最佳实践

1. **分析用户路径**：通过数据分析确定常见的页面跳转路径
2. **合理控制预下载范围**：不要一次预下载过多分包
3. **网络策略**：大分包建议设置为 `"network": "wifi"`
4. **配合数据预拉取**：预下载分包的同时，可以预拉取对应的数据
5. **监控效果**：通过性能数据监控预下载的效果

#### 3.3.5 资源加载优化

**1) 图片优化**
- 使用合适尺寸和格式（WebP）
- 长列表图片懒加载
- 使用 `<image>` 的 `lazy-load` 属性

**2) 字体图标**
- 使用 iconfont 代替图片
- 按需加载字体文件

#### 3.3.6 内存优化

**1) 合理分包**
- 既能减少耗时，也能降低内存占用

**2) 及时清理**
```javascript
onUnload() {
  // 清理定时器
  clearInterval(this.timer)
  
  // 解绑事件监听
  this.observer && this.observer.disconnect()
}
```

### 3.4 其他优化配置

#### 初始渲染缓存
```json
{
  "initialRenderingCache": "static"
}
```

#### 占位组件（Component Placeholder） ⭐⭐

##### 概念与作用

**占位组件**是微信小程序提供的一种优化机制，用于实现自定义组件的**用时注入**（延迟加载）。

**核心作用：**
1. **减少代码注入量**：组件不会在页面初始化时立即注入
2. **提升启动性能**：减少小程序启动时的代码执行量
3. **按需加载**：只有在组件真正需要渲染时才注入代码

##### 工作原理

**流程：**
1. 页面初始化时，先渲染占位组件（通常是简单的 view 或其他轻量组件）
2. 当真正需要使用该组件时，才注入组件的完整代码
3. 替换占位组件，渲染真实组件

**与按需注入的关系：**
- 按需注入：避免注入未访问页面的代码
- 占位组件：进一步优化，连当前页面的部分组件也延迟注入

##### 配置方式

**1. 在页面或组件的 JSON 配置中声明**

```json
{
  "usingComponents": {
    "heavy-chart": "/components/chart/chart",
    "complex-list": "/components/list/list",
    "simple-button": "/components/button/button"
  },
  "componentPlaceholder": {
    "heavy-chart": "view",
    "complex-list": "view"
  }
}
```

**配置说明：**
- `usingComponents`：声明要使用的组件
- `componentPlaceholder`：为指定组件配置占位组件
- 键：组件名称
- 值：占位组件标签（如 `view`、`text` 等）

**2. 在组件内部使用**

```html
<!-- page.wxml -->
<view>
  <heavy-chart data="{{chartData}}" />
  <complex-list items="{{listData}}" />
  <simple-button />
</view>
```

初始渲染时：
```html
<!-- 实际渲染 -->
<view>
  <view />  <!-- heavy-chart 的占位 -->
  <view />  <!-- complex-list 的占位 -->
  <simple-button />  <!-- 没有配置占位，正常渲染 -->
</view>
```

##### 适用场景

**适合使用占位组件的情况：**

1. **复杂的图表组件**
```json
{
  "componentPlaceholder": {
    "echarts": "view"
  }
}
```
- 图表组件通常代码量大
- 可能依赖较大的第三方库
- 不一定立即需要展示

2. **富文本编辑器**
```json
{
  "componentPlaceholder": {
    "rich-editor": "view"
  }
}
```
- 功能复杂，代码量大
- 可能在用户交互后才显示

3. **复杂的表单组件**
```json
{
  "componentPlaceholder": {
    "complex-form": "view"
  }
}
```
- 包含大量表单验证逻辑
- 可能在特定条件下才渲染

4. **条件渲染的组件**
```json
{
  "componentPlaceholder": {
    "user-vip-panel": "view"
  }
}
```
- 根据用户状态决定是否显示
- 非所有用户都会看到

**不适合的情况：**
- 首屏必须展示的组件
- 代码量很小的简单组件
- 频繁使用的基础组件

##### 完整示例

**页面配置（page.json）：**
```json
{
  "usingComponents": {
    "stock-chart": "/components/stock-chart/index",
    "data-table": "/components/data-table/index",
    "user-info": "/components/user-info/index"
  },
  "componentPlaceholder": {
    "stock-chart": "view",
    "data-table": "view"
  }
}
```

**页面代码（page.wxml）：**
```html
<view class="container">
  <!-- 立即渲染 -->
  <user-info user="{{userInfo}}" />
  
  <!-- 延迟渲染 - 使用占位 -->
  <stock-chart wx:if="{{showChart}}" data="{{chartData}}" />
  <data-table items="{{tableData}}" />
</view>
```

**页面逻辑（page.js）：**
```javascript
Page({
  data: {
    userInfo: {},
    showChart: false,
    chartData: [],
    tableData: []
  },
  
  onLoad() {
    // 首屏只加载用户信息
    this.loadUserInfo()
  },
  
  onReady() {
    // 首屏渲染完成后，延迟加载图表和表格
    setTimeout(() => {
      this.loadChartData()
      this.loadTableData()
    }, 500)
  },
  
  loadUserInfo() {
    // 加载用户信息（立即需要）
  },
  
  loadChartData() {
    // 此时才会真正注入 stock-chart 组件代码
    this.setData({ 
      showChart: true,
      chartData: [/* ... */]
    })
  },
  
  loadTableData() {
    // 此时才会真正注入 data-table 组件代码
    this.setData({ 
      tableData: [/* ... */]
    })
  }
})
```

##### 与用时注入的配合使用

```json
// app.json
{
  "lazyCodeLoading": "requiredComponents"
}

// page.json
{
  "usingComponents": {
    "heavy-component": "/components/heavy/heavy"
  },
  "componentPlaceholder": {
    "heavy-component": "view"
  }
}
```

**效果叠加：**
1. `lazyCodeLoading`：页面级别的按需注入
2. `componentPlaceholder`：组件级别的延迟注入
3. 两者结合，实现更精细的代码注入控制

##### 注意事项

1. **占位组件样式**
   - 占位组件会继承真实组件的 class 和 style
   - 可以为占位组件设置骨架屏样式
   
```html
<heavy-chart class="chart-placeholder" />
```

```css
.chart-placeholder {
  height: 300px;
  background: #f5f5f5;
}
```

2. **组件替换时机**
   - 当组件的 data 或 properties 发生变化时
   - 组件被条件渲染（wx:if）时
   - 需要确保有触发条件

3. **性能收益评估**
   - 只对代码量大的组件使用
   - 通过开发者工具的性能面板评估效果

4. **兼容性**
   - 基础库 2.11.1 开始支持
   - 低版本会忽略配置，正常渲染组件

##### 最佳实践

1. **优先级排序**
   - P0（首屏必需）：不使用占位
   - P1（首屏可延迟）：使用占位 + 延迟 300-500ms
   - P2（非首屏）：使用占位 + 按需加载

2. **配合骨架屏**
```css
/* 为占位组件设计骨架屏样式 */
.chart-placeholder {
  height: 300px;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

3. **监控注入时机**
```javascript
Component({
  lifetimes: {
    attached() {
      console.log('组件真正注入并渲染')
    }
  }
})
```

##### 性能对比

**使用占位组件前：**
- 代码注入时间：800ms
- 首屏渲染时间：1200ms

**使用占位组件后：**
- 代码注入时间：500ms（减少 37.5%）
- 首屏渲染时间：800ms（减少 33%）
- 占位组件渲染：100ms
- 真实组件延迟注入：300ms（在用户操作时）

##### 官方文档

- [占位组件 | 微信开放文档](https://developers.weixin.qq.com/miniprogram/dev/framework/custom-component/placeholder.html)

#### 小程序更新机制
```javascript
const updateManager = wx.getUpdateManager()

updateManager.onUpdateReady(() => {
  wx.showModal({
    title: '更新提示',
    content: '新版本已准备好，是否重启应用？',
    success: (res) => {
      if (res.confirm) {
        updateManager.applyUpdate()
      }
    }
  })
})
```

---

## 四、小程序为什么快？

相比普通 H5，小程序有以下优势：

1. **双线程架构**：渲染层和逻辑层并行不阻塞
2. **多个 WebView**：页面切换更流畅，体验接近原生
3. **WebView 预加载**：PageFrame 模板机制快速创建页面
4. **安装包缓存**：本地缓存 + 增量更新
5. **环境预加载**：根据场景提前准备运行环境
6. **原生组件**：关键组件由客户端渲染，性能更好
7. **微信优化**：大量底层优化（QUIC、HTTP/2、预连接等）

---

## 五、开发实践建议

### 5.1 架构设计
- ✅ 合理使用分包，首包体积控制在 2MB 以内
- ✅ 避免路由嵌套超过 10 层
- ✅ 重要页面使用 TabBar

### 5.2 代码规范
- ✅ 按需注入 + 用时注入
- ✅ setData 控制频率、范围、内容
- ✅ 避免 data 层级过深
- ✅ 及时清理定时器和事件监听

### 5.3 资源管理
- ✅ 图片、字体等静态资源使用 CDN
- ✅ 图片懒加载 + 合适格式
- ✅ 避免 base64 内联大图

### 5.4 性能监控
- ✅ 使用微信开发者工具的性能面板
- ✅ 监控启动耗时、渲染耗时
- ✅ 使用 `wx.reportPerformance` 上报性能数据

---

## 参考资料

- [微信小程序官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/)
- [小程序性能优化指南](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips.html)
- [微信小程序底层框架实现原理](https://juejin.cn/book/6982013809212784676)
- [冷启动太慢？4个方法亲测有效](https://mp.weixin.qq.com/s/CfgbwfYYGVyevuE6Va0C2w)

---

## 总结

微信小程序通过 **双线程架构、WebView 预加载、Virtual DOM、组件化框架** 等设计，在保证安全可控的前提下，实现了接近原生 App 的性能和体验。

开发者需要理解小程序的底层原理，合理使用 **分包、按需注入、数据预拉取、渲染优化** 等手段，才能充分发挥小程序的性能优势，为用户提供流畅的使用体验。

