# 修复：导出文件命名方式中"时间范围模式"选项不生效

## 问题描述

在导出页面的**全局配置 → 导出文件命名方式**中，选择"时间范围模式"（date-range）后，实际导出的文件名仍然使用"简洁模式"（classic），不包含时间范围信息。该选项完全不生效。

## 根因分析

`fileNamingMode` 配置在保存（`ExportDefaultsSettingsForm` → `configService.setExportDefaultFileNamingMode()`）到实际使用（Electron 端 `ExportOrchestrator` → `normalizeFileNamingMode()`）之间的数据流中存在 4 处断裂：

### 1. `types.ts` — `ExportOptions` 接口缺少 `fileNamingMode` 字段

前端类型定义中没有该属性，导致 TypeScript 类型系统无法感知这个选项的存在，数据传递的第一步就已丢失。

### 2. `constants.ts` — `createDefaultExportOptions()` 缺少默认值

创建默认选项时未设置 `fileNamingMode`，值始终为 `undefined`。

### 3. `useExportConfig.ts` — 初始加载未读取、`updateOptions` 未持久化

- 从 config 初始化时未调用 `getExportDefaultFileNamingMode()`
- 在 `updateOptions` 中修改时未调用 `setExportDefaultFileNamingMode()` 持久化到存储

### 4. `ExportPage.tsx` — `handleExportDefaultsChanged` 丢弃了 `fileNamingMode` patch

全局配置弹窗关闭时触发的回调中，只传递了 `format` 和 `avatars` 给 `updateOptions`，`fileNamingMode` 被完全忽略。

## 修复方案

逐层打通数据流，使配置从保存 → 加载 → 传递 → 使用形成完整闭环。

## 变更列表

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `src/pages/Export/types.ts` | 新增字段 | 在 `ExportOptions` 接口中新增 `fileNamingMode: 'classic' \| 'date-range'` |
| `src/pages/Export/constants.ts` | 新增默认值 | 在 `createDefaultExportOptions()` 中新增 `fileNamingMode: 'classic'` 默认值 |
| `src/pages/Export/hooks/useExportConfig.ts` | 新增加载+持久化 | `Promise.all` 中添加 `getExportDefaultFileNamingMode()`；加载后赋值给 `newOptions.fileNamingMode`；`updateOptions` 中添加 `setExportDefaultFileNamingMode()` 持久化 |
| `src/pages/Export/ExportPage.tsx` | 补充传递 + 修复 key | `handleExportDefaultsChanged` 中添加 `fileNamingMode` 传递；同时修复了 `format` key 名错误（`defaultFormat` → `format`） |

## 验证

- 在 Windows 桌面客户端上测试通过：选择"时间范围模式"后导出，文件名正确包含日期范围（示例：`私聊_张三_20250101-20250331.json`）
- TypeScript 类型检查通过，编译无错误
