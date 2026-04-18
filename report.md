# 微前端架构分析报告

## 项目结构概览

```
glm/
├── test/home.html          # 子页面
├── test/home.css           # 子页面样式
├── web/base/index.html     # 母页面
└── web/base/page/index.js  # 核心组件代码（webpack打包）
```

---

## 问题一：子页面HTML/CSS如何不通过iframe引入

### 入口函数

**入口函数**: `SandboxLoad` ([index.js:1486](web/base/page/index.js#L1486))

**调用链路**:
```
_BoxPage (React组件)
  └── _BoxPageUI (UI渲染)
        └── SandboxUpdate (更新沙箱)
              └── SandboxLoad (加载子页面)
```

### 工作原理

**核心技术**: Shadow DOM

**实现步骤**:

1. **创建Shadow DOM容器**
   ```javascript
   // index.js:1710
   if (!element.current.shadowRoot) {
     element.current.attachShadow({ mode: 'open' });
   }
   ```
   - 在宿主div元素上创建open模式的Shadow DOM
   - Shadow DOM提供样式隔离，子页面CSS不会污染母页面

2. **fetch获取子页面HTML**
   ```javascript
   // index.js:1490-1493
   fetch(src).then(response => response.text())
     .then(htmlContent => {
       const parser = new DOMParser();
       const page = parser.parseFromString(htmlContent, 'text/html');
     });
   ```

3. **CSS注入到Shadow DOM**
   ```javascript
   // index.js:1510-1517
   const cssLinks = page.head.querySelectorAll('link[rel="stylesheet"]');
   cssLinks.forEach(link => {
     element.current.shadowRoot.appendChild(link);
   });
   ```
   - 提取子页面的`<link rel="stylesheet">`标签
   - 直接appendChild到shadowRoot中
   - 支持excludeFile配置排除特定CSS（如主题文件）

4. **HTML Body注入到Shadow DOM**
   ```javascript
   // index.js:1531
   element.current.shadowRoot.appendChild(page.body.children[0]);
   ```

**关键点**:
- Shadow DOM实现了CSS样式隔离
- 子页面CSS使用`:host()`选择器适配主题
- 示例: `:host(div[trick-theme="dark"])` ([home.css:9](test/home.css#L9))

---

## 问题二：JS沙箱隔离机制

### 入口函数

**入口函数**: `SandboxLoad` ([index.js:1486](web/base/page/index.js#L1486)) 中的JS处理部分

**调用链路**:
```
SandboxLoad
  └── 创建隐藏iframe (index.js:1767)
        └── 创建Proxy代理对象
              ├── windowProxy (index.js:1565)
              └── documentProxy (index.js:1626)
```

### 沙箱架构

**双重隔离机制**:

```
┌─────────────────────────────────────────────────────────┐
│  母页面 (主窗口)                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Shadow DOM (HTML/CSS隔离)                         │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  隐藏iframe (JS隔离)                         │  │  │
│  │  │  - 子页面JS在此执行                          │  │  │
│  │  │  - 通过Proxy访问母页面window/document        │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 具体实现

**1. 隐藏iframe创建**
```javascript
// index.js:1767
<iframe src="about:blank" className="hidden" />
```

**2. JS脚本注入到iframe**
```javascript
// index.js:1554-1558
let scriptsContent = '<head>';
scriptsContent += `<script>var trickAPP = parent["Trick-Comp-_BoxPage"]["${innerLoad.id}"];</script>`;
scripts.forEach(script => {
  scriptsContent += script.outerHTML;
});
iframe.srcdoc = scriptsContent;
```

**3. Proxy代理 - windowProxy**
```javascript
// index.js:1565-1621
const windowProxy = new Proxy(window, {
  get(origin, key, receiver) {
    // 事件监听器劫持 - 自动添加AbortController
    if (key === "addEventListener") { /* ... */ }
    
    // 使用沙箱window的场景
    if (key === "PubSub" || innerLoad.windowSet.includes(key) || !origin.hasOwnProperty(key)) {
      return windowShadow[key]; // iframe的window
    }
    
    // 默认使用宿主window
    return origin[key];
  },
  set(origin, key, value) {
    // 变量设置存入沙箱
    innerLoad.windowSet.push(key);
    windowShadow[key] = value;
    return true;
  }
});
```

**4. Proxy代理 - documentProxy**
```javascript
// index.js:1626-1676
const documentProxy = new Proxy(document, {
  get(origin, key, receiver) {
    // DOM操作重定向到Shadow DOM
    if ((key.startsWith("get") || key.startsWith("create") || key.startsWith("query"))) {
      return documentShadow[key]; // shadowRoot
    }
    
    // body/head/all 返回Shadow DOM
    if (key === "body" || key === "head" || key === "all") {
      return documentShadow;
    }
    
    // defaultView返回windowProxy
    if (key === "defaultView") return windowProxy;
    
    return origin[key];
  }
});
```

**5. trickAPP注入**
```javascript
// index.js:1678-1689
topWindow["Trick-Comp-_BoxPage"][innerLoad.id] = {
  "documentProxy": documentProxy,
  "windowProxy": windowProxy,
  "HTMLDocument": HTMLDocument,
  "HTMLElement": HTMLElement,
  // ... 其他DOM构造函数
};
```

**子页面使用方式** ([home.html:29-40](test/home.html#L29)):
```javascript
(function(document, window, root, ...) {
  // 这里的document/window是Proxy代理对象
  window.TrickAPPRoot = createRoot(document.getElementById('id_body'));
})(trickAPP.documentProxy, trickAPP.windowProxy, ...);
```

---

## 问题三：沙箱通信机制

### 入口函数

**入口函数**: `_BoxPageAction` ([index.js:1405](web/base/page/index.js#L1405))

**调用链路**:
```
_BoxPage (React组件)
  └── PubSubListen (订阅消息) - index.js:1826
        └── _BoxPageAction (处理消息)
              └── CompActSet/CompActGet/CompActEvent
```

### 通信架构

**核心库**: PubSubJS ([index.js:4577](web/base/page/index.js#L4577))

```
┌──────────────────┐      PubSub.publish       ┌──────────────────┐
│  母页面组件       │ ───────────────────────► │  订阅者          │
│  PubSubSend()    │                          │  PubSubListen()  │
└──────────────────┘ ◄─────────────────────── └──────────────────┘
                           PubSub.publish
```

### 通信实现

**1. 消息订阅（母页面）**
```javascript
// index.js:1825-1826
const _id = Tools.ParamRead("_id", "", config);
channelToken.current = Tools.PubSubListen(_id, _BoxPageAction, handler);
```

**2. 消息发送（工具函数）**
```javascript
// common.js - PubSubSend
PubSubSend(channel, data) {
  pubsub.publish(channel, data);
}
```

**3. Proxy中的通信透传**
```javascript
// index.js:1590
// windowProxy.get() 中
if (key === "PubSub" || ...) {
  return windowShadow[key]; // 子页面可访问PubSub
}
```

**4. 生命周期管理**
```javascript
// index.js:1832-1834
return () => {
  channelToken.current = Tools.PubSubCancel(channelToken.current);
};
```

### 通信特点

| 特性 | 说明 |
|-----|------|
| 发布订阅模式 | 基于PubSubJS库 |
| 频道隔离 | 通过`_id`区分不同组件实例 |
| 双向通信 | 母页面↔子页面均可发送/接收 |
| 生命周期绑定 | 组件销毁时自动取消订阅 |

---

## 总结

### 技术栈

| 层面 | 技术 | 用途 |
|-----|------|------|
| HTML/CSS隔离 | Shadow DOM | 样式隔离，DOM隔离 |
| JS隔离 | iframe + Proxy | JavaScript执行环境隔离 |
| 通信机制 | PubSubJS | 发布订阅模式的消息通信 |
| UI框架 | React | 组件化开发 |

### 架构优势

1. **样式隔离**: Shadow DOM天然隔离CSS，无需额外处理
2. **JS隔离**: iframe提供原生隔离，Proxy实现可控访问
3. **通信灵活**: PubSub模式解耦组件间通信
4. **生命周期管理**: AbortController自动清理事件监听

### 关键文件索引

| 文件 | 说明 |
|-----|------|
| [test/home.html](test/home.html) | 子页面示例 |
| [test/home.css](test/home.css) | 子页面样式（含Shadow DOM适配） |
| [web/base/index.html](web/base/index.html) | 母页面入口 |
| [web/base/page/index.js](web/base/page/index.js) | 核心组件代码 |
| [web/base/page/common/common.js](web/base/page/common/common.js) | 工具函数库 |
