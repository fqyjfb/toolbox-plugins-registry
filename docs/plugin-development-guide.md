# ToolBox 插件开发规范

> 版本：1.2.0
> 最后更新：2026-07-16
> 适用范围：所有 ToolBox 插件商店插件

---

## 目录

1. [插件架构概述](#1-插件架构概述)
2. [技术栈](#2-技术栈)
3. [项目结构与文件规范](#3-项目结构与文件规范)
4. [manifest.json 规范](#4-manifestjson-规范)
5. [插件入口规范 (index.tsx)](#5-插件入口规范)
6. [插件 API 参考](#6-插件-api-参考)
7. [构建配置规范](#7-构建配置规范)
8. [代码风格规范](#8-代码风格规范)
9. [主题与 UI 规范](#9-主题与-ui-规范)
10. [无边框窗口样式规范](#10-无边框窗口样式规范)
11. [插件生命周期](#11-插件生命周期)
12. [插件注册表发布规范](#12-插件注册表发布规范)
13. [代码质量要求](#13-代码质量要求)
14. [插件安全检查清单](#14-插件安全检查清单)
15. [附录：完整示例](#15-附录完整示例)

---

## 1. 插件架构概述

### 1.1 核心设计原则

**所有 ToolBox 插件必须在独立 Electron 无边框窗口中运行**，不允许嵌入主窗口。

每个插件拥有自己的独立窗口，窗口**无原生边框**（`frame: false`），使用自定义渲染的标题栏控制按钮（最小化、最大化、关闭），与 ToolBox 主窗口完全隔离，视觉风格统一。

### 1.2 窗口特性

| 特性 | 说明 |
|------|------|
| **窗口类型** | Electron BrowserWindow，`frame: false`（无原生边框） |
| **标题栏** | 自定义渲染，高度 32px，支持拖拽 |
| **控制按钮** | 最小化、最大化/还原、关闭（右上角） |
| **背景** | 浅色模式 `#f8f9fa`，深色模式 `#1f2937` |
| **暗黑模式** | 自动跟随系统偏好，通过 `dark` CSS 类切换 |
| **窗口控制** | 通过 `window.electron.plugin` API 控制 |

### 1.3 插件加载与注册流程

```
ToolBox 主窗口启动
    ↓
动态加载已安装插件的 dist/index.js（注入 React 全局变量）
    ↓
调用 module.exports.register(pluginApi) 注册工具入口和侧边栏按钮
    ↓
用户在工具列表或侧边栏点击插件 → 调用 openPluginWindow()
    ↓
Electron 创建无边框窗口（frame: false），注入 __PLUGIN_DATA__ 全局变量
    ↓
插件检测到 __PLUGIN_DATA__ → 调用 renderStandalone() 渲染独立窗口 UI
    ↓
插件在无边框窗口中运行，拥有自定义标题栏和完整的窗口控制能力
```

### 1.4 关键区分：注册阶段 vs 运行阶段

| 阶段 | 运行位置 | 作用 |
|------|----------|------|
| **注册阶段** | ToolBox 主窗口 | 调用 `registerTool()` / `registerSidebarButton()` 让插件在工具列表和侧边栏中可见 |
| **运行阶段** | 独立 Electron 无边框窗口 | 检测 `__PLUGIN_DATA__` 后调用 `renderStandalone()` 渲染 UI |

### 1.5 运行环境

- **宿主**：Electron + React 19 应用
- **JS 环境**：Chromium V8 引擎
- **React 版本**：19.x（通过 `window.React` 全局注入）
- **样式方案**：Tailwind CSS（通过 CDN 注入到插件窗口）
- **暗黑模式**：自动跟随系统偏好，使用 `dark:` 前缀类名和 `.dark` CSS 类
- **无边框窗口特性**：可通过 `window.electron.plugin` 控制窗口（最小化、最大化、关闭）

---

## 2. 技术栈

### 2.1 必需技术栈

| 技术 | 版本要求 | 说明 |
|------|----------|------|
| **TypeScript** | ^6.0.0 | 源码必须使用 TypeScript |
| **React** | ^19.2.0 | 通过宿主全局注入，**不可独立安装为运行时依赖** |
| **ReactDOM** | ^19.2.0 | 通过宿主全局注入，**不可独立安装为运行时依赖** |
| **Vite** | ^5.0.0 | 构建工具，**必须**使用 |
| **Node.js** | >=20.x | 构建时要求 |

### 2.2 可选依赖

插件可根据需要安装额外依赖，但必须满足以下条件：

- **允许**：纯 JavaScript 工具库（如 lodash、date-fns 等）
- **禁止**：Node.js 原生模块（无法在浏览器环境中运行）
- **禁止**：Electron API（插件运行在渲染进程中，无法直接访问 Electron 主进程 API）
- **禁止**：修改 `window` 或 `document` 全局对象

### 2.3 构建产物要求

- **格式**：IIFE（Immediately Invoked Function Expression）
- **输出文件**：`dist/index.js`（**必须**，单文件）
- **外部依赖**：`react`、`react-dom` 必须标记为 external，从 `window.React`、`window.ReactDOM` 获取
- **ESBuild JSX**：必须配置 `jsxFactory: 'window.React.createElement'`

---

## 3. 项目结构与文件规范

### 3.1 标准目录结构

```
plugin-{name}/
├── package.json          # 必须：项目配置
├── manifest.json         # 必须：插件元信息（注册表用）
├── build.mjs             # 必须：Vite 构建脚本
├── tsconfig.json         # 推荐：TypeScript 配置
├── src/
│   ├── index.tsx         # 必须：插件入口文件
│   ├── ToolPanel.tsx     # 必须：主 UI 组件
│   └── *.ts              # 可选：工具函数、类型定义等
├── dist/                 # 构建输出（不提交到 git）
│   └── index.js
└── .gitignore
```

### 3.2 package.json 规范

```json
{
  "name": "plugin-{name}",
  "version": "1.0.0",
  "description": "插件中文描述",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "build": "node build.mjs"
  },
  "dependencies": {
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0",
    "typescript": "^6.0.0",
    "vite": "^5.0.0"
  }
}
```

**关键字段说明**：

- `name`：必须以 `plugin-` 前缀开头
- `type`：必须为 `"module"`（ESM）
- `main`：必须指向 `dist/index.js`
- `dependencies`：`react` 和 `react-dom` 必须声明（构建时需要），运行时从宿主注入

---

## 4. manifest.json 规范

`manifest.json` 是插件注册表的核心元信息文件，位于插件项目根目录。

### 4.1 完整结构

```json
{
  "id": "plugin-{name}",
  "name": "插件中文名",
  "version": "1.0.0",
  "description": "插件功能描述，不超过 100 字",
  "author": "ToolBox Team",
  "entry": "dist/index.js",
  "icon": "LucideIconName",
  "iconUrl": "https://raw.githubusercontent.com/{owner}/toolbox-plugins-registry/main/icons/{plugin-name}/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "categories": ["工具"],
  "tags": ["tag1", "tag2"],
  "githubRepo": "{owner}/{repo-name}",
  "isBeta": false
}
```

### 4.2 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，格式 `plugin-{name}` |
| `name` | string | 是 | 插件中文名，显示在 UI 中 |
| `version` | string | 是 | 语义化版本号，格式 `x.y.z` |
| `description` | string | 是 | 功能描述，不超过 100 字 |
| `author` | string | 是 | 作者名 |
| `entry` | string | 是 | 入口文件路径（相对于插件目录），默认 `dist/index.js` |
| `icon` | string | 是 | Lucide 图标名，如 `Palette`、`Camera`、`Package` |
| `iconUrl` | string | 否 | 插件图标 URL，存储在注册表 `icons/{plugin-name}/icon.png` |
| `color` | string | 是 | 主色调，HEX 格式 |
| `textColor` | string | 否 | 图标文字色，默认 `#ffffff` |
| `categories` | string[] | 是 | 分类标签数组，可选值：`设计`、`工具`、`开发`、`文件`、`文本`、`数据`、`AI`、`系统` |
| `tags` | string[] | 否 | 搜索标签，小写英文，逗号分隔 |
| `githubRepo` | string | 是 | GitHub 仓库地址，格式 `{owner}/{repo}` |
| `isBeta` | boolean | 否 | 是否为测试版，默认 `false` |

### 4.3 分类值参考

当前 ToolBox 支持的分类：

- `工具` - 通用工具类
- `设计` - 设计相关工具
- `开发` - 开发辅助工具
- `文件` - 文件处理工具
- `文本` - 文本处理工具
- `数据` - 数据格式化工具
- `AI` - AI 相关工具
- `系统` - 系统管理工具

---

## 5. 插件入口规范 (index.tsx)

### 5.1 入口文件结构

`src/index.tsx` 必须包含以下六个部分（**顺序不可调整**）：

1. **头部引入**：从 `window` 获取 `React` 和 `ReactDOM`
2. **PluginHeader 组件**：无边框窗口的自定义标题栏，包含最小化/最大化/关闭按钮
3. **PluginApp 组件**：无边框窗口的根组件
4. **renderStandalone 函数**：无边框窗口的渲染入口
5. **registerPlugin 函数**：注册工具入口和侧边栏按钮
6. **条件渲染**：检测 `__PLUGIN_DATA__` 触发无边框窗口渲染
7. **模块导出**：`module.exports = { register }`

### 5.2 完整入口模板

```typescript
import ToolPanel from './ToolPanel';

const { React, ReactDOM } = window as any;

// === 1. 无边框窗口自定义标题栏组件 ===
const PluginHeader: React.FC<{ title: string }> = ({ title }) => {
  const handleMinimize = () => {
    (window as any).electron?.plugin?.minimizeWindow();
  };

  const handleMaximize = () => {
    (window as any).electron?.plugin?.maximizeWindow();
  };

  const handleClose = () => {
    (window as any).electron?.plugin?.closeWindow();
  };

  return React.createElement('div', { className: 'plugin-header' },
    React.createElement('div', { className: 'plugin-header-title' }, title),
    React.createElement('div', { className: 'plugin-header-controls' },
      React.createElement('button', { onClick: handleMinimize },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('path', { d: 'M5 12h14' })
        )
      ),
      React.createElement('button', { onClick: handleMaximize },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('rect', { x: '3', y: '3', width: '18', height: '18', rx: '2', ry: '2' })
        )
      ),
      React.createElement('button', { onClick: handleClose },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('path', { d: 'M18 6L6 18M6 6l12 12' })
        )
      )
    )
  );
};

// === 2. 无边框窗口根组件 ===
const PluginApp: React.FC = () => {
  const pluginData = (window as any).__PLUGIN_DATA__;
  const title = pluginData?.pluginName || 'ToolBox 插件';

  return React.createElement(React.Fragment, null,
    React.createElement(PluginHeader, { title }),
    React.createElement('div', { className: 'plugin-content' },
      React.createElement(ToolPanel)
    )
  );
};

// === 3. 无边框窗口渲染函数 ===
function renderStandalone() {
  if (!React || !ReactDOM) {
    console.error('React or ReactDOM is not available');
    return;
  }

  const root = document.getElementById('root');
  if (!root) {
    console.error('Root element not found');
    return;
  }

  if (ReactDOM.createRoot) {
    ReactDOM.createRoot(root).render(React.createElement(PluginApp));
  } else {
    ReactDOM.render(React.createElement(PluginApp), root);
  }
}

// === 4. 插件注册函数 ===
function registerPlugin(api: any) {
  const { registerTool, registerSidebarButton, openPluginWindow } = api;

  // 注册工具入口（使插件出现在工具列表中）
  registerTool({
    id: 'plugin-{name}',
    name: '插件中文名',
    iconName: 'LucideIcon',
    color: '#3b82f6',
    textColor: '#ffffff',
    path: '/tools/plugin-{name}',
    component: ToolPanel,
  });

  // 注册侧边栏按钮（点击后打开无边框窗口）
  registerSidebarButton({
    id: 'plugin-{name}-btn',
    icon: 'LucideIcon',
    label: '插件中文名',
    onClick: () => {
      openPluginWindow?.('plugin-{name}');
    },
  });
}

// === 5. 条件渲染（无边框窗口模式） ===
const pluginData = (window as any).__PLUGIN_DATA__;

if (pluginData) {
  renderStandalone();
}

// === 6. 模块导出 ===
module.exports = {
  register: registerPlugin,
};
```

### 5.3 各部分详细说明

#### 5.3.1 头部引入

```typescript
import ToolPanel from './ToolPanel';

const { React, ReactDOM } = window as any;
```

- 从 `window` 获取 React 和 ReactDOM（由宿主全局注入）
- 使用 `as any` 类型断言（编译时需要，运行时安全）
- `ToolPanel` 是插件的主 UI 组件，必须导出

#### 5.3.2 PluginHeader 组件（必须实现）

无边框窗口的自定义标题栏组件，**必须**实现三个窗口控制按钮：

| 按钮 | 功能 | 调用 API | SVG 路径 |
|------|------|----------|----------|
| 最小化 | 最小化窗口到任务栏 | `window.electron.plugin.minimizeWindow()` | `M5 12h14` |
| 最大化 | 窗口最大化/还原 | `window.electron.plugin.maximizeWindow()` | `rect x="3" y="3" width="18" height="18" rx="2" ry="2"` |
| 关闭 | 关闭窗口 | `window.electron.plugin.closeWindow()` | `M18 6L6 18M6 6l12 12` |

**关键实现细节**：

- 标题栏高度固定为 **32px**
- 标题栏支持拖拽：`-webkit-app-region: drag`
- 控制按钮不支持拖拽：`-webkit-app-region: no-drag`
- 关闭按钮悬停时背景变为红色（`#ef4444`），图标变为白色
- 按钮使用内联 SVG 图标（不使用 Lucide 图标库），确保在独立窗口中无依赖渲染

#### 5.3.3 PluginApp 组件（必须实现）

无边框窗口的根组件，结构：

```
PluginApp
├── PluginHeader（自定义标题栏，高度 32px）
└── div.plugin-content（主内容区域，flex: 1）
    └── ToolPanel（插件主 UI）
```

窗口标题从 `window.__PLUGIN_DATA__.pluginName` 读取，fallback 为 `'ToolBox 插件'`。

#### 5.3.4 renderStandalone 函数（必须实现）

无边框窗口的渲染入口，功能：
1. 检查 `React` 和 `ReactDOM` 可用性
2. 查找 `#root` DOM 元素
3. 使用 `ReactDOM.createRoot()`（React 19）或 `ReactDOM.render()`（兼容旧版）渲染 `PluginApp`

#### 5.3.5 registerPlugin 函数（必须实现）

注册函数，接收 `pluginApi` 对象，必须调用：

- `registerTool()`：注册工具入口，使插件在 ToolBox 工具列表中可见
- `registerSidebarButton()`：注册侧边栏按钮，点击后调用 `openPluginWindow()` 打开无边框窗口

#### 5.3.6 条件渲染（必须实现）

```typescript
const pluginData = (window as any).__PLUGIN_DATA__;
if (pluginData) {
  renderStandalone();
}
```

当 `window.__PLUGIN_DATA__` 存在时，说明当前在无边框窗口环境中，触发渲染。

#### 5.3.7 模块导出（必须实现）

```typescript
module.exports = {
  register: registerPlugin,
};
```

导出 `register` 函数，由 ToolBox 主窗口在脚本加载完成后调用。

---

## 6. 插件 API 参考

### 6.1 PluginApi 接口

ToolBox 通过 `pluginApi` 单例对象向插件暴露 API。插件通过 `register(api)` 接收此对象。

### 6.2 registerTool

注册一个工具入口，使插件在 ToolBox 工具列表中可见。

```typescript
api.registerTool({
  id: string,          // 唯一标识符，格式 plugin-{name}
  name: string,        // 工具中文名
  iconName: string,    // Lucide 图标名
  color: string,       // 主色调 HEX
  textColor: string,   // 文字色 HEX
  path: string,        // 路由路径，格式 /tools/{plugin-id}
  component: any,      // React 组件（ToolPanel）
});
```

### 6.3 registerSidebarButton

注册一个侧边栏按钮，点击后打开插件无边框窗口。

```typescript
api.registerSidebarButton({
  id: string,          // 唯一标识符，格式 plugin-{name}-btn
  icon: string,        // Lucide 图标名
  label: string,       // 按钮标签（中文）
  onClick: () => void, // 点击回调，必须调用 openPluginWindow
});
```

### 6.4 openPluginWindow

打开插件无边框窗口。

```typescript
api.openPluginWindow(pluginId: string): Promise<void>;
```

### 6.5 navigate

跳转到指定路由。

```typescript
api.navigate(path: string): void;
```

### 6.6 registerPanel / registerSettingsPanel

注册面板或设置面板（高级用法）。

```typescript
api.registerPanel(id: string, panel: { id: string; render: () => React.ReactNode });
api.registerSettingsPanel(id: string, panel: { id: string; render: () => React.ReactNode });
```

---

## 7. 构建配置规范

### 7.1 build.mjs 标准模板

```javascript
import { build } from 'vite';
import { fileURLToPath } from 'url';
import path from 'path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

async function runBuild() {
  await build({
    root: __dirname,
    build: {
      lib: {
        entry: path.resolve(__dirname, 'src/index.tsx'),
        name: '{PluginName}Plugin',
        formats: ['iife'],
        fileName: () => 'index.js'
      },
      outDir: path.resolve(__dirname, 'dist'),
      emptyOutDir: true,
      minify: false,
      rollupOptions: {
        external: ['react', 'react-dom'],
        output: { globals: { react: 'window.React', 'react-dom': 'window.ReactDOM' } }
      }
    },
    esbuild: {
      jsxFactory: 'window.React.createElement',
      jsxFragment: 'window.React.Fragment'
    }
  });
  console.log("Build complete: dist/index.js generated.");
}
runBuild();
```

### 7.2 关键配置说明

| 配置项 | 值 | 原因 |
|--------|-----|------|
| `formats` | `['iife']` | 必须，确保生成自执行函数 |
| `external` | `['react', 'react-dom']` | 必须，React 由宿主注入 |
| `globals` | `{ react: 'window.React', 'react-dom': 'window.ReactDOM' }` | 必须，指向宿主全局变量 |
| `jsxFactory` | `'window.React.createElement'` | 必须，使用宿主 React |
| `minify` | `false` | 推荐，方便调试 |

### 7.3 构建命令

```bash
npm run build
```

构建完成后，`dist/index.js` 即为最终产物。

---

## 8. 代码风格规范

### 8.1 TypeScript 配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["DOM", "DOM.Iterable", "ES2020"],
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

### 8.2 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 文件名 | kebab-case | `color-utils.ts` |
| 组件名 | PascalCase | `ToolPanel.tsx` |
| 接口/类型 | PascalCase | `ColorScheme` |
| 函数名 | camelCase | `hexToRgb` |
| 变量名 | camelCase | `colorHex` |
| 常量 | UPPER_SNAKE_CASE | `MAX_COLORS` |

### 8.3 代码质量要求

| 规则 | 要求 |
|------|------|
| 方法行数 | ≤ 50 行 |
| 文件行数 | ≤ 300 行 |
| 组件行数 | ≤ 200 行 |
| 无未使用变量 | 必须 |
| 无 console.log | 生产代码禁止 |
| 类型安全 | 禁止 `any`（除非必要，需注释说明） |
| 错误处理 | 必须处理异常，不吞异常 |

### 8.4 注释规范

- **函数/方法**：复杂逻辑必须添加注释
- **魔法数字**：必须用常量替代
- **TODO**：必须标注，格式 `// TODO: 描述`
- **语言**：注释使用中文

### 8.5 禁止模式

- 禁止使用 `setTimeout` / `setInterval` 直接操作 DOM
- 禁止直接修改 `window` 或 `document` 全局对象
- 禁止使用 `eval()`、`new Function()`
- 禁止使用 `localStorage`（除非通过宿主 API）
- 禁止使用 `fetch` 请求非公开 API（需用户明确授权）
- 禁止修改宿主 CSS 样式（使用 Tailwind 类名）

---

## 9. 主题与 UI 规范

### 9.1 颜色系统

插件必须遵循 ToolBox 主题系统，使用 Tailwind CSS 类名。

| 用途 | 浅色模式 | 深色模式 | Tailwind 类名 |
|------|----------|----------|---------------|
| 主文本 | `#111827` | `#f9fafb` | `text-gray-800 dark:text-gray-200` |
| 次级文本 | `#6b7280` | `#d1d5db` | `text-gray-500 dark:text-gray-400` |
| 三级文本 | `#9ca3af` | `#9ca3af` | `text-gray-400 dark:text-gray-500` |
| 背景 | `#ffffff` | `#111827` | `bg-white dark:bg-gray-800` |
| 卡片背景 | `#ffffff` | `#1f2937` | `bg-white dark:bg-gray-800` |
| 次级背景 | `#f8f9fa` | `#1f2937` | `bg-gray-50 dark:bg-gray-700` |
| 边框 | `#e5e7eb` | `#374151` | `border-gray-200 dark:border-gray-700` |
| 主按钮 | `#275D7E` | `#275D7E` | `bg-primary` |
| 成功色 | `#22c55e` | `#22c55e` | `text-green-500` |
| 警告色 | `#f59e0b` | `#fbbf24` | `text-amber-500 dark:text-amber-400` |
| 错误色 | `#ef4444` | `#f87171` | `text-red-500 dark:text-red-400` |
| 信息色 | `#3b82f6` | `#60a5fa` | `text-blue-500 dark:text-blue-400` |

### 9.2 字体规范

| 用途 | 尺寸 | Tailwind 类名 |
|------|------|---------------|
| 小字 | 12px | `text-xs` |
| 辅助文字 | 14px | `text-sm` |
| 正文 | 16px | `text-base` |
| 小标题 | 20px | `text-lg` |
| 标题 | 24px | `text-xl` |
| 大标题 | 32px | `text-2xl` |

- 正文行高：`leading-relaxed`（1.5）
- 标题行高：`leading-tight`（1.25）
- 字体权重：正文 `font-normal`（400），标题 `font-semibold`（600）

### 9.3 间距规范

基于 4px 网格系统：

| 变量 | 值 | Tailwind 类名 |
|------|-----|---------------|
| `--space-1` | 4px | `p-1`, `m-1`, `gap-1` |
| `--space-2` | 8px | `p-2`, `m-2`, `gap-2` |
| `--space-3` | 12px | `p-3`, `m-3`, `gap-3` |
| `--space-4` | 16px | `p-4`, `m-4`, `gap-4` |
| `--space-5` | 24px | `p-5`, `m-5`, `gap-5` |
| `--space-6` | 32px | `p-6`, `m-6`, `gap-6` |
| `--space-7` | 48px | `p-7`, `m-7`, `gap-7` |
| `--space-8` | 64px | `p-8`, `m-8`, `gap-8` |

### 9.4 卡片规范

- **边框**：`border border-gray-200 dark:border-gray-700`
- **圆角**：`rounded-lg`（8px）或 `rounded-md`（6px）
- **阴影**：`shadow-md`（推荐）或 `shadow-sm`
- **内边距**：`p-4`（16px）
- **禁止**：同时使用边框和阴影（选其一）

### 9.5 按钮规范

| 类型 | 样式 | 悬停效果 |
|------|------|----------|
| 主按钮 | 实心填充，无渐变 | 颜色加深 10% |
| 次按钮 | 描边或浅色背景 | 背景加深 |
| 幽灵按钮 | 透明背景，悬停显示背景 | 背景显示 |
| 禁止 | 全圆角（`rounded-full`） | - |

### 9.6 输入框规范

- **边框**：`border border-gray-300 dark:border-gray-600`
- **圆角**：`rounded-lg`（8px）
- **内边距**：`px-3 py-2`
- **聚焦**：`focus:outline-none focus:border-primary`（仅改变边框色）
- **背景**：`bg-white dark:bg-gray-700`

### 9.7 图标规范

- **图标库**：Lucide React（`lucide-react`）
- **内联图标**：`w-4 h-4`（16px）
- **独立图标**：`w-5 h-5`（20px）
- **颜色**：`text-gray-500 dark:text-gray-400`
- **禁止**：使用 emoji 作为功能性图标
- **注意**：无边框窗口标题栏按钮必须使用内联 SVG（不依赖 lucide-react）

### 9.8 暗黑模式适配

所有 UI 组件必须支持暗黑模式，使用 Tailwind 的 `dark:` 前缀：

```tsx
<div className="bg-white dark:bg-gray-800 text-gray-800 dark:text-gray-200 border border-gray-200 dark:border-gray-700">
  内容
</div>
```

插件窗口的暗黑模式由 `plugin-window.html` 自动检测系统偏好并添加 `.dark` class。

### 9.9 禁止的 UI 模式

- 禁止蓝紫渐变背景
- 禁止玻璃拟态（glassmorphism）
- 禁止霓虹色
- 禁止彩虹配色
- 禁止每元素都加阴影
- 禁止超过 2 个阴影层级
- 禁止使用行内样式定义颜色、间距、字体
- 禁止魔法数字（使用 Tailwind 类名或 CSS 变量）

---

## 10. 无边框窗口样式规范

### 10.1 窗口结构

插件运行在 Electron 无边框窗口中，窗口结构如下：

```
┌─────────────────────────────────────────────────────┐
│ plugin-header（自定义标题栏，高度 32px）              │
│ ┌─────────────────────────────────────────────────┐ │
│ │ plugin-header-title（居中标题，13px，font-weight 500）│ │
│ │                    插件名称                      │ │
│ └─────────────────────────────────────────────────┘ │
│                     ↕                              │
│ ┌─────────────────────────────────────────────────┐ │
│ │ plugin-header-controls（右侧控制按钮组）          │ │
│ │ [最小化] [最大化] [关闭]                         │ │
│ └─────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│ plugin-content（主内容区域，flex: 1，overflow: hidden）│
│ ┌─────────────────────────────────────────────────┐ │
│ │              ToolPanel 插件主 UI                │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 10.2 窗口容器样式

插件窗口的 `#root` 元素使用以下样式：

```css
#root {
  height: 100%;
  display: flex;
  flex-direction: column;
}
```

### 10.3 标题栏样式（plugin-header）

标题栏是无边框窗口的核心组件，样式定义在 `plugin-window.html` 中：

| 属性 | 值 | 说明 |
|------|-----|------|
| `display` | `flex` | 弹性布局 |
| `align-items` | `center` | 垂直居中 |
| `justify-content` | `space-between` | 两端对齐 |
| `padding` | `0 4px` | 左右内边距 |
| `height` | `32px` | 固定高度 |
| `background` | `#ffffff` | 浅色模式背景 |
| `border-bottom` | `1px solid #e5e7eb` | 底部边框 |
| `-webkit-app-region` | `drag` | 支持拖拽 |

**暗黑模式**：

| 属性 | 值 |
|------|-----|
| `background` | `#1f2937` |
| `border-bottom-color` | `#374151` |

### 10.4 标题文字样式（plugin-header-title）

| 属性 | 值 | 说明 |
|------|-----|------|
| `flex` | `1` | 占据剩余空间 |
| `display` | `flex` | 弹性布局 |
| `align-items` | `center` | 垂直居中 |
| `justify-content` | `center` | 水平居中 |
| `font-size` | `13px` | 字体大小 |
| `font-weight` | `500` | 字体粗细 |
| `color` | `#374151` | 浅色模式文字色 |

**暗黑模式**：

| 属性 | 值 |
|------|-----|
| `color` | `#e5e7eb` |

### 10.5 控制按钮组样式（plugin-header-controls）

| 属性 | 值 | 说明 |
|------|-----|------|
| `display` | `flex` | 弹性布局 |
| `align-items` | `center` | 垂直居中 |
| `-webkit-app-region` | `no-drag` | 不支持拖拽 |

### 10.6 控制按钮样式

| 属性 | 值 | 说明 |
|------|-----|------|
| `width` | `46px` | 固定宽度 |
| `height` | `32px` | 固定高度（与标题栏一致） |
| `display` | `flex` | 弹性布局 |
| `align-items` | `center` | 垂直居中 |
| `justify-content` | `center` | 水平居中 |
| `border` | `none` | 无边框 |
| `background` | `transparent` | 透明背景 |
| `cursor` | `pointer` | 指针光标 |
| `transition` | `background-color 0.15s ease` | 悬停过渡 |

**悬停效果**：

| 按钮 | 悬停背景色 |
|------|------------|
| 最小化 | `#f3f4f6` |
| 最大化 | `#f3f4f6` |
| 关闭 | `#ef4444`（红色） |

**暗黑模式悬停效果**：

| 按钮 | 悬停背景色 |
|------|------------|
| 最小化 | `#374151` |
| 最大化 | `#374151` |
| 关闭 | `#ef4444`（红色，不变） |

### 10.7 按钮图标样式

| 属性 | 值 | 说明 |
|------|-----|------|
| `width` | `14px` | 图标宽度 |
| `height` | `14px` | 图标高度 |
| `color` | `#6b7280` | 浅色模式图标色 |

**暗黑模式**：

| 属性 | 值 |
|------|-----|
| `color` | `#9ca3af` |

**关闭按钮特殊效果**：

- 关闭按钮悬停时，图标颜色变为白色
- 关闭按钮 SVG 使用 `stroke` 属性，颜色通过 CSS `color` 控制

### 10.8 主内容区域样式（plugin-content）

| 属性 | 值 | 说明 |
|------|-----|------|
| `flex` | `1` | 占据剩余空间 |
| `overflow` | `hidden` | 隐藏溢出 |

### 10.9 完整 CSS 定义

以下是 `plugin-window.html` 中定义的完整样式：

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background: #f8f9fa;
  height: 100vh;
  overflow: hidden;
}
.dark body {
  background: #1f2937;
}
#root {
  height: 100%;
  display: flex;
  flex-direction: column;
}
.plugin-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 4px;
  height: 32px;
  background: #ffffff;
  border-bottom: 1px solid #e5e7eb;
  flex-shrink: 0;
  -webkit-app-region: drag;
}
.dark .plugin-header {
  background: #1f2937;
  border-bottom-color: #374151;
}
.plugin-header-title {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
}
.dark .plugin-header-title {
  color: #e5e7eb;
}
.plugin-header-controls {
  display: flex;
  align-items: center;
  -webkit-app-region: no-drag;
}
.plugin-header-controls button {
  width: 46px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: none;
  background: transparent;
  cursor: pointer;
  transition: background-color 0.15s ease;
}
.plugin-header-controls button:hover {
  background: #f3f4f6;
}
.dark .plugin-header-controls button:hover {
  background: #374151;
}
.plugin-header-controls button:last-child:hover {
  background: #ef4444;
}
.plugin-header-controls button:last-child:hover svg {
  color: white;
}
.plugin-header-controls svg {
  width: 14px;
  height: 14px;
  color: #6b7280;
}
.dark .plugin-header-controls svg {
  color: #9ca3af;
}
.plugin-content {
  flex: 1;
  overflow: hidden;
}
```

### 10.10 插件开发注意事项

1. **样式继承**：插件窗口使用 `plugin-window.html` 作为宿主 HTML，其中已引入 Tailwind CSS CDN 和上述基础样式
2. **类名约束**：插件代码必须使用 `plugin-header`、`plugin-header-title`、`plugin-header-controls`、`plugin-content` 这些类名，不能自定义标题栏结构
3. **拖拽区域**：标题栏默认支持拖拽，控制按钮区域不支持拖拽（已通过 `-webkit-app-region: no-drag` 设置）
4. **暗黑模式**：由 `plugin-window.html` 自动检测系统偏好并添加 `.dark` class，插件只需使用 Tailwind 的 `dark:` 前缀即可
5. **SVG 图标**：控制按钮必须使用内联 SVG，颜色通过 CSS `color` 属性控制，不使用 fill 属性

---

## 11. 插件生命周期

### 11.1 插件加载流程

```
ToolBox 主窗口启动
    ↓
读取已安装插件列表
    ↓
对每个已启用插件：
    ↓
创建 <script> 标签 → 加载 dist/index.js
    ↓
注入 window.React, window.ReactDOM
    ↓
注入 window.module.exports
    ↓
脚本加载完成 → 调用 module.exports.register(pluginApi)
    ↓
插件注册工具入口（registerTool）和侧边栏按钮（registerSidebarButton）
    ↓
插件在工具列表和侧边栏中可见
```

### 11.2 无边框窗口启动流程

```
用户在工具列表或侧边栏点击插件
    ↓
调用 openPluginWindow(pluginId)
    ↓
Electron 创建无边框窗口（frame: false, titleBarStyle: 'hidden'）
    ↓
加载 plugin-window.html（包含 Tailwind CDN 和基础样式）
    ↓
注入 window.React, window.ReactDOM
    ↓
注入 window.__PLUGIN_DATA__（包含 pluginId, pluginName, pluginColor, entryPath）
    ↓
动态创建 <script> 标签加载插件 dist/index.js
    ↓
脚本检测到 __PLUGIN_DATA__ 存在
    ↓
调用 renderStandalone() → 渲染 PluginApp（包含自定义标题栏）
    ↓
无边框窗口显示插件 UI（带自定义标题栏和控制按钮）
```

### 11.3 插件卸载流程

```
用户卸载插件
    ↓
ToolBox 从扩展目录移除插件文件
    ↓
移除 <script> 标签
    ↓
清除 pluginApi 中的注册信息
    ↓
工具列表和侧边栏中移除插件入口
    ↓
插件完全移除
```

### 11.4 插件更新流程

```
检测到新版本
    ↓
用户确认更新
    ↓
下载新版本插件
    ↓
替换旧版本文件
    ↓
重新加载插件脚本
    ↓
更新完成
```

---

## 12. 插件注册表发布规范

### 12.1 发布流程

1. 完成插件开发并通过测试
2. 提交到 GitHub 仓库
3. 向 `toolbox-plugins-registry` 仓库提交 PR：
   - 更新 `registry.json`，添加插件条目
   - 上传图标到 `icons/{plugin-name}/icon.png`
4. PR 审核通过后合并到 `main` 分支

### 12.2 registry.json 条目规范

```json
{
  "id": "plugin-{name}",
  "name": "插件中文名",
  "description": "功能描述",
  "icon": "LucideIcon",
  "iconUrl": "https://raw.githubusercontent.com/fqyjfb/toolbox-plugins-registry/main/icons/{plugin-name}/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "version": "1.0.0",
  "author": "ToolBox Team",
  "entry": "dist/index.js",
  "categories": ["工具"],
  "tags": ["tag1", "tag2"],
  "githubRepo": "{owner}/{repo-name}",
  "isBeta": false
}
```

### 12.3 图标规范

| 属性 | 要求 |
|------|------|
| 尺寸 | 64x64 px（最小），128x128 px（推荐） |
| 格式 | PNG（带透明背景） |
| 文件路径 | `icons/{plugin-name}/icon.png` |
| 命名 | 小写 kebab-case |
| 风格 | 简洁、扁平化、与 ToolBox 品牌一致 |

---

## 13. 代码质量要求

### 13.1 静态检查

- 提交前必须通过 TypeScript 编译（零警告）
- 禁止未使用的 import
- 禁止未使用的变量
- 禁止未使用的函数

### 13.2 性能要求

- 插件加载时间 ≤ 500ms（从脚本加载到注册完成）
- 无边框窗口启动 ≤ 300ms（从 `openPluginWindow` 调用到 UI 渲染完成）
- 内存占用 ≤ 10MB（插件运行时）

### 13.3 兼容性要求

- 必须兼容 ToolBox 最新版本
- 必须兼容浅色/深色主题
- 必须兼容中英文环境

---

## 14. 插件安全检查清单

### 14.1 安全要求

- [ ] 不包含恶意代码
- [ ] 不访问未授权的网络资源
- [ ] 不读取或修改文件系统（除非用户明确授权）
- [ ] 不窃取用户数据
- [ ] 不使用 eval() 或类似危险 API
- [ ] 不注入外部脚本（除非明确标注）
- [ ] 正确处理错误，不暴露敏感信息

### 14.2 功能检查

- [ ] `register()` 方法正确实现
- [ ] `PluginHeader` 组件使用 `plugin-header`、`plugin-header-title`、`plugin-header-controls` 类名
- [ ] `PluginHeader` 组件实现最小化/最大化/关闭按钮（使用内联 SVG）
- [ ] `PluginApp` 组件正确组合 Header 和 ToolPanel，使用 `plugin-content` 类名
- [ ] `renderStandalone()` 函数正确实现
- [ ] `__PLUGIN_DATA__` 条件渲染逻辑正确
- [ ] `manifest.json` 字段完整且格式正确
- [ ] `dist/index.js` 可正常加载
- [ ] 浅色/深色主题均正常显示
- [ ] 无控制台错误
- [ ] 插件卸载后不影响其他插件

### 14.3 代码检查

- [ ] TypeScript 编译零警告
- [ ] 无未使用代码
- [ ] 无 console.log（生产环境）
- [ ] 注释使用中文
- [ ] 命名规范一致
- [ ] 文件结构清晰

---

## 15. 附录：完整示例

以下是一个最小可运行的插件示例（完全对齐 color-palette-plugin 的结构和无边框窗口样式）：

### 15.1 package.json

```json
{
  "name": "plugin-example",
  "version": "1.0.0",
  "description": "ToolBox 示例插件",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "build": "node build.mjs"
  },
  "dependencies": {
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "@types/node": "^24.0.0",
    "@types/react": "^19.2.0",
    "@types/react-dom": "^19.2.0",
    "typescript": "^6.0.0",
    "vite": "^5.0.0"
  }
}
```

### 15.2 manifest.json

```json
{
  "id": "plugin-example",
  "name": "示例插件",
  "version": "1.0.0",
  "description": "ToolBox 插件开发示例",
  "author": "ToolBox Team",
  "entry": "dist/index.js",
  "icon": "Package",
  "iconUrl": "https://raw.githubusercontent.com/fqyjfb/toolbox-plugins-registry/main/icons/example-plugin/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "categories": ["工具"],
  "tags": ["example", "demo"],
  "githubRepo": "fqyjfb/example-plugin",
  "isBeta": false
}
```

### 15.3 src/index.tsx

```typescript
import ExamplePanel from './ExamplePanel';

const { React, ReactDOM } = window as any;

// 无边框窗口自定义标题栏
const PluginHeader: React.FC<{ title: string }> = ({ title }) => {
  const handleMinimize = () => {
    (window as any).electron?.plugin?.minimizeWindow();
  };

  const handleMaximize = () => {
    (window as any).electron?.plugin?.maximizeWindow();
  };

  const handleClose = () => {
    (window as any).electron?.plugin?.closeWindow();
  };

  return React.createElement('div', { className: 'plugin-header' },
    React.createElement('div', { className: 'plugin-header-title' }, title),
    React.createElement('div', { className: 'plugin-header-controls' },
      React.createElement('button', { onClick: handleMinimize },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('path', { d: 'M5 12h14' })
        )
      ),
      React.createElement('button', { onClick: handleMaximize },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('rect', { x: '3', y: '3', width: '18', height: '18', rx: '2', ry: '2' })
        )
      ),
      React.createElement('button', { onClick: handleClose },
        React.createElement('svg', { viewBox: '0 0 24 24', fill: 'none', stroke: 'currentColor', strokeWidth: '2' },
          React.createElement('path', { d: 'M18 6L6 18M6 6l12 12' })
        )
      )
    )
  );
};

// 无边框窗口根组件
const PluginApp: React.FC = () => {
  const pluginData = (window as any).__PLUGIN_DATA__;
  const title = pluginData?.pluginName || 'ToolBox 插件';

  return React.createElement(React.Fragment, null,
    React.createElement(PluginHeader, { title }),
    React.createElement('div', { className: 'plugin-content' },
      React.createElement(ExamplePanel)
    )
  );
};

// 无边框窗口渲染
function renderStandalone() {
  if (!React || !ReactDOM) {
    console.error('React or ReactDOM is not available');
    return;
  }

  const root = document.getElementById('root');
  if (!root) {
    console.error('Root element not found');
    return;
  }

  if (ReactDOM.createRoot) {
    ReactDOM.createRoot(root).render(React.createElement(PluginApp));
  } else {
    ReactDOM.render(React.createElement(PluginApp), root);
  }
}

// 插件注册
function registerPlugin(api: any) {
  const { registerTool, registerSidebarButton, openPluginWindow } = api;

  registerTool({
    id: 'plugin-example',
    name: '示例插件',
    iconName: 'Package',
    color: '#3b82f6',
    textColor: '#ffffff',
    path: '/tools/plugin-example',
    component: ExamplePanel,
  });

  registerSidebarButton({
    id: 'plugin-example-btn',
    icon: 'Package',
    label: '示例插件',
    onClick: () => {
      openPluginWindow?.('plugin-example');
    },
  });
}

// 条件渲染
const pluginData = (window as any).__PLUGIN_DATA__;

if (pluginData) {
  renderStandalone();
}

module.exports = {
  register: registerPlugin,
};
```

### 15.4 src/ExamplePanel.tsx

```typescript
const { React } = window as any;

const ExamplePanel: React.FC = () => {
  return (
    <div className="h-full flex flex-col p-4 overflow-hidden">
      <div className="flex items-center gap-3 mb-4">
        <h2 className="text-lg font-semibold text-gray-800 dark:text-gray-200">
          示例插件
        </h2>
      </div>

      <div className="flex-1 flex items-center justify-center bg-white dark:bg-gray-800 rounded-lg shadow-md">
        <div className="text-center">
          <h1 className="text-2xl font-semibold text-gray-800 dark:text-gray-200 mb-2">
            欢迎使用 ToolBox 插件
          </h1>
          <p className="text-sm text-gray-500 dark:text-gray-400">
            这是一个插件开发示例
          </p>
        </div>
      </div>
    </div>
  );
};

export default ExamplePanel;
```

### 15.5 build.mjs

```javascript
import { build } from 'vite';
import { fileURLToPath } from 'url';
import path from 'path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

async function runBuild() {
  await build({
    root: __dirname,
    build: {
      lib: {
        entry: path.resolve(__dirname, 'src/index.tsx'),
        name: 'ExamplePlugin',
        formats: ['iife'],
        fileName: () => 'index.js'
      },
      outDir: path.resolve(__dirname, 'dist'),
      emptyOutDir: true,
      minify: false,
      rollupOptions: {
        external: ['react', 'react-dom'],
        output: { globals: { react: 'window.React', 'react-dom': 'window.ReactDOM' } }
      }
    },
    esbuild: {
      jsxFactory: 'window.React.createElement',
      jsxFragment: 'window.React.Fragment'
    }
  });
  console.log("Build complete: dist/index.js generated.");
}
runBuild();
```

---

## 附录 A：常见问题

### Q1：为什么插件必须在无边框窗口中运行？

无边框窗口模式提供了以下优势：
- 统一的视觉风格：自定义标题栏与 ToolBox 主窗口风格一致
- 完整的窗口控制能力（最小化、最大化、关闭）
- 插件之间完全隔离，互不影响
- 可以独立调整窗口大小，不受主窗口限制
- 关闭插件窗口不影响主应用

### Q2：为什么标题栏高度固定为 32px？

32px 是 ToolBox 主窗口标题栏的标准高度，保持一致的视觉体验。

### Q3：为什么控制按钮使用内联 SVG 而不是 Lucide 图标？

独立窗口运行在新的 Electron 窗口中，不依赖宿主的全局图标库。内联 SVG 确保按钮在任何环境下都能正常渲染，无需额外依赖。

### Q4：为什么必须使用 IIFE 格式？

ToolBox 通过动态创建 `<script>` 标签加载插件，IIFE 格式可以确保插件代码在一个独立的作用域内执行，避免全局变量污染。

### Q5：为什么 React 必须标记为 external？

React 由 ToolBox 宿主应用提供，通过 `window.React` 全局变量注入。如果插件打包时包含 React，会导致多个 React 实例冲突。

### Q6：插件如何访问宿主 API？

插件通过 `register(api)` 接收的 `api` 对象访问宿主 API，包括 `registerTool`、`registerSidebarButton`、`openPluginWindow` 等。

### Q7：插件可以访问 Electron API 吗？

不能直接访问 Electron 主进程 API。但可以通过 `window.electron.plugin` 访问窗口控制方法（`minimizeWindow`、`maximizeWindow`、`closeWindow`、`openWindow`）。

### Q8：如何在插件中使用 Tailwind CSS？

Tailwind CSS 由 `plugin-window.html` 通过 CDN 注入，插件直接使用 Tailwind 类名即可。无需在插件中配置 Tailwind。

### Q9：暗黑模式是如何实现的？

`plugin-window.html` 会自动检测系统偏好（`prefers-color-scheme`）并在 `html` 元素上添加 `.dark` class。插件使用 Tailwind 的 `dark:` 前缀即可实现暗黑模式适配。

---

*本文档随 ToolBox 项目更新，如有疑问请联系 ToolBox Team。*
