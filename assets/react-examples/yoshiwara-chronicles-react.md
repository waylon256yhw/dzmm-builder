# Yoshiwara Chronicles - React Multi-Component Version

> 吉原花街物语 - React + TypeScript + Vite 多组件架构版本

## 📌 项目概述

这是 yoshiwara-chronicles 项目的 **React 多组件重写版本**，展示了如何使用现代前端框架（React + TypeScript + Vite）开发 DZMM 应用，并最终打包成单 HTML 文件。

**与 Alpine.js 版本的对比**：
- 原版（`yoshiwara-chronicles-dzmm.html`）：84 KB 单文件，Alpine.js 实现
- React 版（本版本）：893 KB 单文件（gzip: 473 KB），完整 React 生态

## 🔗 源代码仓库

**GitHub**: https://github.com/waylon256yhw/yoshiwara-chronicles
**分支**: `dzmm-version`

## 🛠️ 技术栈

```
- React 18
- TypeScript
- Vite 5
- Tailwind CSS 3
- shadcn/ui (组件库)
- React Router (HashRouter)
- vite-plugin-singlefile (单文件打包)
```

## 📂 项目结构

```
yoshiwara-chronicles/
├── src/
│   ├── pages/              # 页面组件
│   │   ├── Welcome.tsx     # 欢迎页
│   │   ├── Character.tsx   # 角色创建
│   │   ├── Story.tsx       # 故事对话
│   │   ├── Music.tsx       # 音乐播放
│   │   └── Saves.tsx       # 存档管理
│   ├── components/         # 可复用组件
│   │   ├── RichText.tsx    # 富文本渲染
│   │   └── SnowEffect.tsx  # 雪花特效
│   ├── services/           # DZMM API 封装
│   │   └── dzmm.ts         # completions, kv 等
│   ├── lib/                # 工具函数
│   │   ├── prompts.ts      # 提示词构建
│   │   └── storage.ts      # 安全存储（处理 sandbox）
│   ├── contexts/           # React Context
│   │   └── DzmmContext.tsx # DZMM 全局状态
│   ├── types/              # TypeScript 类型
│   │   └── dzmm.ts         # DZMM 相关类型
│   └── hooks/              # 自定义 Hooks
│       └── useMessages.ts  # 消息管理
├── vite.config.ts          # Vite 配置（单文件打包）
├── package.json
└── dist-single/
    └── index.html          # 最终打包产物（893 KB）
```

## ✨ 核心特性

### 1. 分层架构设计

**Services 层**（`src/services/dzmm.ts`）：
```typescript
// DZMM API 封装
export async function initDzmm(): Promise<boolean>
export async function completions(options, callback): Promise<void>
export async function saveToCloud<T>(key, data): Promise<void>
export async function loadFromCloud<T>(key): Promise<T | null>
```

**Context 层**（`src/contexts/DzmmContext.tsx`）：
```typescript
// 全局状态管理
const { dzmmReady, sendMessage, isLoading } = useDzmm();
```

**Lib 层**（`src/lib/prompts.ts`）：
```typescript
// 提示词构建逻辑
export function buildMessages(character, messages): DzmmMessage[]
export function buildSystemPrompt(character): string
```

### 2. Sandbox 兼容处理

**localStorage 降级**（`src/lib/storage.ts`）：
```typescript
// 自动检测并降级到内存存储
export function setItem(key: string, value: string): void
export function getItem(key: string): string | null
```

**Form 提交兼容**（所有页面组件）：
```tsx
// ❌ 不使用 <form> 元素
// ✅ 使用 <div> + button onClick
<div>
  <button type="button" onClick={handleSubmit}>提交</button>
</div>
```

### 3. DZMM API 参数校验

**maxTokens 限制**（`src/services/dzmm.ts`）：
```typescript
// 自动校验并调整 maxTokens（200-3000）
if (options.maxTokens && (options.maxTokens < 200 || options.maxTokens > 3000)) {
  console.warn(`maxTokens ${options.maxTokens} 超出范围，自动调整为 3000`);
  options.maxTokens = 3000;
}
```

**连续角色检测**：
```typescript
// 检测并抛出错误，避免连续相同角色消息
for (let i = 1; i < options.messages.length; i++) {
  if (options.messages[i].role === options.messages[i - 1].role) {
    throw new Error(`消息数组格式错误：索引 ${i-1} 和 ${i} 都是 ${options.messages[i].role}`);
  }
}
```

### 4. 消息数组构建

**Emphasis 合并**（避免连续 user 消息）：
```typescript
// 将 emphasis 合并到最后一条 user 消息
const emphasis = getEmphasis();
for (let i = cleanedMessages.length - 1; i >= 0; i--) {
  if (cleanedMessages[i].role === 'user') {
    cleanedMessages[i].content =
      `<last_input>\n${cleanedMessages[i].content}\n</last_input>\n\n${emphasis}`;
    break;
  }
}
```

### 5. 完整的功能实现

- ✅ 多开场系统（夜之章、日之章）
- ✅ 消息管理（reroll、edit、delete）
- ✅ 富文本渲染（placeholder 技术处理嵌套结构）
- ✅ 多槽位存档系统（KV 云存储 + localStorage 降级）
- ✅ 响应式设计（mobile-first，Tailwind 断点）
- ✅ 流式 AI 响应（实时显示 + 自动滚动）
- ✅ 音乐播放系统（多曲目 + 循环模式）
- ✅ 雪花特效（Canvas 粒子系统）

## 🚀 构建流程

```bash
# 开发模式（热重载）
npm run dev

# 多页面构建
npm run build

# 单文件构建（DZMM 部署）
npm run build:single
```

**Vite 配置**（`vite.config.ts`）：
```typescript
import { viteSingleFile } from 'vite-plugin-singlefile';

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    mode === 'singlefile' ? viteSingleFile() : null,
  ].filter(Boolean),
  build: {
    outDir: mode === 'singlefile' ? 'dist-single' : 'dist',
  },
}));
```

## 📊 开发统计

- **总提交数**: 54 commits
- **开发分支**: `dzmm-version`
- **核心文件数**: ~30 个 TypeScript/TSX 文件
- **代码行数**: ~3500 行
- **构建产物**: 893 KB（gzip: 473 KB）

## 🔍 关键修复

本项目在迁移过程中解决了以下 DZMM 平台问题：

| 问题 | 根因 | 解决方案 |
|------|------|---------|
| localStorage sandbox 错误 | iframe 限制 | 内存降级存储 |
| Form 提交被阻止 | `allow-forms` 未开启 | 改用 button onClick |
| HTTP 400 错误 | maxTokens 超限（4000） | 遵守 200-3000 限制 |
| 连续 user 消息 | emphasis 单独追加 | 合并到最后一条 user |
| reroll 第一条消息失败 | 无上下文 | 禁止 reroll index 0 |

## 📚 相关文档

- **Q11**: React/Vue 迁移完整指南（`references/developer-guide.md`）
- **Q12**: 后端对接方法（5 个核心文件模板）
- **DZMM_DEVELOPMENT_GUIDE.md**: 项目内开发指南

## 🎯 适用场景

本示例适合以下场景：

✅ **大型项目**（>5000 行代码）
✅ **需要 TypeScript 类型安全**
✅ **需要组件化开发和维护**
✅ **团队协作开发**
✅ **需要使用 React 生态（shadcn/ui 等）**

⚠️ **不适合场景**：

- 小型单页应用（<1000 行）→ 使用 Alpine.js 更轻量
- 快速原型开发 → 直接用单文件模板

## 📝 使用建议

1. **克隆仓库并切换分支**：
   ```bash
   git clone https://github.com/waylon256yhw/yoshiwara-chronicles.git
   git checkout dzmm-version
   npm install
   ```

2. **本地开发**：
   ```bash
   npm run dev
   # 访问 http://localhost:5173
   ```

3. **构建单文件**：
   ```bash
   npm run build:single
   # 产物：dist-single/index.html
   ```

4. **上传到 DZMM**：
   - 进入 DZMM 工坊
   - 上传 `dist-single/index.html`
   - 测试 sandbox 兼容性

## 💡 学习要点

通过这个示例，你可以学到：

1. **如何用 React 开发 DZMM 应用**（保持组件化优势）
2. **如何处理 sandbox 限制**（localStorage、form）
3. **如何封装 DZMM API**（类型安全、参数校验）
4. **如何构建消息数组**（避免格式错误）
5. **如何实现复杂业务逻辑**（多开场、消息管理、存档）
6. **如何优化响应式设计**（mobile-first）
7. **如何使用 Vite 插件**（单文件打包）

---

**最后更新**: 2025-01
**维护状态**: ✅ 活跃维护
**许可协议**: MIT
