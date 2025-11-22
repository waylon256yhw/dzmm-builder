# DZMM.AI 同层交互页面开发者指南

> 基于逆向工程分析的完整开发文档
> 版本：1.0 | 更新日期：2025-01

---

## 📋 目录

1. [平台简介](#平台简介)
2. [核心专有接口](#核心专有接口)
3. [标准开发流程](#标准开发流程)
4. [三种工作流架构](#三种工作流架构)
5. [最佳实践](#最佳实践)
6. [常见问题](#常见问题)

---

## 🎯 平台简介

**DZMM.AI** 是一个支持 AI 驱动的交互式 Web 应用平台，提供：
- ✅ 流式 AI 对话生成
- ✅ 键值对云端存储
- ✅ 轻量级状态管理
- ✅ 单 HTML 文件部署

### 技术栈推荐
- **前端框架**：Alpine.js 3.x（轻量级响应式）
- **AI 接口**：window.dzmm（平台专有）
- **样式方案**：原生 CSS（支持渐变、动画）

---

## 🔌 核心专有接口

### 1. 初始化：等待 API 就绪

**所有应用必须先等待平台就绪**，否则接口调用会失败。

```javascript
// 标准初始化模式
async function init() {
  await new Promise((resolve) => {
    const handler = (event) => {
      if (event.data?.type === 'dzmm:ready') {
        window.removeEventListener('message', handler);
        resolve();
      }
    };
    window.addEventListener('message', handler);
  });

  console.log('DZMM API 已就绪');
  // 此处可以开始调用其他接口
}
```

**关键点**：
- ✅ 使用 `window.addEventListener('message', handler)` 监听
- ✅ 检查 `event.data?.type === 'dzmm:ready'`
- ✅ 使用 Promise 包装，支持 async/await
- ❌ 不要在就绪前调用 `window.dzmm.*` 接口

---

### 2. AI 生成：`window.dzmm.completions()`

**用途**：调用 AI 模型生成内容（支持流式输出）

#### 接口签名

```typescript
window.dzmm.completions(
  config: {
    model: string;           // 模型名称
    messages: Message[];     // 对话历史
    maxTokens: number;       // 最大生成长度
    headers?: Record<string, string>; // 可选：附加请求头
  },
  callback: (newContent: string, done: boolean) => void
): Promise<void>

interface Message {
  role: 'user' | 'assistant';
  content: string;
}
```

#### 参数说明

| 参数 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `model` | string | 模型名称 | `'nalang-xl'` 或 `'nalang-xl-10'` |
| `messages` | Message[] | 对话历史（按时间顺序） | `[{role: 'user', content: '你好'}]` |
| `maxTokens` | number | 最大生成 token 数 | 1500-3000 |
| `headers` | Record<string,string> | （可选）附加 HTTP 头 | `{ 'x-dzmm-user': 'local-dev' }` |
| `callback` | function | 流式回调函数 | 见下方示例 |

> ⚠️ `Message.role` 目前仅接受 `'user'` / `'assistant'`，传入 `'system'` 将导致 HTTP 400。

#### 回调函数参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `newContent` | string | **累积的完整内容**（不是增量） |
| `done` | boolean | 是否生成完毕 |

#### 完整示例

```javascript
let content = '';

await window.dzmm.completions(
  {
    model: 'nalang-xl-10',
    messages: [
      { role: 'user', content: '讲一个笑话' }
    ],
    maxTokens: 1500
  },
  (newContent, done) => {
    // 实时更新显示内容
    content = newContent;
    document.getElementById('output').textContent = content;

    if (done) {
      console.log('生成完成:', content);
      // 可以在这里保存、处理等
    }
  }
);
```

#### 模型选择指南

| 模型 | 特点 | 适用场景 | maxTokens 推荐 |
|------|------|---------|---------------|
| `nalang-xl` | 速度快，成本低 | 简单列表、短文本生成 | 1500-2000 |
| `nalang-xl-10` | 质量高，理解力强 | 复杂对话、长文本、状态管理 | 2000-3000 |
| `nalang-max-0826` | 最新强化推理与对局策略 | 博弈决策、复杂规则解析、需要更稳输出的游戏 AI | 3000-3500 |

#### 请求规范（避免 HTTP 400）

- **仅支持 `user` / `assistant` 角色**：`role: 'system'` 会被后端直接拒绝，常见表现是 400。将系统提示词改写为首条 `user` 消息或在前端私有变量中维护。
- **消息必须是纯文本字符串**：不要向 `messages` 塞入 `undefined/null`、对象或超长 JSON；必要时使用 `String(value)` 兜底并限制长度。
- **确保在 `dzmm:ready` 之后调用**：在就绪事件触发前提交请求会触发 400/401。
- **限制 `messages` 尺寸**：建议保留最近若干轮（<=20 条），超出部分先 `slice`，避免 payload 超限。
- **捕获异常并记录上下文**：包装在 `try/catch` 中，将 prompt、模型名、消息条数写入日志，方便复现。

```javascript
function formatMessages(history, systemHint) {
  const payload = [];
  if (systemHint) {
    payload.push({
      role: 'user',
      content: `【系统提示】${systemHint}`.slice(0, 1200)
    });
  }
  history.slice(-20).forEach((item) => {
    if (!item?.role || !item?.content) return;
    payload.push({
      role: item.role === 'assistant' ? 'assistant' : 'user',
      content: String(item.content).slice(0, 2000)
    });
  });
  return payload;
}
```

```javascript
try {
  await window.dzmm.completions(
    {
      model: 'nalang-max-0826',
      messages: formatMessages(chatHistory, '你是友好的像素地城向导'),
      maxTokens: 900,
    },
    (newContent, done) => {
      buffer = newContent;
      if (done) {
        console.log('[dzmm] completions done', { length: buffer.length });
      }
    }
  );
} catch (error) {
  console.error('[dzmm] completions failed', {
    error,
    model: 'nalang-max-0826',
    count: chatHistory.length
  });
}
```

#### 单用户请求头

- 平台默认会注入 `x-dzmm-user`，用于区分访客会话；大多数组件无需干预。
- **自定义代理或手动转发**时必须显式透传该头，否则后端会返回 400。推荐使用固定常量或设备指纹：

```javascript
const SINGLE_USER_HEADERS = { 'x-dzmm-user': 'local-dev' };

await window.dzmm.completions(
  {
    model: 'nalang-max-0826',
    messages,
    maxTokens: 800,
    headers: SINGLE_USER_HEADERS, // 仅在自建代理时需要
  },
  streamHandler,
);
```

---

### 3. 云端存储：`window.dzmm.kv`

**用途**：持久化存储游戏进度、用户数据（可选功能）

#### 3.1 保存数据：`kv.put()`

```typescript
window.dzmm.kv.put(
  key: string,
  value: any
): Promise<void>
```

**示例**：
```javascript
// 保存简单数据
await window.dzmm.kv.put('username', '张三');

// 保存对象（自动序列化）
await window.dzmm.kv.put('game_state', {
  level: 5,
  score: 1200,
  items: ['sword', 'shield']
});
```

#### 3.2 读取数据：`kv.get()`

```typescript
window.dzmm.kv.get(
  key: string
): Promise<{ value?: any }>
```

**示例**：
```javascript
// 读取数据
const result = await window.dzmm.kv.get('game_state');

if (result.value) {
  const state = result.value;  // 直接使用，无需 JSON.parse
  console.log('当前等级:', state.level);
} else {
  console.log('没有保存的数据');
}
```

#### 注意事项

⚠️ **返回值格式**：
```javascript
// 可能是 { value: data }
const result = await kv.get('key');
const data = result.value;

// 有时也可能直接返回 data（兼容性处理）
const data = result?.value || result;
```

⚠️ **数据类型**：
- ✅ 支持：Object, Array, String, Number, Boolean
- ❌ 不支持：Function, Symbol, undefined
- 💡 对象会自动序列化，无需手动 `JSON.stringify`

⚠️ **命名规范**：
```javascript
// ✅ 推荐：加版本号，避免旧数据污染
`game_state_v2_${userId}`
`post_detail_v1_${postId}`

// ❌ 不推荐：无版本管理
`game_state`
`post_detail`
```

---

## 🚀 标准开发流程

### 步骤 1：创建 HTML 基础结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>我的应用</title>
</head>
<body>
  <div id="app" x-data="myApp">
    <!-- 你的内容 -->
  </div>

  <script>
    // 你的 JavaScript 代码
  </script>

  <style>
    /* 你的 CSS 样式 */
  </style>

  <!-- 引入 Alpine.js -->
  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

### 步骤 2：初始化 Alpine.js 应用

```javascript
document.addEventListener('alpine:init', () => {
  Alpine.data('myApp', () => ({
    // 状态变量
    content: '',
    isLoading: false,

    // 初始化方法
    async init() {
      // 等待 DZMM API 就绪
      await new Promise((resolve) => {
        const handler = (event) => {
          if (event.data?.type === 'dzmm:ready') {
            window.removeEventListener('message', handler);
            resolve();
          }
        };
        window.addEventListener('message', handler);
      });

      // 读取存档（可选）
      await this.loadState();
    },

    // 你的业务逻辑
    async generate() {
      this.isLoading = true;

      await window.dzmm.completions(
        {
          model: 'nalang-xl',
          messages: [{ role: 'user', content: '生成内容' }],
          maxTokens: 1500
        },
        (newContent, done) => {
          this.content = newContent;
          if (done) {
            this.isLoading = false;
          }
        }
      );
    },

    async loadState() {
      const result = await window.dzmm.kv.get('my_state');
      if (result.value) {
        Object.assign(this, result.value);
      }
    },

    async saveState() {
      await window.dzmm.kv.put('my_state', {
        content: this.content,
        // 其他需要保存的状态
      });
    }
  }));
});
```

### 步骤 3：设计 UI 界面

```html
<div id="app" x-data="myApp">
  <!-- 加载状态 -->
  <div x-show="isLoading">
    <p>加载中...</p>
  </div>

  <!-- 输入区 -->
  <div x-show="!isLoading">
    <textarea x-model="userInput" placeholder="输入内容..."></textarea>
    <button @click="generate">生成</button>
  </div>

  <!-- 输出区 -->
  <div x-show="content">
    <p x-text="content"></p>
  </div>
</div>
```

---

## 🎨 三种工作流架构

根据实际案例总结，提供三种典型架构模式。

---

### 架构 1：无状态生成器（最简单）

**适用场景**：
- ✅ 一次性内容生成（文案、翻译、总结）
- ✅ 无需历史记录
- ✅ 无需用户账号

**特点**：
- 无数据持久化
- 无状态管理
- 代码量最少

#### 完整示例：AI 翻译器

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>AI 翻译器</title>
</head>
<body>
  <div id="app" x-data="translator">
    <h1>🌐 AI 翻译器</h1>

    <textarea x-model="input" placeholder="输入要翻译的文本..."></textarea>

    <button
      @click="translate"
      :disabled="isLoading || !input.trim()">
      <span x-show="!isLoading">翻译</span>
      <span x-show="isLoading">翻译中...</span>
    </button>

    <div x-show="output" class="result">
      <h3>翻译结果：</h3>
      <p x-text="output"></p>
    </div>
  </div>

  <script>
    document.addEventListener('alpine:init', () => {
      Alpine.data('translator', () => ({
        input: '',
        output: '',
        isLoading: false,

        async init() {
          // 等待 API 就绪
          await new Promise((resolve) => {
            const handler = (event) => {
              if (event.data?.type === 'dzmm:ready') {
                window.removeEventListener('message', handler);
                resolve();
              }
            };
            window.addEventListener('message', handler);
          });
        },

        async translate() {
          if (this.isLoading || !this.input.trim()) return;

          this.isLoading = true;
          this.output = '';

          await window.dzmm.completions(
            {
              model: 'nalang-xl',
              messages: [
                { role: 'user', content: `将以下文本翻译成英文：\n\n${this.input}` }
              ],
              maxTokens: 1500
            },
            (newContent, done) => {
              this.output = newContent;
              if (done) {
                this.isLoading = false;
              }
            }
          );
        }
      }));
    });
  </script>

  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 50px auto;
      padding: 20px;
    }

    textarea {
      width: 100%;
      padding: 10px;
      border: 2px solid #ddd;
      border-radius: 8px;
      font-size: 14px;
      margin-bottom: 10px;
    }

    button {
      width: 100%;
      padding: 12px;
      background: #4CAF50;
      color: white;
      border: none;
      border-radius: 8px;
      font-size: 16px;
      cursor: pointer;
    }

    button:disabled {
      background: #ccc;
      cursor: not-allowed;
    }

    .result {
      margin-top: 20px;
      padding: 15px;
      background: #f5f5f5;
      border-radius: 8px;
    }
  </style>

  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

**优点**：
- ✅ 代码简洁，易维护
- ✅ 无状态，无内存泄漏风险
- ✅ 快速开发（30分钟可完成）

**缺点**：
- ❌ 无历史记录
- ❌ 刷新页面后内容丢失

---

### 架构 2：有状态对话系统（中等复杂度）

**适用场景**：
- ✅ 多轮对话（聊天机器人、游戏）
- ✅ 需要记住上下文
- ✅ 需要状态管理（好感度、积分等）

**特点**：
- 使用 Alpine.store 管理全局状态
- 使用 KV 存储持久化数据
- 支持多轮对话历史

#### 完整示例：简易聊天机器人

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>AI 助手</title>
</head>
<body>
  <div id="app" x-data="chatApp">
    <div class="chat-container">
      <h1>🤖 AI 助手</h1>

      <!-- 聊天记录 -->
      <div class="messages">
        <template x-for="msg in $store.chat.displayMessages" :key="msg.id">
          <div :class="'message ' + msg.role">
            <span class="role" x-text="msg.role === 'user' ? '你' : 'AI'"></span>
            <p x-text="msg.content"></p>
          </div>
        </template>

        <div x-show="isLoading" class="message assistant">
          <span class="role">AI</span>
          <p>思考中...</p>
        </div>
      </div>

      <!-- 输入框 -->
      <div class="input-area">
        <input
          x-model="userInput"
          @keydown.enter="sendMessage"
          placeholder="输入消息..."
          :disabled="isLoading" />
        <button @click="sendMessage" :disabled="isLoading || !userInput.trim()">
          发送
        </button>
      </div>
    </div>
  </div>

  <script>
    document.addEventListener('alpine:init', () => {
      // 全局状态存储
      Alpine.store('chat', {
        messages: [],           // 完整对话历史（仅 user/assistant）
        displayMessages: [],    // 显示的消息（用户可见）
        systemHint: '你是一个友好的 AI 助手，擅长回答问题和提供建议。', // 不直接发送，按需拼接

        async init() {
          // 等待 API 就绪
          await new Promise((resolve) => {
            const handler = (event) => {
              if (event.data?.type === 'dzmm:ready') {
                window.removeEventListener('message', handler);
                resolve();
              }
            };
            window.addEventListener('message', handler);
          });

          // 加载历史记录
          const result = await window.dzmm.kv.get('chat_history_v1');
          if (result.value) {
            this.messages = result.value.messages || [];
            this.displayMessages = result.value.displayMessages || [];
            this.systemHint = result.value.systemHint || this.systemHint;
          }
        },

        addUserMessage(content) {
          const msg = { role: 'user', content };
          this.messages.push(msg);
          this.displayMessages.push({ ...msg, id: Date.now() });
        },

        addAssistantMessage(content) {
          const msg = { role: 'assistant', content };
          this.messages.push(msg);
          this.displayMessages.push({ ...msg, id: Date.now() + 1 });
        },

        async save() {
          await window.dzmm.kv.put('chat_history_v1', {
            messages: this.messages.slice(-20),  // 只保留最近 20 条
            displayMessages: this.displayMessages.slice(-20),
            systemHint: this.systemHint,
          });
        }
      });

      // 主组件
      Alpine.data('chatApp', () => ({
        userInput: '',
        isLoading: false,

        async sendMessage() {
          if (this.isLoading || !this.userInput.trim()) return;

          const input = this.userInput.trim();
          this.userInput = '';

          // 添加用户消息
          this.$store.chat.addUserMessage(input);

          this.isLoading = true;
          let aiResponse = '';

          const requestMessages = [];
          if (this.$store.chat.systemHint) {
            requestMessages.push({
              role: 'user',
              content: `【提示】${this.$store.chat.systemHint}`.slice(0, 1200),
            });
          }
          this.$store.chat.messages.slice(-20).forEach((msg) => {
            if (!msg?.content) return;
            requestMessages.push({
              role: msg.role === 'assistant' ? 'assistant' : 'user',
              content: String(msg.content).slice(0, 2000),
            });
          });

          try {
            await window.dzmm.completions(
              {
                model: 'nalang-max-0826',
                messages: requestMessages,
                maxTokens: 1800
              },
              (newContent, done) => {
                aiResponse = newContent;
                if (done) {
                  this.$store.chat.addAssistantMessage(aiResponse);
                  this.$store.chat.save();  // 保存到云端
                  this.isLoading = false;
                }
              }
            );
          } catch (error) {
            console.error('[chat] completions error', {
              error,
              model: 'nalang-max-0826',
              payloadSize: requestMessages.length,
            });
            this.isLoading = false;
          }
        }
      }));
    });
  </script>

  <style>
    body {
      margin: 0;
      font-family: sans-serif;
      background: #f0f0f0;
    }

    .chat-container {
      max-width: 600px;
      margin: 20px auto;
      background: white;
      border-radius: 12px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      display: flex;
      flex-direction: column;
      height: 80vh;
    }

    h1 {
      text-align: center;
      padding: 20px;
      margin: 0;
      border-bottom: 1px solid #eee;
    }

    .messages {
      flex: 1;
      overflow-y: auto;
      padding: 20px;
    }

    .message {
      margin-bottom: 15px;
      padding: 10px;
      border-radius: 8px;
    }

    .message.user {
      background: #e3f2fd;
      margin-left: 50px;
    }

    .message.assistant {
      background: #f5f5f5;
      margin-right: 50px;
    }

    .role {
      font-weight: bold;
      display: block;
      margin-bottom: 5px;
      font-size: 12px;
      color: #666;
    }

    .input-area {
      display: flex;
      padding: 15px;
      border-top: 1px solid #eee;
      gap: 10px;
    }

    .input-area input {
      flex: 1;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 20px;
      font-size: 14px;
    }

    .input-area button {
      padding: 10px 20px;
      background: #2196F3;
      color: white;
      border: none;
      border-radius: 20px;
      cursor: pointer;
    }

    .input-area button:disabled {
      background: #ccc;
      cursor: not-allowed;
    }
  </style>

  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

**优点**：
- ✅ 支持多轮对话
- ✅ 自动保存历史记录
- ✅ 刷新页面后数据不丢失

**缺点**：
- ❌ 需要管理消息历史（可能超出 token 限制）
- ❌ 代码复杂度中等

**优化建议**：
```javascript
// 限制历史记录数量，避免超出 maxTokens
this.messages = this.messages.slice(-20);  // 只保留最近 20 条

// 或者计算 token 数量（粗略估计：1 token ≈ 1.5 个中文字符）
const totalChars = this.messages.reduce((sum, msg) => sum + msg.content.length, 0);
if (totalChars > 3000) {
  this.messages = this.messages.slice(-10);  // 超出限制时删除旧消息
}
```

---

### 架构 3：分层缓存内容平台（高复杂度）

**适用场景**：
- ✅ 内容社区（贴吧、论坛）
- ✅ 大量数据需要按需加载
- ✅ 需要优化性能和成本

**特点**：
- 两层缓存：列表缓存 + 详情缓存
- 无限滚动加载
- 并发控制锁

#### 架构设计

```
用户输入 → 检查列表缓存
              ↓ [未命中]
           生成列表 → 保存轻量级数据 (tieba_list_v1)
              ↓
         点击项目 → 检查详情缓存
              ↓ [未命中]
           生成详情 → 保存完整数据 (tieba_detail_v1_${id})
```

#### 完整示例：AI 故事生成器

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>AI 故事库</title>
</head>
<body>
  <div id="app" x-data="storyApp">
    <!-- 输入页 -->
    <div x-show="view === 'input'" class="input-view">
      <h1>📚 AI 故事库</h1>
      <input
        x-model="theme"
        @keydown.enter="loadStories"
        placeholder="输入故事主题（如：科幻、悬疑）" />
      <button @click="loadStories" :disabled="isLoading || !theme.trim()">
        <span x-show="!isLoading">生成故事</span>
        <span x-show="isLoading">生成中...</span>
      </button>
    </div>

    <!-- 列表页 -->
    <div x-show="view === 'list'" class="list-view">
      <button @click="backToInput">← 返回</button>
      <h2 x-text="theme + ' 故事集'"></h2>

      <div class="story-list">
        <template x-for="story in $store.stories.list" :key="story.id">
          <div class="story-card" @click="viewStory(story.id)">
            <h3 x-text="story.title"></h3>
            <p class="author" x-text="'作者：' + story.author"></p>
          </div>
        </template>

        <div x-show="isLoading" class="loading">加载中...</div>
      </div>
    </div>

    <!-- 详情页 -->
    <div x-show="view === 'detail'" class="detail-view">
      <button @click="backToList">← 返回列表</button>

      <div x-show="$store.stories.currentStory" class="story-detail">
        <h1 x-text="$store.stories.currentStory?.title"></h1>
        <p class="author" x-text="'作者：' + $store.stories.currentStory?.author"></p>
        <div class="content" x-text="$store.stories.currentStory?.content"></div>
      </div>

      <div x-show="isLoading" class="loading">加载故事内容...</div>
    </div>
  </div>

  <script>
    document.addEventListener('alpine:init', () => {
      // 全局故事存储
      Alpine.store('stories', {
        theme: '',
        list: [],
        currentStory: null,
        _generating: false,  // 并发控制锁

        async init(theme) {
          if (this._generating) return;

          this._generating = true;
          this.theme = theme;
          this.list = [];

          try {
            // 1. 检查列表缓存
            const cacheKey = `story_list_v1_${theme}`;
            const cached = await window.dzmm.kv.get(cacheKey);

            if (cached.value) {
              this.list = cached.value.list;
              return;
            }

            // 2. 生成新列表
            await this.generateList(theme);
          } finally {
            this._generating = false;
          }
        },

        async generateList(theme) {
          const prompt = `你是故事创作专家。请围绕"${theme}"主题生成5个故事标题。

使用XML格式输出：
<story>
<title>故事标题（15字以内）</title>
<author>作者名</author>
</story>

生成5个故事。`;

          let content = '';
          await window.dzmm.completions(
            {
              model: 'nalang-xl',
              messages: [{ role: 'user', content: prompt }],
              maxTokens: 1500
            },
            (newContent, done) => {
              content = newContent;
            }
          );

          // 解析 XML
          const matches = [...content.matchAll(/<story>[\s\S]*?<title>(.*?)<\/title>[\s\S]*?<author>(.*?)<\/author>[\s\S]*?<\/story>/g)];

          this.list = matches.map(match => ({
            id: `story_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
            title: match[1].trim(),
            author: match[2].trim()
          }));

          // 保存列表缓存（只保存元数据，不包含内容）
          await window.dzmm.kv.put(`story_list_v1_${theme}`, {
            theme,
            list: this.list
          });
        },

        async loadDetail(storyId) {
          if (this._generating) return null;

          this._generating = true;

          try {
            const story = this.list.find(s => s.id === storyId);
            if (!story) return null;

            // 1. 检查详情缓存
            const cacheKey = `story_detail_v1_${storyId}`;
            const cached = await window.dzmm.kv.get(cacheKey);

            if (cached.value) {
              return cached.value;
            }

            // 2. 生成详情
            const prompt = `写一个完整的故事：

标题：${story.title}
作者：${story.author}

要求：
- 500-800字
- 有开头、发展、高潮、结尾
- 情节生动有趣

直接输出故事内容，不要额外说明。`;

            let content = '';
            await window.dzmm.completions(
              {
                model: 'nalang-xl-10',
                messages: [{ role: 'user', content: prompt }],
                maxTokens: 2500
              },
              (newContent, done) => {
                content = newContent;
              }
            );

            const detail = {
              id: storyId,
              title: story.title,
              author: story.author,
              content: content.trim()
            };

            // 保存详情缓存（独立存储）
            await window.dzmm.kv.put(cacheKey, detail);

            return detail;
          } finally {
            this._generating = false;
          }
        }
      });

      // 主组件
      Alpine.data('storyApp', () => ({
        view: 'input',
        theme: '',
        isLoading: false,

        async init() {
          // 等待 API 就绪
          await new Promise((resolve) => {
            const handler = (event) => {
              if (event.data?.type === 'dzmm:ready') {
                window.removeEventListener('message', handler);
                resolve();
              }
            };
            window.addEventListener('message', handler);
          });
        },

        async loadStories() {
          if (!this.theme.trim() || this.isLoading) return;

          this.isLoading = true;
          await this.$store.stories.init(this.theme);
          this.isLoading = false;
          this.view = 'list';
        },

        async viewStory(storyId) {
          this.$store.stories.currentStory = null;
          this.view = 'detail';

          this.isLoading = true;
          const detail = await this.$store.stories.loadDetail(storyId);
          this.isLoading = false;

          if (detail) {
            this.$store.stories.currentStory = detail;
          } else {
            this.backToList();
          }
        },

        backToList() {
          this.view = 'list';
          this.$store.stories.currentStory = null;
        },

        backToInput() {
          this.view = 'input';
          this.theme = '';
          this.$store.stories.list = [];
        }
      }));
    });
  </script>

  <style>
    body {
      margin: 0;
      font-family: sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      color: #333;
    }

    .input-view {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      color: white;
      padding: 20px;
    }

    .input-view h1 {
      font-size: 48px;
      margin-bottom: 30px;
    }

    .input-view input {
      width: 400px;
      padding: 15px;
      font-size: 16px;
      border: none;
      border-radius: 8px 8px 0 0;
    }

    .input-view button {
      width: 400px;
      padding: 15px;
      font-size: 16px;
      background: #4CAF50;
      color: white;
      border: none;
      border-radius: 0 0 8px 8px;
      cursor: pointer;
    }

    .input-view button:disabled {
      background: #ccc;
      cursor: not-allowed;
    }

    .list-view, .detail-view {
      max-width: 800px;
      margin: 0 auto;
      padding: 20px;
    }

    .list-view button, .detail-view button {
      margin-bottom: 20px;
      padding: 10px 20px;
      background: white;
      border: none;
      border-radius: 8px;
      cursor: pointer;
    }

    h2 {
      color: white;
      text-align: center;
      margin-bottom: 30px;
    }

    .story-list {
      display: grid;
      gap: 15px;
    }

    .story-card {
      background: white;
      padding: 20px;
      border-radius: 12px;
      cursor: pointer;
      transition: transform 0.2s, box-shadow 0.2s;
    }

    .story-card:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(0,0,0,0.2);
    }

    .story-card h3 {
      margin: 0 0 10px 0;
      color: #333;
    }

    .author {
      color: #666;
      font-size: 14px;
      margin: 0;
    }

    .story-detail {
      background: white;
      padding: 30px;
      border-radius: 12px;
    }

    .story-detail h1 {
      margin-top: 0;
    }

    .story-detail .content {
      line-height: 2;
      font-size: 16px;
      white-space: pre-wrap;
    }

    .loading {
      text-align: center;
      padding: 40px;
      color: white;
      font-size: 18px;
    }
  </style>

  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

**优点**：
- ✅ 按需加载，节省带宽和 AI 成本
- ✅ 两层缓存，用户体验流畅
- ✅ 支持大规模内容

**缺点**：
- ❌ 代码复杂度高
- ❌ 需要设计缓存策略

**关键设计点**：

1. **分层缓存**：
```javascript
// 列表缓存（轻量）
await kv.put(`story_list_v1_${theme}`, {
  theme,
  list: [{ id, title, author }]  // 只有元数据
});

// 详情缓存（重量）
await kv.put(`story_detail_v1_${id}`, {
  id, title, author, content  // 包含完整内容
});
```

2. **并发控制**：
```javascript
_generating: false,  // 锁标志

async generateList() {
  if (this._generating) return;  // 防止重复调用
  this._generating = true;
  try {
    // ... 生成逻辑
  } finally {
    this._generating = false;  // 释放锁
  }
}
```

3. **版本管理**：
```javascript
// 加版本号，避免旧数据污染
`story_list_v1_${theme}`
`story_detail_v1_${id}`

// 升级时改为 v2
`story_list_v2_${theme}`
```

---

## 🎯 最佳实践

### 1. AI 提示词设计

#### ✅ 使用结构化输出格式

**XML 方式（推荐用于列表数据）**：
```javascript
const prompt = `生成3个任务：

使用XML格式输出：
<task>
<title>任务标题</title>
<priority>高|中|低</priority>
</task>

生成3个任务。`;

// 解析代码
const matches = [...content.matchAll(/<task>[\s\S]*?<title>(.*?)<\/title>[\s\S]*?<priority>(.*?)<\/priority>[\s\S]*?<\/task>/g)];
const tasks = matches.map(match => ({
  title: match[1].trim(),
  priority: match[2].trim()
}));
```

**JSON 方式（推荐用于复杂对象）**：
```javascript
const prompt = `生成一个配置对象。

【极其重要】必须严格按照这个格式输出：
###CONFIG
{"theme":"dark","language":"zh","notifications":true}
###END

只输出上述格式，不要有任何其他内容。`;

// 解析代码
const configMatch = content.match(/###CONFIG\s*(.*?)\s*###END/s);
if (configMatch) {
  const config = JSON.parse(configMatch[1]);
}
```

**Markdown 方式（推荐用于文章内容）**：
```javascript
const prompt = `写一篇文章：

要求：
- 使用 Markdown 格式
- 包含标题、段落、列表
- 300字左右

直接输出内容。`;

// 渲染代码（需引入 marked.js）
const html = window.marked.parse(content);
```

#### ❌ 避免的提示词问题

```javascript
// ❌ 错误：提示词模糊
const prompt = '生成一些数据';

// ✅ 正确：明确格式和数量
const prompt = `生成5个用户数据。

使用XML格式：
<user>
<name>姓名</name>
<age>年龄（数字）</age>
</user>

生成5个用户。`;
```

---

### 2. 错误处理

```javascript
async function safeGenerate() {
  try {
    await window.dzmm.completions(
      { model: 'nalang-xl', messages: [...], maxTokens: 1500 },
      (newContent, done) => {
        this.content = newContent;

        if (done) {
          // 验证输出格式
          if (!this.validateOutput(newContent)) {
            console.error('AI 输出格式错误');
            this.content = '';
            this.showError('生成失败，请重试');
          }
        }
      }
    );
  } catch (error) {
    console.error('API 调用失败:', error);
    this.showError('网络错误，请检查连接');
  }
}

function validateOutput(content) {
  // 验证是否包含必需的标记
  return content.includes('<task>') && content.includes('</task>');
}
```

### 3. 日志控制策略

- **统一前缀**：使用常量通道（例如 `[dzmm-chat]`）标记日志来源，方便在控制台过滤。
- **只记录关键阶段**：初始化、请求发送、流式完成、错误等核心节点使用 `console.log`/`console.warn`/`console.error`，避免在回调中每个 token 输出一次。
- **结构化 payload**：日志第二个参数传对象，如 `{ model, tokenCount }`，禁止拼接长字符串，便于复制粘贴。
- **按需开关**：为详细调试日志增加布尔开关或环境变量，在生产构建时默认关闭。
- **异常必带上下文**：`catch` 中至少打印 `error`、`model`、`messages.length` 和自定义请求 ID，出现 400 / 500 时可直接定位。

# 开发设计哲学

1. **先确保可用性，再谈体验。** 所有接口调用遵循“准备好才能发”的原则，必要时引入兜底方案，确保每一次交互都能给出反馈。
2. **将复杂流程拆成清晰模块。** 统一在单 HTML 中组织结构、逻辑、样式，同时把状态管理、接口调用、输出解析等职责分离，便于演进与测试。
3. **提示词是系统设计的一部分。** 明确模型角色、输入格式、输出格式，用例外场景验证提示是否具备鲁棒性；保持提示轻量、可维护。
4. **日志即可观察性。** 用结构化、可控的日志帮助快速复现问题；开发阶段多输出，正式环境保留核心节点。
5. **持续回收经验。** 观察每次迭代中模型、KV、UI 的表现，把踩过的坑总结到指南，形成团队共享的最佳实践。

---

### 2. 错误处理

---

### 3. 性能优化

#### 限制对话历史长度
```javascript
// ❌ 错误：无限增长的历史记录
this.messages.push({ role: 'user', content: userInput });

// ✅ 正确：只保留最近N条
this.messages.push({ role: 'user', content: userInput });
this.messages = this.messages.slice(-20);  // 保留最近20条
```

#### 防抖用户输入
```javascript
let debounceTimer;

function onInput() {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    this.generate();
  }, 500);  // 500ms 后才执行
}
```

#### 缓存策略
```javascript
// ✅ 为缓存添加过期时间
await kv.put('cache_key', {
  data: content,
  timestamp: Date.now(),
  version: 1
});

// 读取时检查过期
const cached = await kv.get('cache_key');
if (cached.value) {
  const age = Date.now() - cached.value.timestamp;
  if (age < 3600000) {  // 1小时内有效
    return cached.value.data;
  }
}
```

---

### 4. 用户体验优化

#### 流式输出打字机效果
```javascript
(newContent, done) => {
  // 实时更新内容
  this.content = newContent;

  // 自动滚动到底部
  this.$nextTick(() => {
    const container = document.querySelector('.content');
    container.scrollTop = container.scrollHeight;
  });

  if (done) {
    // 完成后的处理
  }
}
```

#### 加载状态指示
```html
<div x-show="isLoading" class="loading">
  <div class="spinner"></div>
  <p>正在生成内容...</p>
</div>
```

```css
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

---

### 5. 数据安全

#### 不要存储敏感信息
```javascript
// ❌ 错误：存储密码
await kv.put('user_data', {
  username: 'zhangsan',
  password: '123456'  // 不安全！
});

// ✅ 正确：只存储公开信息
await kv.put('user_data', {
  username: 'zhangsan',
  preferences: { theme: 'dark' }
});
```

#### 输入验证
```javascript
function sanitizeInput(input) {
  // 移除 HTML 标签
  const clean = input.replace(/<[^>]*>/g, '');

  // 限制长度
  return clean.substring(0, 1000);
}
```

---

### 6. 调试日志

- ✅ **结构化输出**：为初始化、用户操作等关键节点打印分隔符，快速分辨阶段。
- ✅ **记录关键信息**：输出关键状态快照、输入提示、模型原始回复及解析结果，便于排查。
- ✅ **保留原文再解析**：解析前先打印完整返回内容（必要时截断），方便定位格式问题或模型偏差。
- ✅ **预留开关**：通过布尔变量（如 `debugEnabled`）统一控制日志量，上线环境可关闭以降低噪音。

---

## ❓ 常见问题

### Q1: `completions` 回调函数的 `newContent` 是增量还是全量？

**A**: 是全量累积内容，不是增量。

```javascript
// 错误理解 ❌
(newContent, done) => {
  this.content += newContent;  // 会导致内容重复！
}

// 正确理解 ✅
(newContent, done) => {
  this.content = newContent;  // 直接赋值
}
```

---

### Q2: 如何知道 API 是否就绪？

**A**: 必须等待 `dzmm:ready` 事件。

```javascript
// 标准等待模式
await new Promise((resolve) => {
  const handler = (event) => {
    if (event.data?.type === 'dzmm:ready') {
      window.removeEventListener('message', handler);
      resolve();
    }
  };
  window.addEventListener('message', handler);
});

// 此时才能调用 window.dzmm.*
```

---

### Q3: `kv.get()` 返回什么格式？

**A**: 返回 `{ value?: any }` 对象。

```javascript
const result = await kv.get('key');

// 情况1：有数据
if (result.value) {
  const data = result.value;  // 直接使用，无需 JSON.parse
}

// 情况2：无数据
if (!result.value) {
  console.log('没有找到数据');
}

// 兼容性处理（推荐）
const data = result?.value || result || null;
```

---

### Q4: 如何限制对话历史长度？

**A**: 定期裁剪 `messages` 数组。

```javascript
// 方法1：保留最近N条
this.messages = this.messages.slice(-20);

// 方法2：按 token 数估算（粗略）
const totalChars = this.messages.reduce((sum, msg) => sum + msg.content.length, 0);
if (totalChars > 4000) {
  this.messages = this.messages.slice(-10);
}

// 方法3：只保留系统提示词 + 最近N条
const systemMsg = this.messages.find(m => m.role === 'system');
const recentMsgs = this.messages.filter(m => m.role !== 'system').slice(-10);
this.messages = systemMsg ? [systemMsg, ...recentMsgs] : recentMsgs;
```

---

### Q5: 如何选择模型？

**A**: 根据任务复杂度选择。

| 任务类型 | 推荐模型 | 理由 |
|---------|---------|------|
| 简单文本生成 | `nalang-xl` | 速度快，成本低 |
| 复杂对话、状态管理 | `nalang-xl-10` | 理解力强，输出稳定 |
| 短文本（<200字） | `nalang-xl` | 性价比高 |
| 长文本（>500字） | `nalang-xl-10` | 质量更好 |

---

### Q6: 如何处理 AI 输出格式错误？

**A**: 添加格式验证和重试机制。

```javascript
async function generateWithRetry(maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    let content = '';

    await window.dzmm.completions(
      { model: 'nalang-xl', messages: [...], maxTokens: 1500 },
      (newContent, done) => {
        content = newContent;
      }
    );

    // 验证格式
    if (this.validateFormat(content)) {
      return content;  // 成功
    }

    console.warn(`格式错误，重试 ${i + 1}/${maxRetries}`);
  }

  throw new Error('多次重试后仍失败');
}

function validateFormat(content) {
  // 检查必需的标记
  return content.includes('<task>') && content.includes('</task>');
}
```

---

### Q7: KV 存储有容量限制吗？

**A**: 文档未说明，建议遵循以下原则：

- ✅ 单个 value 尽量小于 100KB
- ✅ 避免存储大文件（图片、视频）
- ✅ 定期清理过期数据
- ✅ 使用版本化 key（便于废弃旧数据）

```javascript
// 示例：清理过期缓存
async function cleanupOldCache() {
  const keys = ['cache_v1_a', 'cache_v1_b', 'cache_v1_c'];

  for (const key of keys) {
    const result = await kv.get(key);
    if (result.value) {
      const age = Date.now() - result.value.timestamp;
      if (age > 7 * 24 * 3600 * 1000) {  // 7天
        // 无法删除，但可以用新版本 key 替代
        console.log(`缓存 ${key} 已过期`);
      }
    }
  }
}
```

---

### Q8: 如何调试 AI 输出？

**A**: 使用控制台日志查看原始输出。

```javascript
await window.dzmm.completions(
  { model: 'nalang-xl', messages: [...], maxTokens: 1500 },
  (newContent, done) => {
    if (done) {
      console.log('===== AI 原始输出 =====');
      console.log(newContent);
      console.log('===== 输出结束 =====');
    }
  }
);
```

---

### Q9: Alpine.js 如何在回调中访问 `this`？

**A**: 使用箭头函数保持上下文。

```javascript
// ❌ 错误：普通函数会丢失 this
await window.dzmm.completions({...}, function(newContent, done) {
  this.content = newContent;  // this 指向错误！
});

// ✅ 正确：箭头函数保持 this
await window.dzmm.completions({...}, (newContent, done) => {
  this.content = newContent;  // this 指向 Alpine 组件
});
```

---

### Q10: 如何实现"重新生成"功能？

**A**: 保留原始输入，重新调用生成函数。

```javascript
Alpine.data('app', () => ({
  userInput: '',
  output: '',
  lastInput: '',  // 保存最后的输入

  async generate() {
    this.lastInput = this.userInput;  // 保存
    // ... 生成逻辑
  },

  async regenerate() {
    if (!this.lastInput) return;
    this.userInput = this.lastInput;
    await this.generate();
  }
}));
```

```html
<button @click="regenerate" x-show="output">
  🔄 重新生成
</button>
```

---

### Q11: 如何从 React/Vue 多组件框架迁移到 DZMM 单文件？

**A**: 使用现代构建工具保持开发体验，最后打包成单文件。

#### 技术栈选择

**推荐方案**：React + TypeScript + Vite + vite-plugin-singlefile

```bash
# 1. 创建项目
npm create vite@latest my-dzmm-app -- --template react-ts

# 2. 安装单文件打包插件
npm install -D vite-plugin-singlefile

# 3. 配置 vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
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

#### 架构设计

```
src/
├── pages/           # 页面组件（对应 Alpine.js 的 x-show 切换）
├── components/      # 可复用组件
├── services/        # DZMM API 封装
│   └── dzmm.ts      # initDzmm, completions, kvPut, kvGet
├── lib/             # 工具函数
│   ├── prompts.ts   # 提示词构建
│   └── storage.ts   # localStorage 安全封装
├── contexts/        # React Context（状态管理）
└── types/           # TypeScript 类型定义
```

#### 关键实践

**1. DZMM API 封装**（`src/services/dzmm.ts`）

```typescript
// 双重检测机制
export async function initDzmm(): Promise<boolean> {
  // 方式1：直接检测
  if (window.dzmm && typeof window.dzmm.completions === 'function') {
    return true;
  }

  // 方式2：事件监听 + 超时重检
  const ready = await Promise.race([
    new Promise<boolean>((resolve) => {
      const handler = (event: MessageEvent) => {
        if (event.data?.type === 'dzmm:ready') {
          window.removeEventListener('message', handler);
          resolve(true);
        }
      };
      window.addEventListener('message', handler);
    }),
    new Promise<boolean>((resolve) => {
      setTimeout(() => {
        resolve(window.dzmm && typeof window.dzmm.completions === 'function');
      }, 10000);
    }),
  ]);

  return ready;
}

// TypeScript 类型安全
export async function completions(
  options: CompletionsOptions,
  callback: StreamCallback
): Promise<void> {
  if (!window.dzmm) throw new Error('DZMM API 不可用');

  await window.dzmm.completions(options, callback);
}
```

**2. Sandbox 环境兼容**（`src/lib/storage.ts`）

DZMM 发布环境使用 iframe sandbox，需处理 localStorage 限制：

```typescript
const memoryStorage: Record<string, string> = {};
let localStorageAvailable: boolean | null = null;

function isLocalStorageAvailable(): boolean {
  if (localStorageAvailable !== null) return localStorageAvailable;
  try {
    const testKey = '__storage_test__';
    window.localStorage.setItem(testKey, testKey);
    window.localStorage.removeItem(testKey);
    localStorageAvailable = true;
    return true;
  } catch {
    localStorageAvailable = false;
    return false;
  }
}

export function setItem(key: string, value: string): void {
  if (isLocalStorageAvailable()) {
    localStorage.setItem(key, value);
  } else {
    memoryStorage[key] = value;
  }
}

export function getItem(key: string): string | null {
  if (isLocalStorageAvailable()) {
    return localStorage.getItem(key);
  }
  return memoryStorage[key] || null;
}
```

**3. Form 提交兼容**

Sandbox 禁止 `<form>` 提交，需改用按钮点击：

```tsx
// ❌ 错误：会在发布环境报错
<form onSubmit={handleSubmit}>
  <button type="submit">提交</button>
</form>

// ✅ 正确：使用 div + button
<div>
  <button type="button" onClick={handleSubmit}>提交</button>
</div>
```

**4. API 参数限制**

```typescript
// ⚠️ 关键错误：maxTokens 超限导致 HTTP 400
export const DZMM_MODELS = [
  { id: 'nalang-max-0826', maxTokens: 4000 },  // ❌ 超出范围
];

// ✅ 正确：遵守 200-3000 范围
export const DZMM_MODELS = [
  { id: 'nalang-max-0826', maxTokens: 3000 },
  { id: 'nalang-xl-0826', maxTokens: 3000 },
  { id: 'nalang-turbo-0826', maxTokens: 2000 },
];
```

**5. 消息数组构建**

```typescript
// 确保没有连续相同角色的消息
export function buildMessages(
  character: Character,
  messages: DzmmMessage[]
): DzmmMessage[] {
  const systemPrompt = buildSystemPrompt(character);
  const cleanedMessages = messages.map(m => ({
    role: m.role,
    content: cleanMessageContent(m.content),
  }));

  // 将 emphasis 合并到最后一条 user 消息
  const emphasis = getEmphasis();
  for (let i = cleanedMessages.length - 1; i >= 0; i--) {
    if (cleanedMessages[i].role === 'user') {
      cleanedMessages[i].content =
        `<last_input>\n${cleanedMessages[i].content}\n</last_input>\n\n${emphasis}`;
      break;
    }
  }

  return [
    { role: 'user', content: systemPrompt },
    ...cleanedMessages,
  ];
}
```

#### 构建和测试

```json
// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "build:single": "vite build --mode singlefile",
    "preview": "vite preview"
  }
}
```

**开发流程**：
1. 本地开发：`npm run dev` - 热重载，快速迭代
2. 构建单文件：`npm run build:single` - 生成 dist-single/index.html
3. DZMM 测试：上传到 DZMM 平台测试 sandbox 兼容性
4. 发布：通过 DZMM 工坊发布

#### 常见陷阱

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| localStorage 访问失败 | "sandboxed" 错误 | 使用 memoryStorage 降级 |
| Form 提交被阻止 | "allow-forms" 错误 | 改用 button + onClick |
| API 返回 400 | maxTokens 过大 | 检查是否在 200-3000 范围内 |
| 连续 user 消息 | API 拒绝 | 合并 emphasis 到最后一条 user |
| reroll 第一条消息 | 空上下文错误 | 禁止 reroll index 0 |

#### 调试技巧

```typescript
// 详细日志帮助排查 API 错误
console.log('[DZMM] 发送请求:', {
  model: options.model,
  maxTokens: options.maxTokens,
  messagesCount: options.messages.length,
});

options.messages.forEach((m, i) => {
  console.log(`  [${i}] role: ${m.role}, length: ${m.content.length}`);
  console.log(`      preview: ${m.content.substring(0, 150)}...`);
});

// 检测连续相同角色
for (let i = 1; i < options.messages.length; i++) {
  if (options.messages[i].role === options.messages[i - 1].role) {
    console.warn(`⚠️ 连续的 ${options.messages[i].role} 消息！索引 ${i-1} 和 ${i}`);
  }
}
```

#### 优势总结

**React/Vue 多组件 vs Alpine.js 单文件**：
- ✅ TypeScript 类型安全
- ✅ 组件化开发，易维护
- ✅ 丰富的生态（UI 库、路由、状态管理）
- ✅ 热重载开发体验
- ✅ 最终仍然打包成单 HTML 文件

**实际案例**：yoshiwara-chronicles 项目（54 commits）
- 900+ KB 单文件（gzip: 473 KB）
- 完整视觉小说系统（多开场、消息管理、富文本渲染、存档系统）
- 从 React 多组件架构成功迁移到 DZMM 平台

---

## 📚 附录：完整模板

### 最小可运行模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DZMM 应用模板</title>
</head>
<body>
  <div id="app" x-data="app">
    <h1>我的应用</h1>

    <input x-model="input" placeholder="输入内容..." />
    <button @click="generate" :disabled="isLoading">生成</button>

    <div x-show="output">
      <p x-text="output"></p>
    </div>
  </div>

  <script>
    document.addEventListener('alpine:init', () => {
      Alpine.data('app', () => ({
        input: '',
        output: '',
        isLoading: false,

        async init() {
          // 等待 DZMM API 就绪
          await new Promise((resolve) => {
            const handler = (event) => {
              if (event.data?.type === 'dzmm:ready') {
                window.removeEventListener('message', handler);
                resolve();
              }
            };
            window.addEventListener('message', handler);
          });
        },

        async generate() {
          if (!this.input.trim() || this.isLoading) return;

          this.isLoading = true;
          this.output = '';

          try {
            await window.dzmm.completions(
              {
                model: 'nalang-xl',
                messages: [{ role: 'user', content: this.input }],
                maxTokens: 1500
              },
              (newContent, done) => {
                this.output = newContent;
                if (done) {
                  this.isLoading = false;
                }
              }
            );
          } catch (error) {
            console.error('生成失败:', error);
            this.isLoading = false;
          }
        }
      }));
    });
  </script>

  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 50px auto;
      padding: 20px;
    }

    input {
      width: 100%;
      padding: 10px;
      font-size: 16px;
      margin-bottom: 10px;
    }

    button {
      padding: 10px 20px;
      font-size: 16px;
      cursor: pointer;
    }

    button:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  </style>

  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

---

## 🎓 学习路径建议

1. **第一步**：复制"最小可运行模板"，理解基本流程
2. **第二步**：实现"无状态生成器"（如翻译器）
3. **第三步**：添加 KV 存储，实现"有状态对话系统"
4. **第四步**：学习"分层缓存架构"，优化性能

---

## 📖 参考资源

- **Alpine.js 官方文档**：https://alpinejs.dev
- **Marked.js 文档**：https://marked.js.org
- **正则表达式测试工具**：https://regex101.com

---

## 📝 更新日志

- **v1.0** (2025-01)：初始版本，基于逆向工程分析

---

**⚠️ 免责声明**：本文档基于对公开案例的逆向分析，非官方文档。接口可能随时变化，请以实际测试为准。

---

**Happy Coding! 🚀**
