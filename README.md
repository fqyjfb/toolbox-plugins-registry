# ToolBox 插件注册表

ToolBox 插件商店的官方插件注册表仓库，用于集中管理所有可用插件的元数据信息。

## 📁 仓库结构

```
toolbox-plugins-registry/
├── registry.json          # 插件注册表主文件
├── icons/                 # 插件图标资源
│   ├── screenshot-plugin/
│   │   └── icon.png
│   ├── color-palette-plugin/
│   │   └── icon.png
│   ├── file-manager-plugin/
│   │   └── icon.png
│   └── ...
├── docs/                  # 文档目录
│   └── plugin-development-guide.md  # 插件开发规范
└── README.md              # 仓库说明文档
```

## 📋 注册表格式

`registry.json` 是一个 JSON 数组，每个元素代表一个可用插件：

```json
{
  "id": "plugin-example",
  "name": "插件名称",
  "description": "插件功能描述，不超过 100 字",
  "icon": "Lucide图标名称",
  "iconUrl": "https://raw.githubusercontent.com/fqyjfb/toolbox-plugins-registry/main/icons/plugin-example/icon.png",
  "color": "#3b82f6",
  "textColor": "#ffffff",
  "version": "1.0.0",
  "author": "作者",
  "entry": "dist/index.js",
  "categories": ["工具"],
  "tags": ["tag1", "tag2"],
  "githubRepo": "fqyjfb/plugin-example",
  "isBeta": false
}
```

## 📝 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 唯一标识符，格式 `plugin-{name}` |
| `name` | string | 是 | 插件中文名，显示在 UI 中 |
| `description` | string | 是 | 功能描述，不超过 100 字 |
| `icon` | string | 是 | Lucide 图标名，如 `Palette`、`Camera`、`Package` |
| `iconUrl` | string | 否 | 插件图标 URL，存储在注册表 `icons/{plugin-name}/icon.png` |
| `color` | string | 是 | 主色调，HEX 格式 |
| `textColor` | string | 否 | 图标文字色，默认 `#ffffff` |
| `version` | string | 是 | 语义化版本号，格式 `x.y.z` |
| `author` | string | 是 | 作者名 |
| `entry` | string | 是 | 入口文件路径（相对于插件目录），默认 `dist/index.js` |
| `categories` | string[] | 是 | 分类标签数组 |
| `tags` | string[] | 否 | 搜索标签，小写英文，逗号分隔 |
| `githubRepo` | string | 是 | GitHub 仓库地址，格式 `{owner}/{repo}` |
| `isBeta` | boolean | 否 | 是否为测试版，默认 `false` |

### 分类值参考

当前 ToolBox 支持的分类：

- `工具` - 通用工具类
- `设计` - 设计相关工具
- `开发` - 开发辅助工具
- `文件` - 文件处理工具
- `文本` - 文本处理工具
- `数据` - 数据格式化工具
- `AI` - AI 相关工具
- `系统` - 系统管理工具

## 🚀 插件发布流程

1. 完成插件开发并通过测试
2. 提交到 GitHub 仓库
3. 向 `toolbox-plugins-registry` 仓库提交 PR：
   - 更新 `registry.json`，添加插件条目
   - 上传图标到 `icons/{plugin-name}/icon.png`
4. PR 审核通过后合并到 `main` 分支

### 图标规范

| 属性 | 要求 |
|------|------|
| 尺寸 | 64x64 px（最小），128x128 px（推荐） |
| 格式 | PNG（带透明背景） |
| 文件路径 | `icons/{plugin-name}/icon.png` |
| 命名 | 小写 kebab-case |
| 风格 | 简洁、扁平化、与 ToolBox 品牌一致 |

## 🔌 已注册插件

| 插件 ID | 名称 | 描述 | 分类 | GitHub |
|---------|------|------|------|--------|
| `plugin-screenshot` | 截图工具 | 原生多显示器截图捕获、标注和裁剪工具 | 工具 | [fqyjfb/Screenshot-plugin](https://github.com/fqyjfb/Screenshot-plugin) |
| `plugin-color-palette` | 调色板 | 颜色选择、多格式转换与配色方案生成工具 | 设计、工具 | [fqyjfb/color-palette-plugin](https://github.com/fqyjfb/color-palette-plugin) |
| `plugin-file-manager` | 文件管理 | 文件管理工具，支持浏览系统文件、收藏常用路径、拖拽复制文件 | 文件、工具 | [fqyjfb/file-manager-plugin](https://github.com/fqyjfb/file-manager-plugin) |

## 📖 开发文档

- [插件开发规范](docs/plugin-development-guide.md) - 详细的插件开发指南，包含架构概述、技术栈、项目结构、API 参考、样式规范、主题适配等

## 📜 许可证

MIT License
