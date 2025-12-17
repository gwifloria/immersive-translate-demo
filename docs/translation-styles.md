# 翻译风格（AI 专家）实现分析

## 概述

翻译风格（也称为 AI 专家/AI 助手）是沉浸式翻译的高级功能，允许用户选择不同领域的专业翻译风格，如技术文档、学术论文、新闻等。每个翻译风格都有专门优化的 prompt 模板和术语配置。

## 预定义翻译风格

### 可用的风格 ID

```javascript
aiAssistantIds: [
  "paraphrase",              // 意译模式
  "plain-english",           // 简明英语
  "paragraph-summarizer-expert", // 段落摘要专家
  "twitter",                 // Twitter
  "tech",                    // 技术文档
  "reddit",                  // Reddit
  "paper",                   // 学术论文
  "news",                    // 新闻
  "music",                   // 音乐
  "medical",                 // 医学
  "legal",                   // 法律
  "github",                  // GitHub
  "game",                    // 游戏
  "ecommerce",               // 电商
  "financial",               // 金融
  "fiction",                 // 小说
  "ao3",                     // AO3 同人小说
  "ebook",                   // 电子书
  "design",                  // 设计
  "web3",                    // Web3
  "bilingual-mix"            // 双语混排
]
```

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI 助手（翻译风格）系统                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────┐  ┌─────────────────────┐              │
│  │  预定义 AI 助手      │  │  用户自定义助手      │              │
│  │  (远程加载)          │  │  (customAiAssistants)│              │
│  └──────────┬──────────┘  └──────────┬──────────┘              │
│             │                        │                          │
│             └────────────┬───────────┘                          │
│                          ▼                                      │
│              ┌─────────────────────┐                            │
│              │   助手匹配引擎       │                            │
│              │   xh() 函数         │                            │
│              └──────────┬──────────┘                            │
│                         │                                       │
│                         ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    匹配条件                               │  │
│  │  - URL 匹配 (matches / excludeMatches)                   │  │
│  │  - 语言对匹配 (languageMatches)                          │  │
│  │  - 翻译服务匹配 (translationService)                     │  │
│  │  - 手动选择 (assistantId)                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │   配置合并引擎       │                            │
│              │   hx() / gx()       │                            │
│              └──────────┬──────────┘                            │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │  最终翻译服务配置    │                            │
│              │  (含 prompt/terms)  │                            │
│              └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

## AI 助手数据结构

### 助手配置对象

```typescript
interface AiAssistant {
  id: string;                    // 唯一标识符
  name?: string;                 // 显示名称
  extensionVersion: string;      // 最低支持版本
  version?: string;              // 助手版本
  priority?: number;             // 优先级（数值越小越优先）

  // 匹配规则
  matches?: string[];            // URL 匹配模式
  excludeMatches?: string[];     // URL 排除模式
  languageMatches?: string[];    // 语言对匹配，如 "en2zh-CN"

  // Prompt 配置
  systemPrompt?: string;         // 系统提示词
  prompt?: string;               // 用户提示词
  multiplePrompt?: string;       // 多段翻译提示词
  multipleSystemPrompt?: string; // 多段翻译系统提示词
  subtitlePrompt?: string;       // 字幕翻译提示词

  // 环境变量
  env?: {
    title_prompt?: string;
    summary_prompt?: string;
    terms_prompt?: string;
  };

  // 术语配置
  terms?: Record<string, string>; // 助手专属术语表

  // 语言覆盖
  langOverrides?: LangOverride[];

  // 性能配置
  temperature?: number;
  maxTextGroupLengthPerRequest?: number;
  maxTextLengthPerRequest?: number;
}
```

### 翻译服务中的助手配置

```json
{
  "translationServices": {
    "openai": {
      "assistantId": "common",        // 默认使用通用助手
      "fallbackAssistantId": "common" // 自动匹配失败时的回退
    },
    "claude": {
      "assistantId": "auto",          // 自动匹配
      "fallbackAssistantId": "paper"  // 回退到学术论文风格
    }
  }
}
```

## 核心实现

### 1. 助手匹配 - `xh()` 函数

```javascript
function xh(context, assistants, assistantId, currentAssistant) {
    // 如果当前助手已匹配当前翻译服务，直接返回
    if (currentAssistant?.applyTranslationService === context.translationService) {
        return currentAssistant;
    }

    let { url } = context;
    let matched = null;

    // 如果是 "common" 或空，不匹配任何助手
    if (assistantId === "common" || !assistantId) {
        return null;
    }

    try {
        // 1. 如果指定了具体 ID，直接查找
        if (assistantId) {
            matched = assistants.find(a => a.id === assistantId);
            if (matched) return matched;
        }

        // 2. 自动匹配：根据 URL 和语言对
        matched = assistants
            .filter(a => {
                // URL 匹配
                return Ve(url, a.matches) && !Ve(url, a.excludeMatches);
            })
            .filter(a => {
                // 语言对匹配
                if (!a.languageMatches) return true;
                return a.languageMatches.some(langMatch => {
                    let [source, target] = langMatch.split("2");
                    return ["auto", context.sourceLanguage].includes(source) &&
                           ["auto", context.targetLanguage].includes(target);
                });
            })?.[0];

        return matched;
    } catch (error) {
        console.error(error);
        return null;
    }
}
```

### 2. 配置合并 - `hx()` 函数

```javascript
async function hx(serviceConfig, defaultConfig, context, assistants, currentAssistant) {
    let config = { ...defaultConfig };

    // 1. 查找匹配的 AI 助手
    let assistant = xh(context, assistants, serviceConfig.assistantId, currentAssistant);

    // 2. 如果自动匹配失败，尝试回退助手
    if (!assistant && defaultConfig.fallbackAssistantId &&
        defaultConfig.fallbackAssistantId !== "common" &&
        defaultConfig.assistantId === "auto") {
        assistant = assistants?.find(a => a.id === defaultConfig.fallbackAssistantId);
    }

    // 3. 合并助手配置
    if (assistant) {
        px(assistant);  // 处理差异配置
        let env = { ...config.env || {}, ...assistant.env || {} };
        Object.assign(config, gx({ ...assistant, env }));
    }

    // 4. 收集页面上下文
    if (config.env && globalThis?.location &&
        !G7(globalThis.location.href, context.withAITitleBlockUrls)) {
        config.env.imt_domain = globalThis.location.hostname || "";
        config.env.imt_title = globalThis.document.originTitle ||
                               globalThis.document.title || "";
    }

    // 5. 应用语言覆盖配置
    config = mx(config, config.langOverrides, context);

    // 6. 处理术语表
    if (u0(defaultConfig, assistant)) {
        let terms = await H4();  // 获取上下文术语
        config.contextTerms = terms;
    }

    return config;
}
```

## 助手加载机制

### 远程加载

```javascript
// 从远程服务器加载助手配置
async function W_(assistantIds, localConfig) {
    let results = await Promise.allSettled(
        assistantIds.map(id =>
            De({ url: `${BASE_AI_URL}api/plugins/${id}.json` })
        )
    );

    results.forEach(result => {
        if (result.status === "fulfilled") {
            let assistant = result.value;
            if (assistant) {
                Wd("add", assistant, localConfig);  // 添加到本地配置
            }
        }
    });
}
```

## 用户自定义助手

### 配置结构

```json
{
  "customAiAssistants": [
    {
      "id": "custom-tech-cn",
      "name": "技术文档中文优化",
      "extensionVersion": "1.0.0",
      "matches": ["*.github.com/*", "*.stackoverflow.com/*"],
      "systemPrompt": "你是一位专业的技术文档翻译专家...",
      "terms": {
        "API": "API",
        "SDK": "SDK",
        "framework": "框架"
      }
    }
  ]
}
```

---

## 相关文档

- [上下文增强](./context-enhancement.md)
- [富文本翻译](./rich-text-translation.md)
- [AI 术语库](./ai-glossary.md)
