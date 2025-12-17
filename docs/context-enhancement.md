# 沉浸式翻译 - 上下文增强功能分析

## 概述

上下文增强 (Context Enhancement) 是沉浸式翻译的核心功能之一，通过向 AI 翻译服务提供页面元数据、主题摘要和术语表，显著提升翻译质量和一致性。

## 功能架构

```
┌─────────────────────────────────────────────────────────────┐
│                    上下文增强系统                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  页面元数据  │  │  主题/摘要  │  │    术语表/词汇表     │ │
│  │  收集模块   │  │  收集模块   │  │     匹配模块         │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
│         │                │                     │            │
│         └────────────────┼─────────────────────┘            │
│                          ▼                                  │
│              ┌─────────────────────┐                        │
│              │   Prompt 模板引擎   │                        │
│              │  (变量替换系统)     │                        │
│              └──────────┬──────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │  System Prompt 构建 │                        │
│              └──────────┬──────────┘                        │
│                         │                                   │
│                         ▼                                   │
│              ┌─────────────────────┐                        │
│              │   AI 翻译服务调用   │                        │
│              └─────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

## 核心配置

### 1. 环境变量模板 (env)

位置：`default_config.json` → `translationServices.ai.env`

```json
{
  "imt_source_field": "text",
  "imt_trans_field": "text",
  "imt_sub_source_field": "text",
  "imt_sub_trans_field": "text",

  "title_prompt": "\n\n## Context Awareness\nDocument Metadata:\nTitle: 《{{imt_title}}》",

  "summary_prompt": "\n\n## Context Awareness\nDocument Metadata:\nSummary: {{imt_theme}}...",

  "terms_prompt": "\n\nRequired Terminology: You MUST use the following terms during translation, If 'source':'target', source == target, keep the source term unchanged.\n\n Terms -> \n\n {{imt_terms}}",

  "sub_summary_prompt": "\n\n## Context Awareness\nDocument Metadata:\nType: Subtitle\nSummary: {{imt_theme}}...",

  "sub_terms_prompt": "\n\nRequired Terminology: You MUST use the following terms during translation, If 'source':'target', source == target, keep the source term unchanged.\n\n Terms -> \n\n {{imt_terms}}"
}
```

### 2. System Prompt 模板

#### 普通翻译模式
```
You are a professional, authentic machine translation engine.{{title_prompt}}{{summary_prompt}}{{terms_prompt}}
```

#### 划词翻译模式
```
# Role Definition
You are a professional multilingual translation engine that can translate the provided text into {{to}}.

# Core Capabilities
1. Input Type Recognition:
 - Single word: Provide dictionary functions (phonetic symbols, part of speech, definitions, example sentences)
 - Phrase/Sentence: Return translation only

2. Context Analysis:
【Current Context】: "{{context_text}}"

# Translation Rules
1. For word input:
 - Return complete dictionary information
 - Group definitions by part of speech (keep concise, must use {{to}} language)
 - Provide contextual analysis
 - Include natural context examples

2. For phrase/sentence input:
 - Return translation only
 - No additional information allowed
...
```

### 3. 相关配置项

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enableContextDiffPreview` | boolean | `false` | 启用上下文差异预览（对比有/无上下文的翻译结果） |
| `withAITitleBlockUrls` | array | `[]` | 不收集页面标题的 URL 黑名单 |
| `glossaries` | array | `[{"k":"LLM","v":""},{"k":"LLMs","v":""}]` | 默认术语表 |
| `isTranslateTitle` | boolean | `true` | 是否翻译页面标题 |

## 核心实现

### 1. 上下文变量收集

**位置**: `content_script.js`

```javascript
// 收集页面元数据
a.env.imt_domain = globalThis.location.hostname || ""
a.env.imt_title = globalThis.document.originTitle || globalThis.document.title || ""
```

**收集的变量**:

| 变量名 | 来源 | 说明 |
|--------|------|------|
| `imt_domain` | `location.hostname` | 当前页面域名 |
| `imt_title` | `document.title` | 页面标题（优先使用原始标题） |
| `imt_theme` | 页面分析 | 页面主题/摘要 |
| `imt_terms` | 术语表匹配 | 格式化的术语列表 |
| `context_text` | 选中文本周围 | 划词翻译时的上下文 |

### 2. 术语表处理

**核心函数**: `getValidGlossaries()`

```javascript
getValidGlossaries(t, n, r) {
    // t: 源语言
    // n: 待翻译文本
    // r: 术语表数组

    if (!r || r.length === 0) return [];

    // 1. 提取术语关键词并按长度排序（优先匹配长词）
    let i = r.map(s => ij(s.k)).sort((s, u) => u.length - s.length);

    if (i.length === 0) return [];

    // 2. 根据语言类型构建不同的正则
    let a;
    if (lj(t) || t == "auto") {
        // CJK 语言：不需要单词边界
        a = new RegExp(`(${i.join("|")})`, "gi");
    } else {
        // 非 CJK 语言：需要单词边界 \b
        a = new RegExp(`\\b(${i.join("|")})\\b`, "gi");
    }

    // 3. 从文本中提取匹配的术语
    let o = $1(n, r, a);

    // 4. 去重处理
    o = Fa(o);

    // 5. 更新全局术语缓存
    PC(s => Fa([...s, ...o]));

    return o;
}
```

### 3. Prompt 构建流程

**核心函数**: `hx()` (混淆后的函数名)

```javascript
async function hx(serviceConfig, defaultConfig, context, assistants, specialAssistant) {
    let config = {...defaultConfig};

    // 1. 查找匹配的 AI 助手配置
    let assistant = xh(context, assistants, serviceConfig.assistantId, specialAssistant);

    // 2. 合并助手配置
    if (assistant) {
        px(assistant);  // 处理差异配置
        let env = {...config.env || {}, ...assistant.env || {}};
        Object.assign(config, gx({...assistant, env}));
    }

    // 3. 收集页面上下文（非黑名单 URL）
    if (config.env && globalThis?.location &&
        !G7(globalThis.location.href, context.withAITitleBlockUrls)) {
        config.env.imt_domain = globalThis.location.hostname || "";
        config.env.imt_title = globalThis.document.originTitle ||
                               globalThis.document.title || "";
    }

    // 4. 应用语言覆盖配置
    config = mx(config, config.langOverrides, context);

    // 5. 处理术语表（如果启用）
    if (u0(defaultConfig, assistant)) {
        let terms = await H4();  // 获取上下文术语
        config.contextTerms = terms;
    }

    return config;
}
```

### 4. 变量替换系统

在发送翻译请求前，系统会将模板中的 `{{变量名}}` 替换为实际值：

```javascript
// 伪代码示例
function replaceVariables(template, env) {
    return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
        return env[key] || '';
    });
}
```

## 数据流

```
用户触发翻译
      │
      ▼
┌─────────────────┐
│ 检查 URL 黑名单 │
│ withAITitleBlockUrls │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│ 收集页面元数据  │────▶│ imt_domain      │
│ document.title  │     │ imt_title       │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│ 加载术语表      │────▶│ 用户自定义术语  │
│ glossaries      │     │ AI助手术语      │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│ 术语匹配        │
│ getValidGlossaries │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│ 格式化术语      │────▶│ imt_terms       │
│ handleGlossaries│     │ 'k': 'v' 格式   │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│ 构建 System Prompt │
│ 变量替换        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 调用翻译 API    │
│ OpenAI/Claude/等│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 返回翻译结果    │
└─────────────────┘
```

## 相关文档

- [富文本翻译实现](./rich-text-translation.md)
- [翻译风格（AI 专家）](./translation-styles.md)
- [AI 术语库](./ai-glossary.md)
