# 微前端架构分析报告

## 项目概述

本项目是一个基于 **Web Components + Shadow DOM + iframe 沙箱** 的微前端实现方案。

- **母页面（上层页面）**: `web/base/index.html`
- **子页面**: `test/home.html`
- **核心技术栈**: React 18、Shadow DOM、Proxy API、iframe sandbox

---

## 问题一：子页面 HTML/CSS 如何不通过 iframe 引入

### 核心机制：Shadow DOM

子页面的 HTML/CSS **并非通过 iframe 加载**，而是使用 **Shadow DOM** 技术直接嵌入到母页面的 DOM 树中。

### 入口函数与调用链路

```
入口函数: SandboxCreate() 
  ↓
调用: element.current.attachShadow({ mode: 'open' })
  ↓
调用: SandboxLoad()
  ↓
调用: fetch(src) 获取子页面 HTML
  ↓
调用: DOMParser().parseFromString() 解析 HTML
  ↓
调用: element.current.shadowRoot.appendChild() 插入 Shadow DOM
```

### 详细工作流程

**文件位置**: `web/base/page/index.js` 中的 `SandboxLoad` 函数 (约第 1500 行)

```javascript
// STEP 1: 获取子页面 HTML 内容
fetch(src).then(response => response.text())
  .then(htmlContent => {
    // STEP 2: 使用 DOMParser 解析 HTML
    const parser = new DOMParser();
    const page = parser.parseFromString(htmlContent, 'text/html');
    
    // STEP 3: 提取并插入 CSS
    const cssLinks = page.head.querySelectorAll('link[rel="stylesheet"]');
    cssLinks.forEach(link => {
      element.current.shadowRoot.appendChild(link);
    });
    
    // STEP 4: 插入 body 内容到 Shadow DOM
    element.current.shadowRoot.appendChild(page.body.children[0]);
  });
```

### 关键点

| 特性 | 说明 |
|------|------|
| **Shadow DOM** | 创建隔离的 DOM 子树，样式完全隔离 |
| **DOMParser** | 将 HTML 字符串解析为可操作的 DOM 对象 |
| **样式隔离** | 子页面 CSS 只在 Shadow DOM 内生效，不影响母页面 |
| **无 iframe** | HTML/CSS 渲染完全不依赖 iframe |

### 样式隔离原理

```css
/* 子页面样式: test/home.css */
/* 使用 :host 选择器适配 Shadow DOM */
:host(div[trick-theme="dark"]){
    .Page-Body {
        color: red !important;
    }
}
```

---

## 问题二：子页面的 JS 沙箱隔离机制

### 核心机制：iframe + Proxy 代理

子页面的 JavaScript 运行在 **iframe 沙箱** 中，通过 **Proxy API** 实现与母页面的受控通信。

### 入口函数与调用链路

```
入口函数: SandboxLoad() 
  ↓
调用: iframe.srcdoc = scriptsContent 注入 JS
  ↓
创建: windowProxy (Proxy 对象)
  ↓
创建: documentProxy (Proxy 对象)
  ↓
挂载: parent["Trick-Comp-_BoxPage"][id] = proxies
  ↓
子页面通过 trickAPP 变量访问代理对象
```

### 沙箱架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        母页面 (Host)                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Shadow DOM (子页面 HTML/CSS)              │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              iframe (子页面 JS 沙箱)              │  │  │
│  │  │                                                 │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │         Proxy 代理层 (windowProxy)         │  │  │  │
│  │  │  │  ┌─────────────────────────────────────┐  │  │  │  │
│  │  │  │  │      子页面 JS (React App)           │  │  │  │  │
│  │  │  │  │                                     │  │  │  │  │
│  │  │  │  │  trickAPP.documentProxy             │  │  │  │  │
│  │  │  │  │  trickAPP.windowProxy               │  │  │  │  │
│  │  │  │  │  trickAPP.rootProxy                 │  │  │  │  │
│  │  │  │  └─────────────────────────────────────┘  │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 沙箱核心实现

**文件位置**: `web/base/page/index.js` 中的 `SandboxLoad` 函数 (约第 1550-1700 行)

#### 1. iframe 创建与 JS 注入

```javascript
// 创建 iframe 作为 JS 执行容器
const iframe = element.current.querySelector('iframe');

// 构建脚本内容
let scriptsContent = '<head>';
scriptsContent += `<script>var trickAPP = parent["Trick-Comp-_BoxPage"]["${innerLoad.id}"];</script>`;
scripts.forEach(script => {
  scriptsContent += script.outerHTML;
});

// 注入到 iframe
iframe.srcdoc = scriptsContent;
```

#### 2. windowProxy 实现

```javascript
const windowProxy = new Proxy(window, {
  get(origin, key, receiver) {
    // 拦截 addEventListener，添加 AbortSignal 支持
    if (key === "addEventListener") {
      return function(...args) {
        const { signal } = innerLoad.abortcontroller;
        args[2] = { ...args[2], signal };
        return Reflect.apply(window[key], window, args);
      };
    }
    
    // 沙箱特有属性访问 iframe.contentWindow
    if (key === "PubSub" || innerLoad.windowSet.includes(key) || !origin.hasOwnProperty(key)) {
      return windowShadow[key];
    }
    
    // 默认访问母页面 window
    return origin[key];
  },
  
  set(origin, key, value) {
    // 事件处理属性特殊处理
    if (key === "onscroll" || key === "onresize") {
      window.addEventListener(key.substring(2), value, { signal });
      return true;
    }
    // 其他属性写入沙箱 window
    innerLoad.windowSet.push(key);
    windowShadow[key] = value;
    return true;
  }
});
```

#### 3. documentProxy 实现

```javascript
const documentProxy = new Proxy(document, {
  get(origin, key, receiver) {
    // DOM 查询方法代理到 Shadow DOM
    if ((key.startsWith("get") || key.startsWith("create") || key.startsWith("query")) 
        && typeof documentShadow[key] === 'function') {
      return function(...args) {
        return Reflect.apply(documentShadow[key], documentShadow, args);
      };
    }
    
    // body/head 指向 Shadow DOM
    if (key === "body" || key === "head") return documentShadow;
    
    // defaultView 指向 windowProxy
    if (key === "defaultView") return windowProxy;
    
    return origin[key];
  }
});
```

### 沙箱隔离特性

| 隔离维度 | 实现方式 | 说明 |
|---------|---------|------|
| **JS 执行环境** | iframe | JS 运行在独立的 iframe 上下文中 |
| **DOM 访问** | Proxy + Shadow DOM | document.querySelector 等操作被代理到 Shadow DOM |
| **Window 访问** | Proxy | window 对象访问被拦截和重定向 |
| **事件监听** | AbortController | 所有事件监听器可批量清理 |
| **全局变量** | 白名单机制 | `innerLoad.windowSet` 记录沙箱写入的全局变量 |

---

## 问题三：子页面 JS 与 HTML 的通信机制

### 核心机制：Proxy 代理 + 全局变量桥接

子页面的 JS 通过 `trickAPP` 全局变量访问代理对象，代理对象将操作转发到母页面的 Shadow DOM。

### 入口函数与调用链路

```
子页面代码
  ↓
trickAPP.documentProxy.getElementById('id_body')
  ↓
documentProxy.get 拦截器
  ↓
documentShadow.getElementById()
  ↓
返回 Shadow DOM 中的真实元素
  ↓
trickAPP.windowProxy.createRoot(element)
  ↓
windowProxy.get 拦截器
  ↓
React.createRoot() 在 Shadow DOM 中渲染
```

### 通信机制详解

#### 1. 桥接变量注入

**文件位置**: `web/base/page/index.js` (约第 1560 行)

```javascript
// 在 iframe 中注入 trickAPP 全局变量
scriptsContent += `<script>var trickAPP = parent["Trick-Comp-_BoxPage"]["${innerLoad.id}"];</script>`;
```

#### 2. 子页面使用桥接变量

**文件位置**: `test/home.html` (第 21-45 行)

```javascript
// 子页面通过 trickAPP 访问代理对象
(function(document, window, root, HTMLDocument, HTMLElement, ...) {
    // document 实际上是 documentProxy
    // window 实际上是 windowProxy
    
    window.TrickAPPRoot = createRoot(document.getElementById('id_body'));
    window.TrickAPPRoot.render(
        <StrictMode>
            <h1>Hello, world</h1>
        </StrictMode>
    );
})(trickAPP.documentProxy, trickAPP.windowProxy, trickAPP.windowProxy, ...);
```

#### 3. 代理对象的通信转发

```javascript
// 母页面挂载代理对象到全局
topWindow["Trick-Comp-_BoxPage"][innerLoad.id] = {
    "documentProxy": documentProxy,    // DOM 操作代理
    "windowProxy": windowProxy,        // Window 操作代理
    "rootProxy": windowProxy,          // 根对象代理
    "HTMLDocument": HTMLDocument,      // DOM 类引用
    "HTMLElement": HTMLElement,
    // ... 其他 DOM 类
};
```

### 通信流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                         子页面 (iframe)                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  React Code                                               │  │
│  │                                                           │  │
│  │  document.getElementById('id_body')                       │  │
│  │         ↓                                                 │  │
│  │  trickAPP.documentProxy.getElementById()                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ↓ 跨帧访问                              │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                         母页面 (Host)                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Proxy 拦截器 (documentProxy)                              │  │
│  │                                                           │  │
│  │  get(target, 'getElementById') {                          │  │
│  │      return (...args) => documentShadow.getElementById(...args) │  │
│  │  }                                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Shadow DOM (真实 DOM 操作)                     │  │
│  │                                                           │  │
│  │  <div id="id_body">                                       │  │
│  │      <h1>Hello, world</h1>  ← React 渲染结果               │  │
│  │  </div>                                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 关键通信点

| 通信方向 | 机制 | 示例 |
|---------|------|------|
| JS → HTML | documentProxy | `document.getElementById()` 操作 Shadow DOM |
| JS → 全局变量 | windowProxy | `window.TrickAPPRoot = createRoot(...)` |
| 母页 → 子页 | parent 对象 | `parent["Trick-Comp-_BoxPage"][id]` |
| 事件通信 | Proxy 拦截 | `addEventListener` 被代理到母页面 |

---

## 架构总结

### 三层隔离架构

```
┌────────────────────────────────────────────────────────────────────┐
│  Layer 1: 渲染层 (Shadow DOM)                                       │
│  - 子页面 HTML/CSS 渲染                                             │
│  - 完全隔离的 DOM 子树                                              │
│  - 样式天然隔离                                                     │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│  Layer 2: JS 执行层 (iframe)                                        │
│  - 子页面 JavaScript 执行环境                                        │
│  - 独立的 JavaScript 上下文                                          │
│  - 通过 srcdoc 注入代码                                              │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│  Layer 3: 代理层 (Proxy)                                            │
│  - windowProxy: 拦截 window 访问                                     │
│  - documentProxy: 拦截 DOM 操作，转发到 Shadow DOM                   │
│  - 实现 JS 与 HTML 的桥接通信                                        │
└────────────────────────────────────────────────────────────────────┘
```

### 核心依赖库

| 库 | 用途 | 版本 |
|---|------|------|
| **React** | UI 渲染框架 | 18.x |
| **React DOM** | DOM 操作 | 18.x |
| **i18next** | 国际化 | - |
| **immer** | 不可变状态管理 | - |
| **PubSubJS** | 发布订阅通信 | - |
| **lodash-es** | 工具函数 | - |

### 优势与特点

1. **真正的样式隔离**: Shadow DOM 确保子页面样式不影响母页面
2. **JS 执行隔离**: iframe 提供独立的 JavaScript 执行环境
3. **受控的 DOM 访问**: Proxy 代理确保子页面只能访问自己的 Shadow DOM
4. **可清理的资源**: AbortController 管理所有事件监听器，便于销毁
5. **React 兼容**: 支持 React 18 的 createRoot API
