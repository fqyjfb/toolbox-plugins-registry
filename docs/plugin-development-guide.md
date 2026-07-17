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
- **自动构建产物**：插件安装时直接使用已构建的 `dist/index.js`，不执行 `pnpm install` 或 `pnpm run build`
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
| Lucide React | 1.x | 图标库 |
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
  "isBeta": false
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
    "lucide-react": "^1.8.0"
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

1. 安装依赖：`pnpm install`
2. 构建：`pnpm run build`
3. 提交构建产物：将 `dist/index.js` 提交到 Git

### GitHub Actions 自动构建

创建 `.github/workflows/release.yml`：

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 9

      - name: Install dependencies
        run: pnpm install

      - name: Build
        run: pnpm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: plugin-build
          path: dist/

  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: plugin-build
          path: dist/

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/index.js
```

### 发布流程

1. 更新 `manifest.json` 和 `package.json` 中的版本号
2. 构建：`pnpm run build`
3. 提交代码：`git add -A && git commit -m "release: v1.0.0"`
4. 推送：`git push origin main`
5. 创建标签：`git tag v1.0.0 && git push origin v1.0.0`
6. 在 `toolbox-plugins-registry` 中注册插件

---

## 💡 完整示例

### 创建一个简单的"待办事项"插件

#### 1. 创建项目结构

```bash
mkdir plugin-todo && cd plugin-todo
pnpm init -y
pnpm add react react-dom lucide-react
pnpm add -D typescript @types/react @types/react-dom vite
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
  "entry": "dist/index.js"
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
pnpm run build
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
   pnpm run electron:dev
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