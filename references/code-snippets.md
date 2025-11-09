# DZMM 代码片段库

本文档提供常用的 DZMM 开发代码片段，按功能分类。

## 1. 初始化与 API 就绪

### 标准初始化模式

```javascript
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

  console.log('DZMM API 已就绪');
  // 此处可以开始调用其他接口
}
```

### Alpine.js 组件初始化

```javascript
document.addEventListener('alpine:init', () => {
  Alpine.data('myApp', () => ({
    // 状态变量
    content: '',
    isLoading: false,

    async init() {
      await new Promise((resolve) => {
        const handler = (event) => {
          if (event.data?.type === 'dzmm:ready') {
            window.removeEventListener('message', handler);
            resolve();
          }
        };
        window.addEventListener('message', handler);
      });

      // 加载存档
      await this.loadState();
    }
  }));
});
```

## 2. AI Completions 调用

### 基础 AI 调用

```javascript
async generate() {
  this.isLoading = true;
  let response = '';

  try {
    await window.dzmm.completions(
      {
        model: 'nalang-xl',
        messages: [
          { role: 'user', content: '你的提示词' }
        ],
        maxTokens: 1500
      },
      (newContent, done) => {
        response = newContent;  // 注意：是全量内容，不是增量

        if (done) {
          this.content = response;
          this.isLoading = false;
        }
      }
    );
  } catch (error) {
    console.error('[AI 调用失败]', error);
    this.isLoading = false;
  }
}
```

### 多轮对话模式

```javascript
async sendMessage() {
  if (!this.userInput.trim() || this.loading) return;

  const input = this.userInput.trim();
  this.userInput = '';

  // 添加用户消息
  this.conversationHistory.push({
    role: 'user',
    content: input
  });

  this.loading = true;
  let aiResponse = '';

  // 构建消息（保留最近 20 条）
  const messages = this.conversationHistory.slice(-20).map(msg => ({
    role: msg.role,
    content: String(msg.content).slice(0, 2000)
  }));

  try {
    await window.dzmm.completions(
      {
        model: 'nalang-max-0826',
        messages: messages,
        maxTokens: 1800
      },
      (newContent, done) => {
        aiResponse = newContent;
        if (done) {
          this.conversationHistory.push({
            role: 'assistant',
            content: aiResponse
          });
          this.loading = false;
        }
      }
    );
  } catch (error) {
    console.error('[对话失败]', error);
    this.loading = false;
  }
}
```

### 带系统提示词的调用

```javascript
async callAI(userInput) {
  const systemPrompt = `你是一个友好的 AI 助手。
请用简洁的语言回答用户问题。`;

  const messages = [
    { role: 'user', content: systemPrompt },
    ...this.conversationHistory.slice(-10),
    { role: 'user', content: userInput }
  ];

  await window.dzmm.completions(
    {
      model: 'nalang-xl-10',
      messages: messages,
      maxTokens: 2000
    },
    (newContent, done) => {
      // 处理回应
    }
  );
}
```

## 3. KV 存储操作

### 保存数据

```javascript
async saveState() {
  await window.dzmm.kv.put('my_app_state_v1', {
    content: this.content,
    timestamp: Date.now(),
    version: 1
  });
}
```

### 读取数据

```javascript
async loadState() {
  const result = await window.dzmm.kv.get('my_app_state_v1');

  if (result.value) {
    const state = result.value;
    this.content = state.content || '';

    // 检查版本
    if (state.version === 1) {
      console.log('加载状态成功');
    }
  } else {
    console.log('没有保存的状态');
  }
}
```

### 带过期检查的缓存

```javascript
async loadWithExpiry(key, maxAge = 3600000) {
  const result = await window.dzmm.kv.get(key);

  if (result.value) {
    const age = Date.now() - result.value.timestamp;
    if (age < maxAge) {
      return result.value.data;
    }
  }
  return null;
}

async saveWithTimestamp(key, data) {
  await window.dzmm.kv.put(key, {
    data: data,
    timestamp: Date.now()
  });
}
```

## 4. 指令解析系统

### 解析 JSON 指令

```javascript
function parseEffectCommand(content) {
  const effectMatch = content.match(/###EFFECT\s*({[\s\S]*?})\s*###END/);

  if (effectMatch) {
    try {
      const effect = JSON.parse(effectMatch[1]);
      return effect;
    } catch (e) {
      console.error('[解析指令失败]', e);
      return null;
    }
  }
  return null;
}

// 使用
const effect = parseEffectCommand(aiResponse);
if (effect) {
  executeEffect(effect);
  // 从显示内容中移除指令
  aiResponse = aiResponse.replace(/###EFFECT[\s\S]*?###END/, '').trim();
}
```

### 解析 XML 格式

```javascript
function parseXMLList(content, tagName) {
  const regex = new RegExp(
    `<${tagName}>[\\s\\S]*?<title>(.*?)<\\/title>[\\s\\S]*?</${tagName}>`,
    'g'
  );

  const matches = [...content.matchAll(regex)];
  return matches.map(match => ({
    title: match[1].trim()
  }));
}

// 使用
const items = parseXMLList(aiResponse, 'item');
```

## 5. Alpine.js 状态管理

### 全局 Store

```javascript
Alpine.store('chat', {
  messages: [],
  systemHint: '你是一个友好的助手',

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

    // 加载历史
    await this.load();
  },

  addMessage(role, content) {
    this.messages.push({
      role,
      content,
      id: Date.now()
    });
  },

  async save() {
    await window.dzmm.kv.put('chat_state_v1', {
      messages: this.messages.slice(-20),
      systemHint: this.systemHint
    });
  },

  async load() {
    const result = await window.dzmm.kv.get('chat_state_v1');
    if (result.value) {
      this.messages = result.value.messages || [];
      this.systemHint = result.value.systemHint || this.systemHint;
    }
  }
});
```

## 6. 视觉效果系统

### CSS 动画切换

```javascript
function applyEffect(effectName, duration = 1000) {
  document.body.classList.add(effectName);
  setTimeout(() => {
    document.body.classList.remove(effectName);
  }, duration);
}

// CSS
/*
.shake {
  animation: shake 0.5s;
}

@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-10px); }
  75% { transform: translateX(10px); }
}
*/
```

### 粒子系统基础

```javascript
class ParticleSystem {
  constructor(canvasId) {
    this.canvas = document.getElementById(canvasId);
    this.ctx = this.canvas.getContext('2d');
    this.particles = [];
    this.resize();
  }

  resize() {
    this.canvas.width = window.innerWidth;
    this.canvas.height = window.innerHeight;
  }

  addParticle(x, y, vx, vy, color = '#ffffff') {
    this.particles.push({ x, y, vx, vy, life: 1.0, color });
  }

  animate() {
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    this.particles = this.particles.filter(p => {
      p.x += p.vx;
      p.y += p.vy;
      p.vy += 0.3; // 重力
      p.life -= 0.02;

      if (p.life > 0) {
        this.ctx.globalAlpha = p.life;
        this.ctx.fillStyle = p.color;
        this.ctx.fillRect(p.x, p.y, 4, 4);
        return true;
      }
      return false;
    });

    if (this.particles.length > 0) {
      requestAnimationFrame(() => this.animate());
    }
  }
}
```

### Web Audio 音效

```javascript
class SoundSystem {
  constructor() {
    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
  }

  beep(frequency = 440, duration = 200, volume = 0.3) {
    const osc = this.ctx.createOscillator();
    const gain = this.ctx.createGain();

    osc.frequency.value = frequency;
    gain.gain.setValueAtTime(volume, this.ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(
      0.01,
      this.ctx.currentTime + duration / 1000
    );

    osc.connect(gain);
    gain.connect(this.ctx.destination);
    osc.start();
    osc.stop(this.ctx.currentTime + duration / 1000);
  }
}

const sounds = new SoundSystem();
sounds.beep(440, 200, 0.3);
```

## 7. 错误处理与日志

### 统一错误处理

```javascript
async function safeAICall(config, callback) {
  try {
    await window.dzmm.completions(config, callback);
  } catch (error) {
    console.error('[AI 调用失败]', {
      error,
      model: config.model,
      messageCount: config.messages.length
    });

    // 显示用户友好的错误信息
    callback('[系统错误：无法连接到 AI 服务，请稍后重试]', true);
  }
}
```

### 结构化日志

```javascript
function log(level, message, context = {}) {
  const timestamp = new Date().toISOString();
  const logEntry = {
    timestamp,
    level,
    message,
    ...context
  };

  if (level === 'error') {
    console.error(`[${timestamp}] ${message}`, context);
  } else {
    console.log(`[${timestamp}] ${message}`, context);
  }
}

// 使用
log('info', 'AI 调用开始', { model: 'nalang-xl', promptLength: 150 });
log('error', 'AI 调用失败', { error: e.message, model: 'nalang-xl' });
```

## 8. 实用工具函数

### 防抖函数

```javascript
function debounce(func, wait) {
  let timeout;
  return function(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => func.apply(this, args), wait);
  };
}

// 使用
const debouncedSearch = debounce(() => {
  this.search();
}, 500);
```

### 文本清理

```javascript
function sanitizeInput(input, maxLength = 1000) {
  // 移除 HTML 标签
  let clean = input.replace(/<[^>]*>/g, '');

  // 限制长度
  clean = clean.substring(0, maxLength);

  // 移除多余空格
  clean = clean.trim().replace(/\s+/g, ' ');

  return clean;
}
```

### 滚动到底部

```javascript
function scrollToBottom(elementRef) {
  this.$nextTick(() => {
    const container = this.$refs[elementRef];
    if (container) {
      container.scrollTop = container.scrollHeight;
    }
  });
}
```

## 9. 提示词模板

### 结构化输出提示词

```javascript
const structuredPrompt = `请生成3个任务。

使用 XML 格式输出：
<task>
<title>任务标题</title>
<priority>高|中|低</priority>
</task>

生成 3 个任务。`;
```

### 角色设定提示词

```javascript
const systemPrompt = `你是一个专业的 ${role}。

你的职责：
- ${responsibility1}
- ${responsibility2}

风格要求：
- ${style1}
- ${style2}

请用 ${length} 字左右回答。`;
```

## 10. 完整应用模板

### 最小可运行模板

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>DZMM 应用</title>
</head>
<body>
  <div id="app" x-data="app">
    <input x-model="input" placeholder="输入内容...">
    <button @click="generate" :disabled="isLoading">生成</button>
    <div x-show="output" x-text="output"></div>
  </div>

  <script>
    document.addEventListener('alpine:init', () => {
      Alpine.data('app', () => ({
        input: '',
        output: '',
        isLoading: false,

        async init() {
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

  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</body>
</html>
```

---

## 使用建议

1. **复制粘贴即用**：所有代码片段都可以直接使用或稍作修改
2. **组合使用**：根据需求组合多个片段
3. **参考完整示例**：查看 assets/examples/ 中的完整应用
4. **阅读详细文档**：参考 developer-guide.md 了解更多细节
