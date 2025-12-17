# 沉浸式翻译 - 技术文档

本目录包含沉浸式翻译 Chrome 扩展的核心功能实现分析。

## 文档索引

| 文档 | 说明 |
|------|------|
| [上下文增强](./context-enhancement.md) | 页面元数据收集、术语表处理、Prompt 构建流程 |
| [富文本翻译](./rich-text-translation.md) | HTML 标签分类、文本过滤阈值、调试指南 |
| [翻译风格](./translation-styles.md) | AI 专家/助手系统、自动匹配、配置合并 |
| [AI 术语库](./ai-glossary.md) | 术语优先级、领域标记、匹配算法 |

## 快速参考

### 文本过滤阈值

```json
{
  "blockMinTextCount": 24,   // 块级元素最小字符数
  "blockMinWordCount": 4,    // 块级元素最小单词数
  "paragraphMinTextCount": 2 // 段落最小字符数
}
```

### 标签分类

- **inlineTags**: A, SPAN, STRONG, EM, I, B... → 与周围文本一起翻译
- **stayOriginalTags**: CODE, TT, math... → 保留原文
- **excludeTags**: SCRIPT, STYLE, SVG... → 完全跳过

### AI 助手 ID

```
tech, paper, news, medical, legal, github, game,
ecommerce, financial, fiction, ao3, ebook, web3...
```

## 相关链接

- [扩展配置](../default_config.json)
- [内容脚本](../content_script.js)
- [Manifest](../manifest.json)
