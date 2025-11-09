# DZMM Builder

> Comprehensive Claude Code skill for building AI-driven interactive web applications on the DZMM.AI platform.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://claude.com/claude-code)

## 🎯 Overview

**DZMM Builder** is a specialized Claude Code skill that provides comprehensive support for creating single-file HTML applications on the [DZMM.AI](https://dzmm.ai) platform. Whether you're building chatbots, visual novels, dating sims, content generators, or interactive games, this skill offers complete examples, reusable code patterns, and expert guidance.

## ✨ Features

### 🏗️ 5 Architecture Patterns

1. **Stateless Generator** - One-shot content generation (translators, summarizers)
2. **State Management Game** - Interactive games with dynamic variables (dating sims, RPGs)
3. **Stateful Dialogue System** - Multi-turn conversations with history
4. **Layered Cache Platform** - Content communities with two-tier caching
5. **Visual Novel / Galgame System** - Narrative-driven interactive fiction

### 🛠️ Core Capabilities

- **Complete API Integration** - Streaming AI, KV storage, chat API with branching narratives
- **Effect System** - "Fourth wall breaking" browser control (lights, sounds, particles, visual effects)
- **Rich Text Rendering** - Placeholder technique for nested structures (dialogue, options, formatting)
- **Message Management** - Reroll/regenerate, edit, delete with context preservation
- **Multi-Opening System** - Dynamic scene switching with state management
- **Resource Reuse Strategy** - URL-based asset loading (100% fidelity vs ≤60% CSS simulation)
- **Responsive Design** - Mobile-first with Tailwind breakpoints
- **Migration Guide** - Complete React/Vue to DZMM conversion workflow

### 📚 19 Best Practices

Including:
- DZMM API initialization (dual detection with timeout)
- Resource reuse strategy (critical for migrations)
- Conversation history management
- Error handling with retry logic
- Advanced prompt engineering (XML structure, emphasis sections)
- KV storage patterns (versioning, chunking, caching)
- Streaming AI response optimization
- Mobile responsive design patterns

## 📁 Structure

```
dzmm-builder/
├── SKILL.md                      # Main skill documentation (780+ lines)
├── README.md                     # This file
├── .gitignore                    # Git ignore rules
├── assets/
│   └── examples/                 # 7 complete working applications
│       ├── 小红书文案.html       # Stateless generator (15KB)
│       ├── 恋爱游戏.html         # State management game (35KB)
│       ├── horror-story.html     # Effect system (29KB)
│       ├── dungeon-adventure.html # Turn-based game (41KB)
│       ├── neon-gomoku.html      # Board game (24KB)
│       ├── 贴吧.html             # Content platform (26KB)
│       └── yoshiwara-chronicles-dzmm.html # Complete visual novel (84KB) ⭐
└── references/
    ├── developer-guide.md        # Complete DZMM platform documentation
    └── code-snippets.md          # Reusable code patterns library
```

## 🚀 Usage

### As a Claude Code Skill

1. **Install** this skill in your Claude Code environment:
   ```bash
   cd ~/.claude/skills/
   git clone https://github.com/waylon256yhw/dzmm-builder.git
   ```

2. **Invoke** the skill when working with DZMM projects:
   - "Build me a DZMM visual novel"
   - "How to add message reroll functionality?"
   - "Help me migrate my React app to DZMM"
   - "How to implement rich text rendering?"

3. **Reference** examples and code snippets directly from the skill files.

### As Documentation Reference

Browse the files directly for:
- **SKILL.md** - Complete skill guide with all capabilities
- **references/developer-guide.md** - DZMM platform documentation
- **references/code-snippets.md** - 17 categories of reusable code
- **assets/examples/** - 6 complete working applications

## 📖 Quick Start Examples

### Example 1: Simple Content Generator

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/alpinejs@3"></script>
</head>
<body x-data="app()">
  <textarea x-model="input"></textarea>
  <button @click="generate()">Generate</button>
  <div x-html="output"></div>

  <script>
    function app() {
      return {
        input: '',
        output: '',
        async generate() {
          await window.dzmm.completions({
            model: 'nalang-turbo-0826',
            messages: [{ role: 'user', content: this.input }]
          }, (content, done) => {
            if (done) this.output = content;
          });
        }
      }
    }
  </script>
</body>
</html>
```

### Example 2: Complete Visual Novel System

See `assets/examples/yoshiwara-chronicles-dzmm.html` (84KB) for the most comprehensive implementation including:
- **Multi-opening system** - Switch between "Night Chapter" and "Day Chapter" scenes
- **Rich text rendering** - Placeholder technique for nested options, dialogue quotes, italics
- **Message management** - Reroll/regenerate, edit, delete with full context preservation
- **Multi-slot save/load** - 3 save slots with character/message count preview
- **Modular prompts** - XML-structured system (main + character + guidance + emphasis)
- **Advanced techniques** - `<last_input>` emphasis, token optimization, streaming responses
- **Responsive design** - Mobile-optimized navigation, compressed spacing, flex-wrap buttons
- **Resource reuse** - GitHub Raw URLs for background images (100% visual fidelity)

This is a production-ready visual novel template based on 54 commits of iterative development.

Also see other examples in `assets/examples/` for specific patterns:
- 小红书文案.html - Simple content generator
- 恋爱游戏.html - State management game with progress bars
- horror-story.html - Effect system with lights/sounds/particles

## 🎓 Key Patterns

### Rich Text Rendering (Placeholder Technique)

```javascript
renderRichText(text) {
  // 1. Extract complex structures
  const optionsMap = [];
  let result = text.replace(/<options>([\s\S]*?)<\/options>/g, (match, content) => {
    const placeholder = '___OPTIONS_' + optionsMap.length + '___';
    optionsMap.push(content);
    return placeholder;
  });

  // 2. Process simple text
  result = result
    .replace(/\*([^*]+)\*/g, '<em>$1</em>')
    .replace(/「([^」]+)」/g, '<span class="dialogue">「$1」</span>')
    .replace(/\n/g, '<br>');

  // 3. Restore structures as HTML
  optionsMap.forEach((content, i) => {
    const buttons = /* generate buttons */;
    result = result.replace('___OPTIONS_' + i + '___', buttons);
  });

  return result;
}
```

### Message Reroll with Context Preservation

```javascript
async rerollMessage(messageIndex) {
  const contextMessages = this.messages.slice(0, messageIndex);

  // Clean history and emphasize last input
  const cleanedMessages = contextMessages.map(msg => ({
    ...msg,
    content: msg.role === 'assistant'
      ? msg.content.replace(/<options>[\s\S]*?<\/options>/g, '').trim()
      : msg.content
  }));

  await window.dzmm.completions({
    model: this.selectedModel,
    messages: [
      { role: 'user', content: systemPrompt },
      ...cleanedMessages,
      { role: 'user', content: emphasis }
    ]
  }, (content, done) => {
    if (done) this.messages[messageIndex].content = content;
  });
}
```

## 🌟 Real-World Application

This skill was developed based on the **yoshiwara-chronicles** project, a complete visual novel application featuring:
- 2 opening scenes (Night Chapter, Day Chapter)
- Rich text rendering with dialogue quotes and italics
- Message management (reroll, edit, delete)
- 3-slot save system with preview
- Modular prompt system (XML-structured)
- Mobile responsive design
- Resource reuse from original React project (100% visual fidelity)

See the [yoshiwara-chronicles repository](https://github.com/waylon256/yoshiwara-chronicles) for the complete implementation.

## 📊 Model Selection Guide

| Model | Context | Speed | Use Case |
|-------|---------|-------|----------|
| nalang-turbo-0826 | 32K | ⚡⚡⚡ | Simple tasks, quick responses |
| nalang-medium-0826 | 32K | ⚡⚡ | Moderate complexity |
| nalang-max-0826 | 32K | ⚡ | Game AI, complex rules, stable output |
| nalang-xl-0826 | 32K | ⚡ | Complex dialogues, long-form content |
| nalang-max-0826-16k | 16K | ⚡⚡ | Fast version with shorter context |
| nalang-xl-0826-16k | 16K | ⚡⚡ | Fast XL with shorter context |

## 🤝 Contributing

Contributions are welcome! Please feel free to:
- Report bugs or issues
- Suggest new features or patterns
- Submit pull requests with improvements
- Share your DZMM applications built with this skill

## 📄 License

MIT License - feel free to use this skill in your projects.

## 🔗 Resources

- **DZMM Platform**: https://dzmm.ai
- **Claude Code**: https://claude.com/claude-code
- **Reference Project**: [yoshiwara-chronicles](https://github.com/waylon256/yoshiwara-chronicles)
- **Development Guide**: See `DZMM_DEVELOPMENT_GUIDE.md` in yoshiwara-chronicles for comprehensive 54-commit development history

## 📮 Contact

For questions or discussions:
- Open an issue in this repository
- Reference the skill in your Claude Code session
- Check the comprehensive documentation in `SKILL.md`

---

**Built with ❤️ for the DZMM.AI community**

🤖 Generated with [Claude Code](https://claude.com/claude-code)
