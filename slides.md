---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: 前端监控演讲
info: |
  ## 前端监控演讲
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# https://sli.dev/guide/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/guide/syntax#mdc-syntax
mdc: true
---

# 前端稳定性监控

演讲者：周传森

<div class="abs-br m-6 flex items-center gap-2">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
     <carbon:arrow-right class="inline"/>
  </span>
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# 为什么需要前端监控

前端监控主要用于跟踪和理解用户在使用应用时的体验和问题
<v-click>

- 📝 **提升用户体验** - 可通过监控用户交互、页面渲染时间、页面加载速度等，来发现和优化瓶颈，进一步提升用户体验。

</v-click>
<v-click>

- 🛠 **故障排查** - 前端监控可以帮助我们实时收集和上报网页错误、性能问题等，当问题发生时可以尽快发现和定位问题，降低故障恢复时间。

</v-click>
<v-click>

- 🧑‍💻 **用户行为分析** - 通过前端监控，我们可以收集用户的行为数据，如点击事件、页面停留时间、路径分析等，帮助我们更好地理解用户使用习惯，优化产品设计。

</v-click>
<v-click>

- 🤹 **业务数据监控** - 可以监控关键业务数据的变化，例如购物车转化率、注册量等，及时发现潜在问题。

</v-click>

  <br>
  <br>

<!--
You can have `style` tag in markdown to override the style for the current page.
Learn more: https://sli.dev/guide/syntax#embedded-styles
-->

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---

# 为什么不用第三方监控

<div
  v-motion
  :initial="{ x: -80, opacity: 0}"
  :enter="{ x: 10, opacity: 1, scale: 1, transition: { delay: 100, duration: 1300 } }"
>

- 方便团队做自定义的UV用户识别，比如通过登录账号ID或者通过设备信息；甚至从设备信息转入登录态后的继承
- 方便接入自己团队的各种告警业务等
- 方便做各维度数据的联合分析，比如发生错误可以联动查询用户行为追溯数据等
- 方便做业务需求上的拓展，比如自定义埋点、特殊的数据分析维度
- 方便前后端全链路的一个API请求链路分析；
- 私有化部署需要付费且价格不低，不易定制化。

</div>

  <br>
  <br>

---

# 前端监控做了那些

前端搭建监控体系，可以概括为为了做两件事：

<div
  v-motion
  :initial="{ x: -80, opacity: 0}"
  :enter="{ x: 10, opacity: 1, scale: 1, transition: { delay: 100, duration: 1300 } }"
>

- 如何及时发现问题
- 如何快速定位问题

</div>

  <br>
  <br>

<v-click>

### 可以拆分为

- **页面的性能情况** - <span v-mark.red="2"><code>如何监控页面的性能情况</code> </span>
- **用户的行为情况** - <span v-mark.red="3"><code>包括PV、UV、访问来路，路由跳转等</code> </span>
- **接口的调用情况** - <span v-mark.red="4"><code>通过http访问的外部接口的成功率、耗时情况等 </code> </span>
- **页面的稳定情况** - <span v-mark.red="5"><code>各种前端异常等</code> </span>
- **数据上报及优化** - <span v-mark.red="6"><code>如何将监控捕获到的数据优雅的上报</code> </span>

</v-click>

---

# 前端监控系统目录

<Toc  minDepth="1" maxDepth="2"></Toc>

---

# 错误监控

<div
  v-motion
  :initial="{ x: -80, opacity: 0}"
  :enter="{ x: 10, opacity: 1, scale: 1, transition: { delay: 100, duration: 1300 } }"
>

- <span v-mark.circle.orange="1"> JS运行异常 </span>
- 静态资源加载异常
- Promise异常
- HTTP请求异常
- 跨域脚本错误
- Vue2、Vue3 错误捕获

</div>

<br>

<Error></Error>

<v-click >

<div class="bg-white absolute top-105px right-200px w-120  p-5px">

```ts twoslash
const handler = (event: ErrorEvent) => {
  // 阻止向上抛出控制台报错
  event.preventDefault();
  // 处理错误
  console.log(event);
  // HandleEvents.handleError(event);
};

window.addEventListener('error', (event) => handler(event), true);

function throwError() {
  setTimeout(() => {
    throw new Error('error');
  }, 1000);
}

throwError();
```

</div>

</v-click>

<v-click >
  <br>
  <br>
  <br>
  <br>
  <br>
  <br>

### <span v-mark.red="2">以及最后sourceMap上传到私服的过程</span>

</v-click>

---

## 整体错误上报流程

<v-drag pos="666,90,307,324" class="w-100 bg-white p-5px">
  <img
    class="w-full"
    src="/public/all-input.png"
    alt=""
  />
</v-drag>

以Js运行异常为例，整体上报流程如下：

<v-click>

````md magic-move {lines: true}
```ts {*|5}
// setupReplace 中添加 addReplaceHandler
// 替代处理的回调函数和类型
addReplaceHandler({
  callback: (error) => {
    HandleEvents.handleError(error);
  },
  type: EventTypes.ERROR,
});
```

```ts {*|1,3}
// addReplaceHandler 添加订阅和函数替代
export function addReplaceHandler(handler: ReplaceHandler) {
  if (!subscribeEvent(handler)) return; // 添加订阅
  replace(handler.type as EventTypes); // 替换函数
}
```

```ts {1,5-7}
// subscribeEvent 添加订阅，存在handlers订阅中心
export function subscribeEvent(handler: ReplaceHandler): boolean {
  if (!handler || getFlag(handler.type)) return false;
  setFlag(handler.type, true);
  handlers[handler.type] = handlers[handler.type] || [];
  handlers[handler.type].push(handler.callback);
  return true;
}
```

```ts {1,4}
// addReplaceHandler 添加订阅和函数替代
export function addReplaceHandler(handler: ReplaceHandler) {
  if (!subscribeEvent(handler)) return; // 添加订阅
  replace(handler.type as EventTypes); // 替换函数
}
```

```ts {1,7}
// replace 使用策略模式 匹配 EventTypes.ERROR 并使用 listenError 函数
function listenError(): void {
  on(
    _global,
    'error',
    function (e: ErrorEvent) {
      triggerHandlers(EventTypes.ERROR, e);
    },
    true
  );
}
```

```ts {1,4-13|7}
// replace 使用策略模式 匹配 EventTypes.ERROR 并使用 listenError 函数
export function triggerHandlers(type: EventTypes | WxEvents, data: any): void {
  if (!type || !handlers[type]) return;
  handlers[type].forEach((callback) => {
    nativeTryCatch(
      () => {
        callback(data);
      },
      (e: Error) => {
        console.error(`重写事件triggerHandlers的回调函数发生错误：${e}`);
      }
    );
  });
}
```

```ts {5}
// setupReplace 中添加 addReplaceHandler
// 替代处理的回调函数和类型
addReplaceHandler({
  callback: (error) => {
    HandleEvents.handleError(error);
  },
  type: EventTypes.ERROR,
});
```

```ts {1,5|9|11-16|17}
// HandleEvents 是个map对象，这里就对应handleError函数
  handleError(errorEvent: ErrorEvent) {
    const target = errorEvent.target as ResourceErrorTarget;
    // code error
    const { message, filename, lineno, colno, error } = errorEvent;
    let result: ReportDataType;

    // 处理SyntaxError，stack没有lineno、colno
    result || (result = transformData(message, filename, lineno, colno));
    result.type = ErrorTypes.JAVASCRIPT_ERROR;
    breadcrumb.push({
      type: BreadCrumbTypes.CODE_ERROR,
      category: breadcrumb.getCategory(BreadCrumbTypes.CODE_ERROR),
      data: { ...result },
      level: Severity.Error,
    });
    transportData.send(result);
  },
```
````

</v-click>

<v-click>

后面就是生成 <span v-mark.red="15"><code>errorId</code> </span> 和 <span v-mark.circle.orange="16">上报方式</span> 的流程

</v-click>

---

## TransportData上报流程

<v-drag pos="608,90,365,324" class="w-100 bg-white p-5px">
  <img
    class="w-full"
    src="/public/all-input.png"
    alt=""
  />
</v-drag>

<v-click>

````md magic-move {lines: true}
```ts {1,10|11|3-9|16|17-18}
// TransportData 类
export class TransportData {
  async beforePost(data: FinalReportType) {
    // 生成errorId
    const errorId = createErrorId(data, this.apikey);
    if (!errorId) return false;
    data.errorId = errorId;
    return data;
  }

  async send(data: FinalReportType) {
    const result = await this.beforePost(data);
    if (!result) return;

    return typeof navigator.sendBeacon === 'function'
      ? this.beaconTransport(result, dsn)
      : this.xhrPost(result, dsn);
  }
}
```

```ts {1,4|5,6|9,10}
// beacon 形式上报
beaconTransport = (data: any, url: string): Function => {
  const requestFun = () => {
    const status = window.navigator.sendBeacon(url, JSON.stringify(data));
    // 如果数据量过大，则本次大数据量用 XMLHttpRequest 上报
    if (!status) this.xhrPost().apply(this, data, url);
  };
  // this.queue.addFn(requestFun);
  requestFun;
};
```

```ts {1,4-8}
// XMLHttpRequest 形式上报
async xhrPost(data: any, url: string) {
  const requestFun = (): void => {
    const xhr = new XMLHttpRequest();
    xhr.open(EMethods.Post, url);
    xhr.setRequestHeader('Content-Type', 'application/json;charset=UTF-8');
    xhr.withCredentials = true;
    xhr.send(JSON.stringify(data));
  };
  // this.queue.addFn(requestFun);
  requestFun;
}
```
````

</v-click>

<v-click>

这就是使用 <span v-mark.red="10"><code>addEventListener('error')</code> </span> 进行监听的全部流程

</v-click>

---

## sourceMap上传流程

自定义vite-plugin-sourcemap-xk插件，将sourceMap上传到私服

<v-click>

````md magic-move {lines: true}
```ts {*|10-12|13-15}
import path from 'path';
import fs from 'fs';
import request from 'request';

const TAG = '[vite-plugin-sourcemap-xk]: ';

export default function vitePluginSourcemapXk(pluginOptions) {
  return {
    name: 'sourcemap-xk',
    writeBundle(options: any, bundle: any) {
      // ...获取sourcemap文件
    },
    async closeBundle() {
      // ...上传sourcemap
    },
  };
}
```

```ts
writeBundle(options: any, bundle: any) {
  const { dir } = options;
  outDir = dir;
  const fileNames = Object.keys(bundle);
  sourcemapFiles = fileNames.filter((fileName) =>
    fileName.endsWith('.js.map')
  );
  return;
},
```

```ts {*|2-11|4}
async closeBundle() {
  for (const file of sourcemapFiles) {
    try {
      const { code, msg } = await upload(`${outDir}/${file}`);
      if (code !== 0) {
        console.error(TAG, msg);
      }
    } catch (error) {
      console.error(TAG, error);
    }
  }
  console.log(TAG, 'upload finished');
  return;
},
```

```ts
function upload(filePath: string): Promise<ResponseType> {
  return new Promise((resolve, rejected) => {
    const fileStream = fs.createReadStream(filePath);
    const filename = path.basename(filePath);
    const config = {
      method: 'POST',
      url,
      formData: {
        file: {
          value: fileStream,
          options: {
            filename: filePath,
            contentType: null,
          },
        },
        dirname: appname,
        filename,
      },
    };

    request(config, function (err, res) {});
  });
}
```

```ts
import path from 'path';
import fs from 'fs';
import request from 'request';

const TAG = '[vite-plugin-sourcemap-xk]: ';

export default function vitePluginSourcemapXk(pluginOptions) {
  return {
    name: 'sourcemap-xk',
    writeBundle(options: any, bundle: any) {
      // ...获取sourcemap文件
    },
    async closeBundle() {
      // ...上传sourcemap
    },
  };
}
```
````

</v-click>

<v-click>

这就是使用 <span v-mark.red="10"><code>sourceMap</code> </span> 上传流程

</v-click>

---

# 性能监控

- BBC页⾯加载时长每增加1秒，⽤户流失10%
- Pinterest减少页⾯加载时长40%, 提⾼了搜索和注册数15%
- DoubleClick发现如果移动⽹站加载时长超过3秒，53%的⽤户会放弃
  <br>
  <br>

### 网页性能指标及影响因素

1. performance.timing
2. performance.getEntries()
3. PerformanceObserver

```ts {2|4-6} twoslash
//直接往 PerformanceObserver() 入参匿名回调函数，成功 new 了一个 PerformanceObserver 类的，名为 observer 的对象
var observer = new PerformanceObserver(function (list, obj) {
  var entries = list.getEntries();
  for (var i = 0; i < entries.length; i++) {
    //处理“navigation”和“resource”事件
  }
});
//调用 observer 对象的 observe() 方法
observer.observe({ entryTypes: ['navigation', 'resource'] });
```

<img
  v-click
  class="absolute bottom-205px right-200px w-100"
  src="/public/timing.png"
  alt=""
/>

---

## Core Web Vitals（核心网页指标）

[Core Web Vitals](https://web.dev/articles/vitals?hl=zh-cn/)

<img
  v-click
  class="absolute bottom-205px right-100px w-100"
  src="/public/web-core-user.png"
  alt=""
/>

<!--   v-drag="'square'" -->

<img
  v-click
  class="absolute bottom-255px left-50px w-80"
  src="/public/web-core.png"
  alt=""
/>

<div v-click class="mt-200px">

## 性能监控工具

- [Lighthouse](https://developers.google.com/web/tools/lighthouse)
- [WebPageTest](https://www.webpagetest.org/)
- [GTmetrix](https://gtmetrix.com/)
- [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/)
- [Calibre](https://calibreapp.com/)

 </div>

---

# 行为监控

<div
  v-motion
  :initial="{ x: -80, opacity: 0}"
  :enter="{ x: 10, opacity: 1, scale: 1, transition: { delay: 100, duration: 1300 } }"
>

- PV、UV
- 路由跳转
  - <span v-mark.red="1"><code>Hash 路由</code> </span>
  - <span v-mark.red="1"><code>History 路由</code> </span>
- 用户点击事件
- 用户自定义埋点
- HTTP 请求捕获
- 页面停留时间
- 访客来路
- User Agent 解析

</div>

  <br>

<div class="bg-white absolute top-105px right-200px w-100 p-5px">
  <img
    class="w-full"
    src="/public/behavior.png"
    alt=""
  />

</div>

---

<div
  class="flex items-center h-full"
  v-motion
  :initial="{ x: -80, opacity: 0}"
  :enter="{ x: 300, opacity: 1, scale: 1, transition: { delay: 100, duration: 1300 } }"
>
  <div class="text-3xl">Thanks for Listening!</div>
</div>
