# DevJSON

程序员 JSON / Go Struct / Proto 转换调试工具，以 Chrome 扩展形式运行，点击图标在新标签页打开。

## 功能

| 标签页 | 说明 |
|---|---|
| 格式化 | JSON 格式化 / 压缩 |
| 路径提取 | 树形展示，点击字段生成访问路径 |
| JSON 转 Go Struct | 根据 JSON 生成对应 Go 结构体 |
| JSON 转 Proto | 根据 JSON 生成 proto3 message |
| Go Struct 转 JSON | 将 Go 结构体反转为 JSON 示例 |
| Proto 转 JSON | 将 proto message 反转为 JSON 示例 |
| JSON Diff | 对比两段 JSON，输出新增 / 删除 / 变化字段 |

侧边栏支持设置结构体名称、Proto package，以及查看最近 20 条本地历史记录。

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
