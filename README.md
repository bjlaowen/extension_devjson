# DevJSON

面向开发者的多语言数据结构转换 Chrome 扩展，点击图标在新标签页打开，无需联网，数据本地处理。

## 功能

**JSON 处理**

| 标签页 | 说明 |
|---|---|
| 格式化 | JSON 格式化 / 压缩，显示类型与字节大小 |
| 路径提取 | 树形展示 JSON，点击任意字段生成访问路径 |
| JSON Diff | 对比两段 JSON，输出新增 / 删除 / 变化字段 |

**代码生成（JSON → 目标语言）**

| 标签页 | 说明 |
|---|---|
| JSON 转 Go Struct | 生成带 `json` tag 的 Go 结构体 |
| JSON 转 Proto | 生成 proto3 message，支持 package 设置 |
| JSON 转 TS Interface | 生成 TypeScript Interface，嵌套类型自动拆分 |
| JSON 转 Java Class | 生成带 `@JsonProperty` 注解和 getter/setter 的 Java 类 |
| JSON 转 C# Model | 生成带 `[JsonPropertyName]` 的 C# 属性类 |
| JSON 转 Python Dataclass | 生成 `@dataclass` 类，子类自动前置 |

**反转生成（目标语言 → JSON）**

| 标签页 | 说明 |
|---|---|
| Go Struct 转 JSON | 解析 Go 结构体，生成字段默认值 JSON 示例 |
| Proto 转 JSON | 解析 proto message，生成字段默认值 JSON 示例 |

侧边栏支持自定义结构体 / Message 名称、Proto package，并保留最近 20 条本地历史记录。

## 技术栈

- Vue 3 + Vite
- Chrome Extension Manifest V3

## 开发

```bash
npm install
npm run build
```

构建产物在 `dist/` 目录。

## 加载扩展

1. 打开 Chrome，进入 `chrome://extensions/`
2. 开启右上角「开发者模式」
3. 点击「加载已解压的扩展程序」，选择 `dist/` 目录
4. 点击工具栏图标，将在新标签页打开
