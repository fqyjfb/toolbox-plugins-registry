# ToolBox 插件注册表

ToolBox 插件商店的官方插件注册表仓库，用于集中管理所有可用插件的元数据信息。

## 仓库结构

```
toolbox-plugins-registry/
├── registry.json          # 插件注册表主文件
├── plugins/               # 插件截图等资源
│   ├── plugin-translate/
│   ├── plugin-color-picker/
│   └── ...
└── README.md              # 仓库说明文档
```

## 注册表格式

`registry.json` 是一个 JSON 数组，每个元素代表一个可用插件：

```json
{
  "id": "plugin-example",
  "name": "插件名称",
  "description": "插件简短描述",
  "icon": "Lucide图标名称",
  "color": "#00BCD4",
  "textColor": "#ffffff",
  "version": "1.0.0",
  "author": "作者",
  "categories": ["分类"],
  "tags": ["标签"],
  "isBeta": false
}
```

## 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 插件唯一标识符，格式：`plugin-{name}` |
| `name` | string | 是 | 插件显示名称 |
| `description` | string | 是 | 插件简短描述 |
| `icon` | string | 是 | Lucide 图标名称 |
| `color` | string | 是 | 插件图标背景色（HEX） |
| `textColor` | string | 是 | 插件图标文字颜色（HEX） |
| `version` | string | 是 | 插件版本号 |
| `author` | string | 是 | 插件作者 |
| `categories` | string[] | 是 | 插件分类数组 |
| `tags` | string[] | 否 | 插件标签数组 |
| `githubRepo` | string | 否 | 插件 GitHub 仓库路径 |
| `entry` | string | 否 | 插件入口文件路径 |
| `isBeta` | boolean | 否 | 是否为 Beta 版本 |

## 添加新插件

1. 编辑 `registry.json` 文件
2. 在数组末尾添加新插件对象
3. 提交并推送到 GitHub

## 许可证

MIT License