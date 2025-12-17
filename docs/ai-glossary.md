# AI 术语库实现分析

## 概述

AI 术语库是确保翻译一致性的重要功能，允许用户定义特定术语的翻译方式。术语库支持：
- 全局术语表（应用于所有翻译）
- AI 助手专属术语（随助手配置加载）
- 领域标记术语（仅在特定领域生效）

## 术语库数据结构

### 基础术语格式

```typescript
interface GlossaryItem {
  k: string;    // 源词（key）
  v: string;    // 译词（value），空字符串表示保留原文
  domain?: string;  // 领域标记（可选）
}
```

### 配置示例

```json
{
  "generalRule": {
    "glossaries": [
      { "k": "LLM", "v": "" },           // 保留原文
      { "k": "API", "v": "API" },         // 保留原文
      { "k": "machine learning", "v": "机器学习" },
      { "k": "kick[Football]", "v": "踢球" }  // 领域特定
    ]
  }
}
```

## 术语优先级体系

```
最高优先级
    │
    ▼
┌─────────────────────────────────────────┐
│  1. AI 助手专属术语 (assistant.terms)   │
│  随翻译风格加载，领域针对性最强         │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  2. 用户自定义术语 (glossaries)         │
│  用户在设置中手动添加的术语             │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  3. 精确域匹配术语                      │
│  带 [domain] 标记且匹配当前页面         │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  4. 无域标记术语（绝对优先）            │
│  在所有页面都强制应用                   │
└─────────────────────────────────────────┘

最低优先级
```

## 核心实现

### 1. 术语合并

```javascript
async function handleGlossaries(sourceLanguage, env, text, glossaries, assistant) {
    // 1. 合并 AI 助手的术语
    if (assistant?.terms) {
        Object.keys(assistant.terms).forEach(key => {
            if (assistant.terms[key]) {
                glossaries.push({
                    k: key,
                    v: assistant.terms[key]
                });
            }
        });
    }

    // 2. 获取文本中实际出现的术语
    let validGlossaries = this.getValidGlossaries(sourceLanguage, text, glossaries);

    if (validGlossaries.length) {
        let hasWithDomain = false;   // 是否有带域标记的术语
        let hasWithoutDomain = false; // 是否有不带域标记的术语

        // 3. 格式化术语为 'source': 'target' 格式
        env.imt_terms = validGlossaries.map(item => {
            let domain = item.domain || "";
            if (domain) {
                domain = `[${domain}]`;
                hasWithDomain = true;
            } else {
                hasWithoutDomain = true;
            }
            // 转义反引号
            return `'${item.k.replace(/`/g, "\\`")}${domain}': '${(item.v || item.k).replace(/`/g, "\\`")}'`;
        }).join(", ");

        // 4. 控制是否显示特定类型的术语提示
        env.imt_terms_with_domain = hasWithDomain ? env.imt_terms_with_domain : "";
        env.imt_terms_without_domain = hasWithoutDomain ? env.imt_terms_without_domain : "";
    }
}
```

### 2. 术语匹配算法

```javascript
getValidGlossaries(sourceLanguage, text, glossaries) {
    if (!glossaries || glossaries.length === 0) return [];

    // 1. 提取术语关键词并按长度排序（优先匹配长词）
    let keywords = glossaries
        .map(item => extractKeyword(item.k))
        .sort((a, b) => b.length - a.length);

    if (keywords.length === 0) return [];

    // 2. 根据语言类型构建正则表达式
    let regex;
    if (isCJK(sourceLanguage) || sourceLanguage === "auto") {
        // CJK 语言不需要单词边界
        regex = new RegExp(`(${keywords.join("|")})`, "gi");
    } else {
        // 非 CJK 语言需要单词边界
        regex = new RegExp(`\\b(${keywords.join("|")})\\b`, "gi");
    }

    // 3. 从文本中提取匹配的术语
    let matched = extractMatches(text, glossaries, regex);

    // 4. 去重
    matched = deduplicate(matched);

    // 5. 更新全局术语缓存
    updateTermsCache(existing => deduplicate([...existing, ...matched]));

    return matched;
}
```

### 3. 关键词提取

```javascript
function extractKeyword(term) {
    // 移除领域标记，如 "kick[Football]" → "kick"
    return term.replace(/\[.*?\]$/, "").trim();
}
```

### 4. 术语在 Prompt 中的应用

```
Required Terminology: You MUST use the following terms during translation.
If 'source':'target', source == target, keep the source term unchanged.

Terms ->

'API': 'API', 'machine learning[AI]': '机器学习', 'kick[Football]': '踢球'
```

## 领域标记详解

### 标记格式

```
源词[领域]
```

### 使用场景

| 术语 | 含义 | 应用范围 |
|------|------|----------|
| `API` | 无领域标记 | 所有页面，强制应用 |
| `kick[Football]` | 足球领域 | 仅在足球相关页面生效 |
| `token[AI]` | AI 领域 | 仅在 AI 相关页面生效 |
| `cell[Biology]` | 生物学领域 | 仅在生物学相关页面生效 |

## 最佳实践

### 术语表配置建议

```json
{
  "glossaries": [
    // 1. 通用技术术语（保留原文）
    { "k": "API", "v": "" },
    { "k": "SDK", "v": "" },
    { "k": "UI", "v": "" },

    // 2. 需要翻译的术语
    { "k": "machine learning", "v": "机器学习" },
    { "k": "deep learning", "v": "深度学习" },

    // 3. 领域特定术语
    { "k": "cell[Biology]", "v": "细胞" },
    { "k": "cell[Excel]", "v": "单元格" },

    // 4. 品牌/专有名词
    { "k": "Anthropic", "v": "" },
    { "k": "Claude", "v": "" }
  ]
}
```

### 自定义 AI 助手术语

```json
{
  "customAiAssistants": [{
    "id": "my-tech-style",
    "terms": {
      "repository": "仓库",
      "pull request": "拉取请求",
      "merge": "合并",
      "branch": "分支"
    }
  }]
}
```

---

## 相关文档

- [上下文增强](./context-enhancement.md)
- [富文本翻译](./rich-text-translation.md)
- [翻译风格（AI 专家）](./translation-styles.md)
