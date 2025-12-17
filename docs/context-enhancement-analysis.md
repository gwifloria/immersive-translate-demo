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

### 3. 术语表格式化

**核心函数**: `handleGlossaries()`

```javascript
async handleGlossaries(t, n, r, i, a) {
    // t: 源语言
    // n: 环境变量对象
    // r: 待翻译文本
    // i: 术语表
    // a: AI 配置

    // 合并 AI 助手自带的术语
    if (a?.terms) {
        Object.keys(a.terms).forEach(u => {
            a.terms[u] && i.push({
                k: u,
                v: a.terms[u]
            });
        });
    }

    // 获取有效术语
    let o = this.getValidGlossaries(t, r, i);

    if (o.length) {
        let s = false,  // 是否有带域标记的术语
            u = false;  // 是否有不带域标记的术语

        // 格式化术语为 'source': 'target' 格式
        n.imt_terms = o.map(l => {
            let c = l.domain || "";
            if (c) {
                c = `[${c}]`;
                s = true;
            } else {
                u = true;
            }
            return `'${l.k.replace(/`/g, "\\`")}${c}': '${(l.v || l.k).replace(/`/g, "\\`")}'`;
        }).join(", ");

        // 控制是否显示特定类型的术语提示
        n.imt_terms_with_domain = s ? n.imt_terms_with_domain : "";
        n.imt_terms_without_domain = u ? n.imt_terms_without_domain : "";
    }
}
```

**术语格式示例**:
```
'API': 'API',
'machine learning[AI]': '机器学习',
'kick': 'kick',
'LLM[Tech]': '大语言模型'
```

### 4. Prompt 构建流程

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

### 5. 变量替换系统

在发送翻译请求前，系统会将模板中的 `{{变量名}}` 替换为实际值：

```javascript
// 伪代码示例
function replaceVariables(template, env) {
    return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
        return env[key] || '';
    });
}

// 示例
let systemPrompt = "You are a professional machine translation engine.{{title_prompt}}{{terms_prompt}}";
let env = {
    title_prompt: "\n\n## Context Awareness\nDocument Metadata:\nTitle: 《How to Learn Programming》",
    terms_prompt: "\n\nRequired Terminology: ...\n\n Terms -> \n\n 'API': 'API', 'function': '函数'"
};

// 替换后
// "You are a professional machine translation engine.
//
// ## Context Awareness
// Document Metadata:
// Title: 《How to Learn Programming》
//
// Required Terminology: ...
//
//  Terms ->
//
//  'API': 'API', 'function': '函数'"
```

### 6. 条件 Prompt 包含

系统会根据变量是否存在来决定是否包含某个 prompt 片段：

```javascript
// 如果 imt_title 为空，则 title_prompt 也设为空
if (!env.imt_title) {
    env.title_prompt = "";
}

// 如果 imt_theme 为空，则 summary_prompt 也设为空
if (!env.imt_theme) {
    env.summary_prompt = "";
}

// 如果没有匹配的术语，则 terms_prompt 也设为空
if (!env.imt_terms) {
    env.terms_prompt = "";
}
```

## 上下文差异预览

当 `enableContextDiffPreview` 为 `true` 时，系统会：

1. **两次翻译**：
   - 第一次：带上下文术语
   - 第二次：不带上下文术语

2. **差异对比**：使用 `J1()` 函数对比两次翻译结果

3. **高亮显示**：在界面上展示差异，帮助用户理解上下文增强的效果

## 术语表优先级

```
高优先级
    │
    ▼
┌─────────────────────────────────┐
│  精确域匹配的术语               │
│  例: 'kick[Football]': '踢球'   │
│  在足球相关页面优先使用         │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  无域标记的术语（绝对优先）     │
│  例: 'API': 'API'               │
│  在所有页面都使用               │
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  带域标记但不匹配当前页面       │
│  这些术语会被过滤掉             │
└─────────────────────────────────┘

低优先级
```

## 关键文件索引

| 文件 | 说明 |
|------|------|
| `default_config.json` | 主配置文件，包含所有 prompt 模板和默认配置 |
| `content_script.js` | 内容脚本，负责上下文收集和术语匹配 |
| `background.js` | 后台服务，处理翻译 API 调用 |
| `popup.js` | 弹窗界面，包含相同的翻译逻辑 |
| `side-panel.js` | 侧边栏界面，包含相同的翻译逻辑 |
| `options.js` | 设置页面，包含上下文配置选项 |

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

## 最佳实践

### 1. 术语表设置

```json
{
  "glossaries": [
    {"k": "API", "v": "API"},           // 保持原文
    {"k": "function", "v": "函数"},      // 固定翻译
    {"k": "kick[Football]", "v": "踢球"} // 领域特定翻译
  ]
}
```

### 2. URL 黑名单

对于标题可能包含敏感信息或误导翻译的网站：

```json
{
  "withAITitleBlockUrls": [
    "https://mail.google.com/*",
    "https://*.bank.com/*"
  ]
}
```

### 3. 自定义 Prompt

可以通过 AI 助手配置自定义 prompt 模板，覆盖默认行为。

## 性能考虑

1. **术语匹配优化**：术语按长度降序排列，优先匹配长术语
2. **正则缓存**：编译后的正则表达式可复用
3. **条件包含**：空变量对应的 prompt 片段不会包含在最终请求中
4. **缓存机制**：术语表结果会被缓存，避免重复计算

## 总结

沉浸式翻译的上下文增强功能通过以下方式提升翻译质量：

1. **页面感知**：自动收集页面标题、域名等元数据
2. **术语一致性**：通过术语表确保专业术语翻译统一
3. **智能匹配**：只包含文本中实际出现的术语
4. **领域适配**：支持领域特定的术语翻译
5. **灵活配置**：支持通过配置自定义各个环节

---

# 富文本翻译实现分析

## 概述

沉浸式翻译支持富文本翻译，即在翻译时智能处理 HTML 标签。核心问题是：**哪些标签内容需要翻译，哪些需要保留原文？**

## 标签分类体系

### 1. inlineTags（内联标签）

内联标签内的内容会**与周围文本一起翻译**，标签结构保留。

```json
"inlineTags": [
  "A", "ABBR", "FONT", "ACRONYM", "B", "INS", "DEL",
  "RUBY", "RP", "RB", "BDO", "MARK", "BIG", "RT", "NOBR",
  "CITE", "DFN", "EM", "I", "LABEL", "Q", "S", "SMALL",
  "SPAN", "STRONG", "SUB", "SUP", "U", "KBD", "TT", "VAR",
  "IMG", "CODE", "TIME", "WBR"
  // ... 更多
]
```

**示例**：
```html
<!-- 原文 -->
<p>This is a <strong>bold</strong> and <em>italic</em> text.</p>

<!-- 翻译后 -->
<p>这是一段<strong>加粗</strong>和<em>斜体</em>的文字。</p>
```

### 2. stayOriginalTags（保留原文标签）

这些标签内的内容**不翻译，保留原文**。

```json
"stayOriginalTags": [
  "CODE",      // 代码
  "TT",        // 等宽字体
  "IMG",       // 图片
  "SUP",       // 上标
  "SUB",       // 下标
  "SAMP",      // 示例输出
  "math",      // 数学公式相关
  "semantics",
  "mrow",
  "mo",
  "mfrac",
  "msup",
  "mi",
  "mn",
  "msqrt",
  "d-math"
]
```

**示例**：
```html
<!-- 原文 -->
<p>Use the <code>console.log()</code> function to debug.</p>

<!-- 翻译后 -->
<p>使用 <code>console.log()</code> 函数来调试。</p>
<!-- CODE 标签内容保持不变 -->
```

### 3. excludeTags（排除标签）

这些标签**完全跳过**，不参与翻译过程。

```json
"excludeTags": [
  "TITLE",     // 标题（在 head 中）
  "LINK",      // 链接
  "SCRIPT",    // 脚本
  "STYLE",     // 样式
  "TEXTAREA",  // 文本域
  "SVG",       // SVG 图形
  "NOSCRIPT",  // 无脚本内容
  "BUTTON",    // 按钮
  "PRE",       // 预格式化文本
  "KBD",       // 键盘输入
  "META",      // 元信息
  "MATH"       // 数学公式
  // ... 更多
]
```

### 4. atomicBlockSelectors（原子块选择器）

这些元素作为**整体翻译单元**处理。

```json
"atomicBlockSelectors": [
  "relin-hc",
  "x-p",
  "app-keyword-content"
]
```

## 核心判断逻辑

### 判断流程图

```
┌─────────────────────────────────────────────────────────┐
│                    遍历 DOM 节点                         │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │ 是否有 translate="no" │──是──▶ 跳过不翻译
              │ 或 class="notranslate"│
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 标签在 excludeTags 中？│──是──▶ 完全跳过
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 标签在 stayOriginalTags│──是──▶ 保留原文
              │ 且是块级元素？         │       （不翻译内容）
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 标签在 inlineTags 中？ │──是──▶ 作为内联元素
              │                       │       与父段落一起翻译
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 检查 CSS display 属性  │
              │ block/flex → 块级元素  │
              │ inline → 内联元素      │
              └───────────────────────┘
```

### 核心函数实现

#### 1. 判断是否排除/保留原文 - `zb()`

```javascript
function zb(element, config, checkExclude) {
    // 检查节点类型
    if (!(element.nodeType === Node.ELEMENT_NODE ||
          element.nodeType === Node.TEXT_NODE)) {
        return true;  // 跳过
    }

    // 检查 translate="no" 或 notranslate class
    if (element.nodeType === Node.ELEMENT_NODE &&
        (element.getAttribute("translate") === "no" ||
         element.classList.contains("notranslate"))) {
        return true;  // 跳过
    }

    let { stayOriginalTags, excludeTags } = config;
    let tagsToCheck = [];

    if (checkExclude && excludeTags?.length > 0) {
        tagsToCheck = excludeTags;
    } else {
        // 排除 stayOriginalTags 中也存在于 excludeTags 的标签
        tagsToCheck = excludeTags.filter(tag => !stayOriginalTags.includes(tag));
    }

    // 检查标签是否在排除列表中
    return Hl(element.nodeName, tagsToCheck);
}
```

#### 2. 判断是否为内联元素 - `o4()`

```javascript
function o4(element, config) {
    let inlineTags = config.inlineTags;

    if (element.nodeType === Node.ELEMENT_NODE) {
        // 如果在 inlineTags 中，或者是未知标签
        if (Hl(element.nodeName, inlineTags) || a4(element, config)) {
            // 排除 BR 标签
            if (Hl(element.nodeName, ["BR"])) {
                return false;
            }

            // 检查未知标签的 CSS display
            if (a4(element, config)) {
                let style = globalThis.getComputedStyle(element);
                if (style.display === "block" || style.display === "flex") {
                    return false;  // 块级元素
                }
            }

            // 递归检查子元素是否都是内联
            return BN(element, config);
        }
    }
    return false;
}
```

#### 3. 检查标签名是否匹配 - `Hl()`

```javascript
function Hl(nodeName, tagList) {
    if (!nodeName || !tagList) return false;

    if (!Array.isArray(tagList)) {
        tagList = [tagList];
    }

    nodeName = nodeName.toUpperCase();

    for (let tag of tagList) {
        if (nodeName === tag) {
            return true;
        }
    }
    return false;
}
```

#### 4. 判断是否为未知标签 - `a4()`

```javascript
function a4(element, config) {
    // 合并所有已知标签
    let allKnownTags = config.allBlockTags
        .concat(config.inlineTags)
        .concat(config.excludeTags);

    // 如果标签不在任何已知列表中，则为"未知"
    return !Hl(element.nodeName, allKnownTags);
}
```

## 实际应用场景

### 场景 1: 链接 `<a>` 标签

```html
<!-- 原文 -->
<p>Click <a href="/docs">here</a> for more information.</p>

<!-- 翻译后 -->
<p>点击<a href="/docs">这里</a>获取更多信息。</p>
```

`<a>` 在 `inlineTags` 中，所以链接文本 "here" 会被翻译为 "这里"。

### 场景 2: 代码 `<code>` 标签

```html
<!-- 原文 -->
<p>The <code>map()</code> method creates a new array.</p>

<!-- 翻译后 -->
<p><code>map()</code> 方法创建一个新数组。</p>
```

`<code>` 在 `stayOriginalTags` 中，所以 "map()" 保持原样。

### 场景 3: 数学公式

```html
<!-- 原文 -->
<p>The formula is <math><mi>E</mi><mo>=</mo><mi>m</mi><msup><mi>c</mi><mn>2</mn></msup></math>.</p>

<!-- 翻译后 -->
<p>公式是 <math><mi>E</mi><mo>=</mo><mi>m</mi><msup><mi>c</mi><mn>2</mn></msup></math>。</p>
```

数学公式相关标签都在 `stayOriginalTags` 中，完整保留。

### 场景 4: 复杂嵌套

```html
<!-- 原文 -->
<p>
  Use <strong><code>async/await</code></strong> for
  <em>asynchronous</em> operations.
</p>

<!-- 翻译后 -->
<p>
  使用 <strong><code>async/await</code></strong> 进行
  <em>异步</em>操作。
</p>
```

- `<strong>` 和 `<em>` 是内联标签，内容会翻译
- `<code>` 在 stayOriginal 中，"async/await" 保持原样

## 配置扩展

### 添加自定义内联选择器

```json
"additionalInlineSelectors": [
  ".MathJax_Preview",
  ".MathJax",
  ".highlighter--highlighted",
  "ruby *",
  "p > button"
]
```

### 添加自定义排除选择器

```json
"additionalExcludeSelectors": [
  "[contenteditable=\"true\"]",
  "#immersive-translate-popup",
  ".prism-code",
  "[role=code]"
]
```

### 站点特定规则

可以在 `rules` 数组中为特定网站自定义标签处理：

```json
{
  "matches": ["docs.github.com"],
  "stayOriginalTags": ["CODE", "PRE", "VAR"],
  "additionalInlineSelectors": [".markdown-body a"]
}
```

## 性能优化

1. **标签名缓存**：标签名会被转换为大写后比较，避免重复转换
2. **CSS 计算缓存**：`getComputedStyle` 结果会被缓存到元素上
3. **批量处理**：相邻的内联元素会合并为一个翻译单元
4. **提前退出**：遇到排除标签时直接跳过子树遍历

---

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
    // ... 其他变量
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

### 3. 配置提取 - `gx()` 函数

```javascript
function gx(assistant) {
    let config = {
        env: assistant?.env,
        prompt: assistant?.prompt,
        systemPrompt: assistant?.systemPrompt,
        multipleSystemPrompt: assistant?.multipleSystemPrompt,
        multiplePrompt: assistant?.multiplePrompt,
        subtitlePrompt: assistant?.subtitlePrompt,
        langOverrides: assistant?.langOverrides,
        temperature: assistant?.temperature,
        maxTextGroupLengthPerRequest: assistant?.maxTextGroupLengthPerRequest,
        maxTextLengthPerRequest: assistant?.maxTextLengthPerRequest,
        maxTokensRatio: assistant?.maxTokensRatio,
        minTokensRatio: assistant?.minTokensRatio,
        maxTextGroupLengthPerRequestForSubtitle:
            assistant?.maxTextGroupLengthPerRequestForSubtitle
    };

    // 移除空值
    for (let key in config) {
        if (config[key] == null) delete config[key];
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

### 版本检查

```javascript
// 检查助手版本是否兼容
function dx(assistant) {
    return Kn(Aa(), assistant.extensionVersion);  // 比较扩展版本
}

// 检查是否需要更新
function Q_(assistant, remoteVersion) {
    if (!remoteVersion) return false;
    return Kn(assistant.version, remoteVersion);  // 版本过旧需要更新
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

### 助手管理

```javascript
// 添加或编辑助手
async function Wd(action, assistant, localConfig) {
    localConfig = localConfig || await An();
    let assistants = localConfig.aiAssistants || [];
    let updated = false;

    if (action === "edit" && dx(assistant)) {
        // 编辑现有助手
        for (let i = assistants.length - 1; i >= 0; i--) {
            if (assistants[i].id === assistant.id) {
                assistants[i] = assistant;
                updated = true;
            }
        }
    } else if (action === "add" && dx(assistant)) {
        // 添加新助手（先删除同 ID 的旧版本）
        for (let i = assistants.length - 1; i >= 0; i--) {
            if (assistants[i].id === assistant.id) {
                assistants.splice(i, 1);
            }
        }
        assistants.push(assistant);
        updated = true;
    } else {
        // 删除助手
        for (let i = assistants.length - 1; i >= 0; i--) {
            if (assistants[i].id === assistant.id) {
                assistants.splice(i, 1);
            }
        }
        updated = true;
    }

    // 按优先级排序并保存
    localConfig.aiAssistants = assistants.sort((a, b) => a.priority - b.priority);

    // 更新助手 ID 列表
    let syncConfig = await Gt();
    syncConfig.aiAssistantIds = [...new Set(assistants.map(a => a.id))];

    await ai(localConfig);
    await Fn(syncConfig);

    return updated;
}
```

---

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

### 领域匹配逻辑

```javascript
function shouldApplyTerm(term, currentDomain) {
    let domain = extractDomain(term.k);

    if (!domain) {
        // 无领域标记，始终应用
        return true;
    }

    // 检查当前页面是否匹配该领域
    return currentDomain === domain;
}
```

## 术语同步机制

### 本地存储

```javascript
// 术语表存储在 generalRule.glossaries
{
  "generalRule": {
    "glossaries": [...]
  }
}
```

### 同步到云端

```javascript
// 独立同步的键
const independentSyncKeys = [
  "generalRule.glossaries",  // 术语表
  "generalRule.injectedCss",
  "aiAssistantsMatches",
  "customAiAssistants"
];
```

## 性能优化

1. **长词优先**：术语按长度降序排列，避免短词误匹配长词的一部分
2. **正则预编译**：正则表达式编译后可复用
3. **增量更新**：只添加新出现的术语到缓存
4. **延迟加载**：AI 助手术语在需要时才加载
5. **去重处理**：避免重复术语影响翻译质量

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
