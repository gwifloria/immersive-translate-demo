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

---

# 文本过滤阈值配置

## 完整阈值配置

沉浸式翻译使用以下阈值来过滤不需要翻译的短文本：

```json
{
  "generalRule": {
    // 块级元素阈值（如 div, li, p）
    "blockMinTextCount": 24,      // 最小字符数
    "blockMinWordCount": 4,       // 最小单词数

    // 段落级别阈值（更宽松）
    "paragraphMinTextCount": 2,   // 最小字符数
    "paragraphMinWordCount": 1,   // 最小单词数

    // 其他
    "lineBreakMaxTextCount": 0    // 换行最大文本数，0 = 不限制
  }
}
```

## 正则排除规则

```json
{
  "noTranslateRegexp": [
    "^\\d+.+ago$",                             // "5 minutes ago"
    "^\\d+\\s+MIN\\s+READ$",                   // "3 MIN READ"
    "^[\\u200B\\u200C\\u200D\\u2060\\uFEFF]+$", // 零宽字符
    "^[\\uD800-\\uDBFF][\\uDC00-\\uDFFF]$",    // 单个 emoji
    "^[a-zA-Z]{1}$",                           // 单个字母
    "^\\ue821$",                               // 特殊图标字符
    "<img id=0>",
    "<canvas id=0>"
  ]
}
```

## 判断逻辑

### 核心函数 `_a()` / `Hs()`

```javascript
function _a(options) {
    let {
        noTranslateRegexp,
        minTextCount,      // paragraphMinTextCount 或 blockMinTextCount
        minWordCount,      // paragraphMinWordCount 或 blockMinWordCount
        delimiters,
        text,
        html
    } = options;

    let content = html || text;
    let cleanText = content.trim();

    // 移除变量占位符
    cleanText = cleanText.replace(delimiterRegex, "");
    cleanText = cleanText.trim();

    // 各种跳过条件检查
    if (cleanText === "" ||
        /^[\u200B\u200C\u200D\u2060\uFEFF]+$/.test(cleanText) ||  // 零宽字符
        /^[0-9.,\/#!$%\^&\*;:{}=\-_`~()\s]+$/.test(content) ||   // 纯符号/数字
        cleanText.includes("</style>") ||
        delimiterRegex.test(cleanText) ||
        (noTranslateRegexp && new RegExp(noTranslateRegexp.join("|"), "gi").test(cleanText))
    ) {
        return false;  // 不翻译
    }

    // 最终检查：文本长度和单词数
    return Hs(text?.trim(), minTextCount, minWordCount);
}

function Hs(text, minTextCount, minWordCount) {
    if (!text) return false;

    let charCount = text.length;
    let wordCount = text.split(/\s+/).length;

    // 满足其一即可翻译
    return charCount >= minTextCount || wordCount >= minWordCount;
}
```

### 判断流程

```
┌─────────────────────────────────────────────────────────┐
│                    文本内容                              │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │ 是否为空/零宽字符？    │──是──▶ 不翻译
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 是否纯符号/数字？      │──是──▶ 不翻译
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 匹配 noTranslateRegexp?│──是──▶ 不翻译
              └───────────┬───────────┘
                          │ 否
                          ▼
              ┌───────────────────────┐
              │ 字符数 >= minTextCount │──是──▶ ✅ 翻译
              │ 或                    │
              │ 单词数 >= minWordCount │
              └───────────┬───────────┘
                          │ 否
                          ▼
                      不翻译
```

## 问题案例：Wikipedia 列表内容

### 问题 HTML

```html
<div class="tfa-recent">
  Recently featured:
  <div class="hlist inline">
    <ul>
      <li><a href="...">Littlehampton libels</a></li>
      <li><a href="...">Simon Cameron</a></li>
      <li><a href="...">Commander Keen in Goodbye, Galaxy!</a></li>
    </ul>
  </div>
</div>
```

### 原因分析

| 链接文本 | 字符数 | 单词数 | >= 24 chars 或 >= 4 words? |
|---------|--------|--------|---------------------------|
| "Littlehampton libels" | 20 | 2 | ❌ 不满足 |
| "Simon Cameron" | 13 | 2 | ❌ 不满足 |
| "Commander Keen in Goodbye, Galaxy!" | 35 | 5 | ✅ 满足 |

**结论**：前两个链接因为文本长度不足，被自动跳过翻译。

### 解决方案

**方案 1：降低阈值**

```json
{
  "generalRule": {
    "blockMinTextCount": 10,
    "blockMinWordCount": 2
  }
}
```

**方案 2：配置额外内联选择器**

```json
{
  "rules": [{
    "matches": ["*.wikipedia.org"],
    "additionalInlineSelectors": [".hlist li", ".hlist a"]
  }]
}
```

**方案 3：配置原子块选择器**

```json
{
  "rules": [{
    "matches": ["*.wikipedia.org"],
    "atomicBlockSelectors": [".tfa-recent"]
  }]
}
```

**方案 4：在插件中修改判断逻辑**

```javascript
function shouldTranslate(text, config) {
    let charCount = text.length;
    let wordCount = text.split(/\s+/).length;

    // 如果文本包含有意义的单词，放宽条件
    if (/[a-zA-Z\u4e00-\u9fff]{3,}/.test(text)) {
        return charCount >= 5 || wordCount >= 2;
    }

    return charCount >= config.blockMinTextCount ||
           wordCount >= config.blockMinWordCount;
}
```

## 调试技巧

1. **检查元素翻译状态**
   ```javascript
   element.getAttribute('imt-state')  // "dual", "original", "translation"
   ```

2. **查看当前配置**
   ```javascript
   JSON.parse(localStorage.getItem('immersive_translate_config'))?.generalRule
   ```

3. **强制翻译某个区域**
   ```json
   {
     "selectors": [".your-selector"]
   }
   ```

---

## 相关文档

- [上下文增强](./context-enhancement.md)
- [翻译风格（AI 专家）](./translation-styles.md)
- [AI 术语库](./ai-glossary.md)
