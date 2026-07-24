# ToolBox 插件开发规范

本文档详细描述了如何开发 ToolBox 插件商店的插件，包括项目结构、技术栈、API 参考、样式规范等。

---

## 📋 目录

- [概述](#-概述)
- [技术栈](#-技术栈)
- [项目结构](#-项目结构)
- [核心文件详解](#-核心文件详解)
- [插件 API 参考](#-插件-api-参考)
- [样式规范](#-样式规范)
- [主题适配](#-主题适配)
- [构建与发布](#-构建与发布)
- [完整示例](#-完整示例)
- [调试指南](#-调试指南)

---

## 🎯 概述

ToolBox 插件是独立的 React 应用，运行在独立的 Electron 窗口中。插件通过 `manifest.json` 声明元数据，通过 Vite 构建为 IIFE 格式，最终在 ToolBox 插件商店中展示和安装。

### 核心特点

- **独立窗口运行**：插件在独立的 Electron BrowserWindow 中运行，不影响主窗口
- **可配置窗口尺寸**：插件通过 `manifest.json` 中的 `width` 和 `height` 字段声明窗口初始尺寸，默认 `800 × 600`，最小限制 `400 × 300`
- **自动构建产物**：插件安装时直接使用 GitHub Release 中已构建的 zip 包，不执行 `npm install` 或构建命令
- **主题跟随**：插件窗口自动继承主应用的主题模式（浅色/深色）
- **API 隔离**：通过 preload 脚本暴露有限的 Electron API，确保安全性

---

## 🛠️ 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 19.x | UI 框架 |
| TypeScript | 6.x | 类型安全 |
| Vite | 5.x | 构建工具 |
| Tailwind CSS | 3.x | 样式框架（CDN） |
| Lucide React | 0.454.x | 图标库 |
| Electron | 42.x | 桌面端运行时 |

---

## 📁 项目结构

```
plugin-example/
├── src/
│   ├── index.tsx          # 入口文件，包含渲染逻辑和插件注册
│   └── ToolPanel.tsx      # 主界面组件（建议拆分）
├── dist/
│   └── index.js           # 构建产物（git 跟踪）
├── .github/
│   └── workflows/
│       └── release.yml     # GitHub Actions 自动构建
├── .gitignore
├── README.md
├── build.mjs              # Vite 构建配置
├── manifest.json          # 插件元数据
├── package.json           # 依赖配置
└── tsconfig.json          # TypeScript 配置
```

---

## 📝 核心文件详解

### 1. manifest.json

插件元数据文件，必须与注册表中的信息一致：

```json
{
  "id": "plugin-example",
  "name": "示例插件",
  "version": "1.0.0",
  "description": "插件功能描述，不超过 100 字",
  "author": "Author Name",
  "icon": "Package",
  "iconUrl": "https://raw.githubusercontent.com/fqyjfb/toolbox-plugins-registry/main/icons/plugin-example/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "categories": ["工具"],
  "tags": ["example", "demo"],
  "githubRepo": "fqyjfb/plugin-example",
  "entry": "dist/index.js",
  "isBeta": false,
  "width": 800,
  "height": 600
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，格式 `plugin-{name}`，小写 kebab-case |
| `name` | string | 是 | 插件中文名，显示在 UI 中 |
| `version` | string | 是 | 语义化版本号，格式 `x.y.z` |
| `description` | string | 是 | 功能描述，不超过 100 字 |
| `author` | string | 是 | 作者名 |
| `icon` | string | 是 | Lucide 图标名，如 `Palette`、`Camera`、`Package` |
| `iconUrl` | string | 否 | 插件图标 URL |
| `color` | string | 是 | 主色调，HEX 格式 |
| `textColor` | string | 否 | 图标文字色，默认 `#ffffff` |
| `categories` | string[] | 是 | 分类标签数组 |
| `tags` | string[] | 否 | 搜索标签，小写英文 |
| `githubRepo` | string | 是 | GitHub 仓库地址，格式 `{owner}/{repo}` |
| `entry` | string | 是 | 入口文件路径，默认 `dist/index.js` |
| `isBeta` | boolean | 否 | 是否为测试版，默认 `false` |
| `width` | number | 否 | 插件窗口宽度（像素），默认 `800` |
| `height` | number | 否 | 插件窗口高度（像素），默认 `600` |

### 窗口尺寸说明

`width` 和 `height` 字段由插件自身声明，控制插件窗口的初始尺寸。如果未指定，将使用默认值 `800 × 600`。

- 窗口尺寸的最小限制为 `400 × 300`（由 ToolBox 主程序强制），即使插件声明小于该值也会被覆盖
- 建议根据插件 UI 复杂度选择合适的初始尺寸，如文件管理器类插件可设为 `1200 × 800`
- 用户可在运行时手动调整窗口大小

### 2. package.json

依赖配置，注意 React 和 ReactDOM 在构建时会打包到产物中：

```json
{
  "name": "plugin-example",
  "version": "1.0.0",
  "description": "示例插件",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "build": "node build.mjs"
  },
  "dependencies": {
    "react": "^19.2.0",
    "react-dom": "^19.2.0",
    "lucide-react": "^0.454.0"
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

### 3. build.mjs

Vite 构建配置，必须输出为 IIFE 格式：

```javascript
import { build } from 'vite';
import { fileURLToPath } from 'url';
import path from 'path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

async function runBuild() {
  await build({
    root: __dirname,
    build: {
      outDir: path.resolve(__dirname, 'dist'),
      emptyOutDir: true,
      minify: false,
      rollupOptions: {
        input: path.resolve(__dirname, 'src/index.tsx'),
        output: { 
          dir: path.resolve(__dirname, 'dist'),
          format: 'iife',
          entryFileNames: 'index.js'
        }
      }
    }
  });
  console.log("Build complete: dist/index.js generated.");
}
runBuild();
```

### 4. tsconfig.json

TypeScript 配置：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

### 5. src/index.tsx

入口文件，必须包含渲染逻辑和插件注册：

```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import ToolPanel from './ToolPanel';

const PluginApp: React.FC = () => {
  return React.createElement(ToolPanel);
};

function renderStandalone() {
  const root = document.getElementById('root');
  if (!root) return;

  if (ReactDOM.createRoot) {
    ReactDOM.createRoot(root).render(React.createElement(PluginApp));
  } else {
    ReactDOM.render(React.createElement(PluginApp), root);
  }
}

function registerPlugin(api: any) {
  const { registerTool, registerSidebarButton, openPluginWindow } = api;

  registerTool({
    id: 'plugin-example',
    name: '示例插件',
    iconName: 'Package',
    color: '#3b82f6',
    textColor: '#ffffff',
    path: '/tools/plugin-example',
    component: ToolPanel,
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

const pluginData = (window as any).__PLUGIN_DATA__;

if (pluginData) {
  renderStandalone();
}
```

---

## 🔌 插件 API 参考

### window.__PLUGIN_DATA__

插件启动时注入的插件数据：

```typescript
interface PluginData {
  pluginId: string;          // 插件 ID
  pluginName: string;        // 插件名称
  manifest: PluginManifest;  // 完整的 manifest 对象
  isDark: boolean;           // 当前主题是否为深色模式
}
```

### window.electron

通过 preload 脚本暴露的 Electron API：

#### 插件窗口控制

```typescript
window.electron.plugin.minimizeWindow();  // 最小化窗口
window.electron.plugin.maximizeWindow();  // 最大化/还原窗口
window.electron.plugin.closeWindow();     // 关闭窗口
```

#### 文件管理器

```typescript
// 获取系统路径
const paths = await window.electron.fileManager.getSystemPaths();

// 获取特定类型路径
const desktopPath = await window.electron.fileManager.getPath('desktop');

// 列出目录文件
const files = await window.electron.fileManager.listFiles(path);

// 打开文件
await window.electron.fileManager.openFile(filePath);

// 复制文件
const count = await window.electron.fileManager.copyFiles(sourcePaths, destPath);
```

#### 截图

```typescript
// 开始截图捕获
await window.electron.screenshot.startCapture();

// 取消截图
window.electron.screenshot.cancel();

// 保存截图
await window.electron.screenshot.save({ buffer: dataUrl, format: 'png' });

// 复制到剪贴板
window.electron.screenshot.copyToClipboard(dataUrl);
```

#### 其他常用 API

```typescript
// 打开外部链接
window.electron.openExternal(url);

// 获取应用数据目录
const userDataPath = await window.electron.getUserDataPath();

// 获取版本信息
const version = await window.electron.getVersion();
```

### window.toolboxApi

ToolBox 插件 API，用于注册工具和侧边栏按钮：

```typescript
const { registerTool, registerSidebarButton, openPluginWindow } = window.toolboxApi;

// 注册工具
registerTool({
  id: 'plugin-example',
  name: '示例插件',
  iconName: 'Package',
  color: '#3b82f6',
  textColor: '#ffffff',
  path: '/tools/plugin-example',
  component: ToolPanel,
});

// 注册侧边栏按钮
registerSidebarButton({
  id: 'plugin-example-btn',
  icon: 'Package',
  label: '示例插件',
  onClick: () => {
    openPluginWindow?.('plugin-example');
  },
});
```

---

## 🎨 样式规范

### 设计系统

插件必须遵循 ToolBox 的设计系统：

#### 色彩系统

```css
/* 背景色 */
.bg-white              /* #ffffff */
.bg-gray-50            /* #f9fafb */
.bg-gray-800           /* #1f2937 */
.bg-gray-900           /* #111827 */

/* 文字色 */
.text-gray-900         /* #111827 - 主文字 */
.text-gray-600         /* #6b7280 - 次要文字 */
.text-gray-400         /* #9ca3af - 辅助文字 */

/* 边框色 */
.border-gray-200       /* #e5e7eb */
.border-gray-700       /* #374151 */
```

**禁用**：
- 蓝紫渐变（`linear-gradient` 包含 purple/violet/indigo）
- 霓虹色、彩虹配色
- 玻璃拟态效果
- 表情符号作为功能图标

#### 排版

| 尺寸 | CSS 变量 | 用途 |
|------|----------|------|
| 12px | `--text-xs` | 辅助文字 |
| 14px | `--text-sm` | 正文、按钮文字 |
| 16px | `--text-base` | 标准正文 |
| 20px | `--text-lg` | 小标题 |
| 24px | `--text-xl` | 标题 |
| 32px | `--text-2xl` | 大标题 |

- 行高：正文 1.5，标题 1.25
- 字重：正文 400，标题 600

#### 间距

基于 4px 网格系统：

| 变量 | 值 | 用途 |
|------|-----|------|
| `--space-1` | 4px | 紧凑间距 |
| `--space-2` | 8px | 小间距 |
| `--space-3` | 12px | 中等间距 |
| `--space-4` | 16px | 标准间距 |
| `--space-6` | 24px | 较大间距 |
| `--space-8` | 32px | 大间距 |

**禁用**：魔法数字（13px、7px、23px 等）

#### 组件规范

**卡片**：
- 使用边框或阴影，不可同时使用
- 边框：`border border-gray-200 dark:border-gray-700`
- 阴影层级 1：`shadow-md`（0 1px 3px rgba(0,0,0,0.08)）
- 圆角：6px 或 8px

**按钮**：
- 主按钮：实心填充 `bg-primary text-white`
- 次按钮：描边 `border border-gray-300 text-gray-700`
- 悬停效果：加深 10%，不切换颜色
- 禁用全圆角（`rounded-full`）

**输入框**：
- 边框：`border border-gray-300`
- 圆角：6px
- 聚焦：边框色变化 + outline，无 glow

---

## 🌓 主题适配

插件窗口会自动继承主应用的主题模式，在 HTML 根元素和 body 上添加 `dark` class。

### 实现方式

使用 Tailwind CSS 的 `dark:` 前缀类：

```tsx
<div className="bg-white dark:bg-gray-800">
  <h1 className="text-gray-900 dark:text-gray-200">标题</h1>
  <p className="text-gray-600 dark:text-gray-400">正文</p>
  <button className="bg-gray-100 dark:bg-gray-700 hover:bg-gray-200 dark:hover:bg-gray-600">
    按钮
  </button>
</div>
```

### 主题检测

可以通过 `window.__PLUGIN_DATA__.isDark` 获取当前主题状态：

```typescript
const pluginData = (window as any).__PLUGIN_DATA__;
if (pluginData?.isDark) {
  // 深色模式逻辑
}
```

### 插件窗口头部

插件窗口的标题栏由 ToolBox 自动生成，已包含主题适配，插件无需处理。

---

## 🚀 构建与发布

### 构建流程

1. 安装依赖：`npm install`
2. 构建：`node build.mjs`
3. 提交构建产物：将 `dist/index.js` 和 `package-lock.json` 提交到 Git（**必须提交，插件安装时直接使用构建产物**）

### 构建产物管理

**重要**：插件安装时不执行 `npm install` 或构建命令，而是直接使用 GitHub Release 中已构建的 zip 包。因此：

- **dist/index.js 必须提交到 Git**：确保 `.gitignore` 中不排除 `dist/` 目录
- **package-lock.json 必须提交到 Git**：CI 使用 `npm ci` 安装依赖，必须有 lockfile
- **禁止使用 pnpm-lock.yaml**：项目统一使用 npm，避免依赖管理器冲突
- **构建产物跟随代码一起推送**：每次代码变更后需重新构建并推送构建产物
- **GitHub Actions 自动构建**：推荐配置 Actions 工作流，在推送 tag 时自动构建并上传到 Release

### GitHub Actions 自动构建

创建 `.github/workflows/release.yml`：

```yaml
name: Create Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '24.x'

      - name: Install dependencies
        run: npm ci

      - name: Build plugin
        run: node build.mjs

      - name: Create zip archive
        run: zip -r plugin-{name}.zip dist manifest.json

      - name: Create Release
        uses: ncipollo/release-action@v1
        with:
          name: Release ${{ github.ref_name }}
          tag: ${{ github.ref_name }}
          body: |
            ## Changes
            - Build dist files
            - Update plugin manifest
          draft: false
          prerelease: false
          artifacts: 'plugin-{name}.zip'
```

#### 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `permissions: contents: write` | **必须配置**，确保有权限创建 Release |
| `node-version: '24.x'` | 使用 Node.js 24 版本，与开发环境一致 |
| `npm ci` | 使用 npm ci 安装依赖，**必须提交 `package-lock.json`** |
| `node build.mjs` | 直接执行构建脚本，不经过 npm 脚本 |
| `zip -r plugin-{name}.zip dist manifest.json` | 将构建产物和 manifest 打包为 zip |
| `ncipollo/release-action@v1` | 创建正式的 GitHub Release，自动上传 zip 包 |

#### 常见错误排查

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| `npm ci` 失败 | 缺少 `package-lock.json` | 执行 `npm install` 生成并提交 lockfile |
| `contents.write` 权限不足 | 未配置 `permissions` | 添加 `permissions: contents: write` |
| Release 创建失败 | 使用了错误的 Action | 使用 `ncipollo/release-action@v1` |
| 构建产物缺失 | `.gitignore` 排除了 `dist/` | 确保 `dist/` 目录在 git 跟踪中 |
| `@lucide/shared` 404 | lucide-react 版本过高 | 使用 `^0.454.0` 版本 |

### 分支管理规范

插件仓库必须使用 `main` 作为主分支，禁止使用 `master`。

| 分支类型 | 命名格式 | 用途 |
|----------|----------|------|
| 主分支 | `main` | 稳定版本，包含已构建的 `dist/index.js` |
| 功能分支 | `feature/{feature-name}` | 开发新功能 |
| 修复分支 | `hotfix/{issue-name}` | 紧急修复 |

**创建仓库时**：确保 GitHub 仓库默认分支为 `main`

```bash
# 初始化时使用 main 分支
git init -b main

# 若已有 master 分支，重命名为 main
git branch -m master main
git push -u origin main
```

### 发布流程

1. 更新 `manifest.json` 和 `package.json` 中的版本号
2. 构建：`node build.mjs`
3. **提交构建产物**：`git add -A`（确保 `dist/index.js` 和 `package-lock.json` 包含在内）
4. **提交代码**：`git commit -m "release: v1.0.0"`
5. **推送代码和构建产物到 main 分支**：`git push origin main`（构建产物必须跟随代码一起推送）
6. 创建标签触发 Release：`git tag v1.0.0 && git push origin v1.0.0`
7. 在 `toolbox-plugins-registry` 中注册插件

### 注册表更新

插件发布后，需要在 `toolbox-plugins-registry` 的 `registry.json` 中添加插件条目：

```json
{
  "id": "plugin-example",
  "name": "示例插件",
  "version": "1.0.0",
  "description": "插件功能描述",
  "icon": "Package",
  "iconUrl": "https://raw.githubusercontent.com/fqyjfb/toolbox-plugins-registry/main/icons/plugin-example/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "author": "Author Name",
  "entry": "dist/index.js",
  "categories": ["工具"],
  "tags": ["example", "demo"],
  "githubRepo": "fqyjfb/plugin-example",
  "isBeta": false,
  "width": 800,
  "height": 600
}
```

注册表更新后，需将 `toolbox-plugins-registry` 仓库推送到 GitHub，以便插件商店获取最新插件列表。

---

## 💡 完整示例

### 创建一个简单的"待办事项"插件

#### 1. 创建项目结构

```bash
mkdir plugin-todo && cd plugin-todo
npm init -y
npm install react react-dom lucide-react@^0.454.0
npm install -D typescript @types/react @types/react-dom vite
```

#### 2. 创建配置文件

**manifest.json**：

```json
{
  "id": "plugin-todo",
  "name": "待办事项",
  "version": "1.0.0",
  "description": "简单的待办事项管理工具",
  "author": "ToolBox Team",
  "icon": "CheckSquare",
  "color": "#059669",
  "textColor": "#ffffff",
  "categories": ["工具"],
  "tags": ["todo", "task", "list"],
  "githubRepo": "fqyjfb/plugin-todo",
  "entry": "dist/index.js",
  "width": 400,
  "height": 600
}
```

**build.mjs** 和 **tsconfig.json**：参考上方模板

#### 3. 创建组件

**src/ToolPanel.tsx**：

```tsx
import React, { useState, useCallback } from 'react';
import { Plus, Check, Trash2 } from 'lucide-react';

interface TodoItem {
  id: string;
  text: string;
  completed: boolean;
}

const ToolPanel: React.FC = () => {
  const [todos, setTodos] = useState<TodoItem[]>([]);
  const [inputValue, setInputValue] = useState('');

  const addTodo = useCallback(() => {
    if (!inputValue.trim()) return;
    const newTodo: TodoItem = {
      id: Date.now().toString(),
      text: inputValue.trim(),
      completed: false,
    };
    setTodos((prev) => [...prev, newTodo]);
    setInputValue('');
  }, [inputValue]);

  const toggleTodo = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((todo) =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }, []);

  const deleteTodo = useCallback((id: string) => {
    setTodos((prev) => prev.filter((todo) => todo.id !== id));
  }, []);

  return (
    <div className="h-full flex flex-col bg-white dark:bg-gray-800 p-4">
      <div className="flex items-center gap-3 mb-4">
        <CheckSquare className="w-6 h-6 text-primary" />
        <h2 className="text-lg font-semibold text-gray-800 dark:text-gray-200">
          待办事项
        </h2>
      </div>

      <div className="flex gap-2 mb-4">
        <input
          type="text"
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && addTodo()}
          className="flex-1 px-3 py-2 border border-gray-300 dark:border-gray-600 rounded-lg bg-white dark:bg-gray-700 text-gray-800 dark:text-gray-200 focus:outline-none focus:border-primary"
          placeholder="添加新任务..."
        />
        <button
          onClick={addTodo}
          className="px-4 py-2 bg-primary text-white rounded-lg hover:bg-primary/90 transition-colors"
        >
          <Plus className="w-5 h-5" />
        </button>
      </div>

      <div className="flex-1 overflow-y-auto space-y-2">
        {todos.length === 0 ? (
          <div className="flex flex-col items-center justify-center h-full text-gray-500 dark:text-gray-400">
            <CheckSquare className="w-12 h-12 mb-3 opacity-50" />
            <p className="text-sm">暂无待办事项</p>
          </div>
        ) : (
          todos.map((todo) => (
            <div
              key={todo.id}
              className="flex items-center gap-3 p-3 bg-gray-50 dark:bg-gray-700 rounded-lg"
            >
              <button
                onClick={() => toggleTodo(todo.id)}
                className={`w-5 h-5 rounded border-2 flex items-center justify-center transition-colors ${
                  todo.completed
                    ? 'bg-primary border-primary text-white'
                    : 'border-gray-300 dark:border-gray-600 hover:border-primary'
                }`}
              >
                {todo.completed && <Check className="w-3 h-3" />}
              </button>
              <span
                className={`flex-1 text-sm ${
                  todo.completed
                    ? 'text-gray-400 line-through'
                    : 'text-gray-800 dark:text-gray-200'
                }`}
              >
                {todo.text}
              </span>
              <button
                onClick={() => deleteTodo(todo.id)}
                className="p-1 text-gray-400 hover:text-red-500 transition-colors"
              >
                <Trash2 className="w-4 h-4" />
              </button>
            </div>
          ))
        )}
      </div>
    </div>
  );
};

export default ToolPanel;
```

**src/index.tsx**：参考上方模板，将 ID 和名称替换为 `plugin-todo`

#### 4. 构建并发布

```bash
node build.mjs
git add -A
git commit -m "feat: initial release"
git push origin main
git tag v1.0.0
git push origin v1.0.0
```

---

## 🐛 调试指南

### 本地开发

1. 将插件代码复制到 ToolBox 的扩展目录：
   - Windows: `%APPDATA%\toolbox\extensions\plugin-example`
   - macOS: `~/Library/Application Support/toolbox/extensions/plugin-example`

2. 启动 ToolBox 开发模式：
   ```bash
   cd ToolBox
   npm run electron:dev
   ```

3. 在插件商店中找到插件并启动

### 调试技巧

- 使用 `console.log` 输出调试信息，会显示在开发者工具中
- 通过 `window.__PLUGIN_DATA__` 查看插件初始化数据
- 使用 `window.electron` API 前检查是否可用

### 常见问题

**Q: 插件窗口打开后空白？**
A: 检查 `dist/index.js` 是否存在，确保构建成功。查看开发者工具控制台是否有错误。

**Q: React 未定义？**
A: 确保构建配置正确，React 和 ReactDOM 应打包到产物中。检查 `build.mjs` 中是否使用了 `format: 'iife'`。

**Q: 主题不生效？**
A: 确保使用了 `dark:` 前缀的 Tailwind 类。检查插件窗口 HTML 是否包含 `dark` class。

**Q: Electron API 不可用？**
A: 确保通过 `window.electron` 访问，而非直接使用 `require`。检查 preload 脚本是否正确配置。

---

## 📜 许可证

插件默认使用 MIT License，与 ToolBox 保持一致。