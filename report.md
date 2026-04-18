# 微前端实现分析报告

## 一、项目概述

本项目是一个基于 React 的微前端实现，核心组件为 `_BoxPage`，采用 **Shadow DOM + 隐藏 iframe + JS Proxy** 的技术方案实现无侵入式的微前端沙箱隔离。

---

## 二、问题1：子页面 HTML/CSS 非 iframe 引入机制

### 2.1 入口函数

**入口函数：`SandboxCreate` (./Code/Component/_BoxPage/__SandBox.js:1708)**

**调用链路：**
```
_BoxPage (useEffect) → SandboxCreate → SandboxLoad
```

### 2.2 核心实现原理

#### 技术方案：Shadow DOM 渲染

| 阶段 | 函数位置 | 关键操作 |
|------|---------|---------|
| 创建 Shadow Root | __SandBox.js:1710-1712 | `element.current.attachShadow({mode: 'open'})` |
| 加载子页面 | __SandBox.js:1488-1493 | `fetch(src)` 获取 HTML 文本 |
| DOM 解析 | __SandBox.js:1505-1506 | `DOMParser().parseFromString(htmlContent, 'text/html')` |
| CSS 注入 | __SandBox.js:1509-1517 | 提取子页面 `<link>` 并追加到 shadowRoot |
| HTML 注入 | __SandBox.js:1531 | `shadowRoot.appendChild(page.body.children[0])` |

### 2.3 关键代码说明

**web/base/page/index.js:1710-1712**
```javascript
// Shadow DOM 初始化
if (!element.current.shadowRoot) {
  element.current.attachShadow({ mode: 'open' });
}
```

**web/base/page/index.js:1509-1517**
```javascript
// 样式隔离：将子页面 CSS 注入 Shadow DOM
const cssLinks = page.head.querySelectorAll('link[rel="stylesheet"]');
cssLinks.forEach(link => {
  // 支持排除指定 CSS 文件
  let isInclude = true;
  if (link.href) {
    for (const key in excludeFile) {
      if (link.href.includes(excludeFile[key])) {
        isInclude = false;
        break;
      }
    }
  }
  if (isInclude) element.current.shadowRoot.appendChild(link);
});
```

### 2.4 工作原理总结

1. **不使用 iframe 承载 HTML/CSS**：仅用 `display: none` 的 iframe 做 JS 沙箱
2. **Shadow DOM 实现样式隔离**：子页面 DOM 树完全挂载在 Shadow Root 中，与主 DOM 树隔离
3. **原生浏览器能力**：利用浏览器标准的 Shadow DOM API，无额外性能开销
4. **CSS 作用域**：子页面样式仅在 Shadow DOM 范围内生效，避免全局污染

---

## 三、问题2：JS 沙箱隔离机制

### 3.1 入口函数

**入口函数：`SandboxLoad` (./Code/Component/_BoxPage/__SandBox.js:1485)**

**核心沙箱创建位置：__SandBox.js:1564-1675**

### 3.2 沙箱技术方案

| 沙箱层级 | 技术实现 | 作用 |
|---------|---------|------|
| JS 执行环境沙箱 | 隐藏 iframe.contentWindow | 提供独立的 JS 全局作用域 |
| window 代理层 | `new Proxy(window, {...})` | 拦截全局变量读写，实现白名单机制 |
| document 代理层 | `new Proxy(document, {...})` | 拦截 DOM 操作，重定向到 Shadow DOM |

### 3.3 隐藏 iframe 沙箱实现

**web/base/page/index.js:1543-1561**
```javascript
// 1. 创建隐藏 iframe (_BoxPageUI.js 中渲染)
// <iframe src="about:blank" className="hidden" ... />

// 2. 子页面 JS 注入到 iframe 执行
const scriptsContent = '<head>';
scriptsContent += `<script>
  var trickAPP = parent["Trick-Comp-_BoxPage"]["${innerLoad.id}"];
</script>`;
scripts.forEach(script => {
  scriptsContent += script.outerHTML;
});
iframe.srcdoc = scriptsContent;

// 3. 保存沙箱引用
const windowShadow = iframe.contentWindow;    // iframe 原生 window - JS 执行沙箱
const documentShadow = element.current.shadowRoot;  // Shadow DOM - 渲染沙箱
```

### 3.4 window Proxy 代理机制

**web/base/page/index.js:1565-1623**

```javascript
const windowProxy = new Proxy(window, {
  get(origin, key, receiver) {
    // 1. 事件监听劫持 - 自动注入 AbortSignal 实现自动清理
    if (key === "addEventListener") {
      return function(...args) {
        const { signal } = innerLoad.abortcontroller;
        args[2] = { ...args[2], signal };  // 自动绑定销毁信号
        return Reflect.apply(window[key], window, args);
      };
    }
    
    // 2. 白名单机制：使用 iframe 沙箱中的变量
    //    - PubSub: 通信通道
    //    - innerLoad.windowSet: 子页面写入的变量
    //    - 主 window 不存在的变量
    if (key === "PubSub" || innerLoad.windowSet.includes(key) || !origin.hasOwnProperty(key)) {
      return typeof windowShadow[key] === 'function' 
        ? (...args) => Reflect.apply(windowShadow[key], windowShadow, args)
        : windowShadow[key];
    }
    
    // 3. 默认：透传主 window 变量
    return typeof origin[key] === 'function'
      ? (...args) => Reflect.apply(origin[key], origin, args)
      : origin[key];
  },
  
  set(origin, key, value) {
    // 1. 特殊事件代理到主 window
    if (key === "onscroll" || key === "onresize") {
      window.addEventListener(key.substring(2), value, { signal });
      return true;
    }
    // 2. 其他变量全部写入 iframe 沙箱
    innerLoad.windowSet.push(key);
    windowShadow[key] = value;  // <<< 核心：变量写入 iframe 作用域
    return true;
  }
});
```

### 3.5 document Proxy 代理机制

**web/base/page/index.js:1626-1675**

```javascript
const documentProxy = new Proxy(document, {
  get(origin, key, receiver) {
    // 1. DOM 查询/创建方法重定向到 Shadow DOM
    if ((key.startsWith("get") || key.startsWith("create") || key.startsWith("query")) 
        && typeof documentShadow[key] === 'function') {
      return function(...args) {
        return Reflect.apply(documentShadow[key], documentShadow, args);
      };
    }
    
    // 2. 特殊属性映射
    if (key === "body" || key === "head" || key === "all") return documentShadow;
    if (key === "defaultView") return windowProxy;  // 返回代理后的 window
    
    // 3. 其他方法透传主 document
    return typeof origin[key] === 'function'
      ? (...args) => Reflect.apply(origin[key], origin, args)
      : origin[key];
  },
  
  set(origin, key, value) {
    documentShadow[key] = value;  // DOM 属性写入 Shadow DOM
    return true;
  }
});
```

### 3.6 沙箱机制总结

1. **双重沙箱架构**
   - **隐藏 iframe**：提供真正隔离的 JS 执行环境，全局变量写入 iframe.contentWindow
   - **Proxy 代理层**：作为中间层实现精确控制和重定向

2. **变量隔离机制**
   - 子页面写入的变量全部存入 `iframe.contentWindow`
   - 主页面只读变量（如 React、lodash）直接透传
   - 事件监听自动绑定 `AbortSignal`，组件销毁时自动清理

3. **DOM 操作隔离**
   - 所有 DOM 查询/创建操作自动重定向到 Shadow DOM
   - `document.body` → Shadow Root
   - `document.getElementById` → Shadow DOM 内查询

---

## 四、问题3：JS 沙箱与 HTML 通信机制

### 4.1 入口函数

**子页面入口：** `test/home.html:31` IIFE 立即执行函数

**沙箱通信桥梁创建：** `__SandBox.js:1678-1689`

### 4.2 通信桥梁：trickAPP 全局对象

**web/base/page/index.js:1678-1689**

```javascript
// 主页面：在 parent window 上暴露通信对象
topWindow["Trick-Comp-_BoxPage"][innerLoad.id] = {
  // 1. DOM 操作代理
  "documentProxy": documentProxy,   // Shadow DOM 代理
  "windowProxy": windowProxy,       // window 沙箱代理
  "rootProxy": windowProxy,
  
  // 2. 浏览器原生 API 透传（构造函数）
  "HTMLDocument": HTMLDocument,
  "HTMLElement": HTMLElement,
  "HTMLCollection": HTMLCollection,
  "Document": Document,
  "Node": Node,
  "Element": Element,
  "NodeList": NodeList
};
```

**子页面注入：** `__SandBox.js:1547`
```javascript
// iframe 内预先注入通信桥梁
scriptsContent += `<script>
  var trickAPP = parent["Trick-Comp-_BoxPage"]["${innerLoad.id}"];
</script>`;
```

### 4.3 子页面注入模式

**test/home.html:31-42**
```javascript
// 子页面 IIFE 函数注入沙箱上下文
(function(
  document, window, root, 
  HTMLDocument, HTMLElement, HTMLCollection, 
  Document, Node, Element, NodeList
) {
  // 子页面业务代码在此执行
  !function() {
    // React 根节点必须挂载到 window.TrickAPPRoot
    window.TrickAPPRoot = createRoot(document.getElementById('id_body'));
    window.TrickAPPRoot.render(
      <StrictMode>
        <h1>Hello, world</h1>
      </StrictMode>
    );
  }();
})(
  trickAPP.documentProxy,    // 第一个参数：代理后的 document
  trickAPP.windowProxy,      // 第二个参数：代理后的 window
  trickAPP.windowProxy,      // 第三个参数：根代理
  trickAPP.HTMLDocument,     // 浏览器原生构造函数
  trickAPP.HTMLElement,
  trickAPP.HTMLCollection,
  trickAPP.Document,
  trickAPP.Node,
  trickAPP.Element,
  trickAPP.NodeList
);
```

### 4.4 完整通信链路

```
                        ┌───────────────────────────────────────────────────┐
                        │                    主页面                        │
                        │  ┌────────────────────────────────────────────┐  │
                        │  │           parent window                    │  │
                        │  │  "Trick-Comp-_BoxPage": {                  │  │
                        │  │     [id] → { documentProxy, windowProxy }  │  │
                        │  └───────────────────┬────────────────────────┘  │
                        └───────────────────────┼───────────────────────────┘
                                                │
                    ┌───────────────────────────┼───────────────────────────┐
                    │  隐藏 iframe (JS 沙箱)    │                           │
                    │                           │                           │
                    │  var trickAPP = parent["Trick-Comp-_BoxPage"][id]     │
                    │                           │                           │
                    │  ┌───────────────────────▼───────────────────────┐   │
                    │  │                 子页面 IIFE                   │   │
                    │  │  (document, window, root, ...) => {           │   │
                    │  │     // 所有变量访问都经过 Proxy                │   │
                    │  │     window.TrickAPPRoot → 沙箱 windowShadow   │   │
                    │  │     document.getElementById → Shadow DOM      │   │
                    │  └───────────────────────────────────────────────┘   │
                    └───────────────────────────────────────────────────────┘
```

### 4.5 React 渲染生命周期约定

**沙箱销毁机制：** `__SandBox.js:1447-1466`
```javascript
function SandboxClean(element, innerLoad) {
  // 1. 通过约定变量销毁 React 应用
  innerLoad.windowShadow["TrickAPPRoot"].unmount();
  
  // 2. 触发 AbortController 取消所有事件监听
  innerLoad.abortcontroller.abort();
  
  // 3. 清空 Shadow DOM
  element.current.shadowRoot.innerHTML = "";
}
```

### 4.6 通信机制总结

1. **全局共享对象：trickAPP**
   - 主页面通过 `parent["Trick-Comp-_BoxPage"][id]` 暴露代理对象
   - iframe 沙箱启动前预先注入该引用

2. **IIFE 注入模式**
   - 子页面代码全部包裹在 IIFE 中
   - 通过函数参数注入代理后的 `document`/`window`
   - 子页面无感知使用沙箱环境

3. **约定大于配置**
   - 子页面必须将 React 根挂载到 `window.TrickAPPRoot`
   - 主页面通过该约定变量实现子应用生命周期管理

4. **原生构造函数透传**
   - HTMLDocument/HTMLElement 等原生类直接透传
   - 保证 `instanceof` 等操作正常工作
   - 避免跨 iframe 的原型链判断问题

---

## 五、整体架构总结

### 5.1 技术栈

| 层级 | 技术 | 位置 |
|------|------|------|
| 主框架 | React 18 + immer + use-immer | webpack 打包 |
| 状态管理 | Immer produce + React Context | |
| 微前端核心 | Shadow DOM + iframe + Proxy | `_BoxPage` 组件 |
| 事件总线 | PubSubJS | `Tools.PubSubListen/Send` |
| 国际化 | react-i18next | |

### 5.2 核心创新点

1. **渲染与执行分离**：Shadow DOM 做 HTML/CSS 渲染，隐藏 iframe 做 JS 执行
2. **双层代理架构**：window/document 双 Proxy 实现精确控制
3. **无侵入式设计**：子页面仅需简单包裹即可接入
4. **自动资源清理**：AbortController 自动管理事件监听

### 5.3 文件对应关系

| 功能 | 文件位置 |
|------|---------|
| 子页面示例 | test/home.html |
| 主页面入口 | web/base/index.html |
| 沙箱核心实现 | web/base/page/index.js:1441-1728 (_SandBox.js) |
| _BoxPage 组件 | web/base/page/index.js:1786-1851 |
