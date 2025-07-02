---
title: '解决 Next.js Build 时特定分类 API 分页失败的诡异 Bug'
date: '2025-06-27'
tags: ['Next.js', 'API', 'Pagination', 'Cache', 'Debug', 'Build']
draft: false
summary: '记录在 Next.js 项目中遇到的一个非常诡异的 Bug：project-notice 分类在 build 时无法正常分页，但其他分类和直接 API 调用都正常。经过深入调试，最终发现是网络请求缓存导致的问题。'
---

## 问题背景

在开发学会官网的过程中，我遇到了一个极其诡异的问题：在 Next.js build 过程中，`project-notice`（项目通知）分类的文章列表始终无法正常获取，而其他分类如 `xuehui-news`（学会新闻）、`science-column`（科普专栏）等都工作正常。

### 问题现象

- ✅ **本地开发环境**：所有功能正常
- ✅ **直接 curl API 调用**：返回正确数据
- ✅ **其他分类**：build 时工作正常
- ❌ **project-notice 在 build 时**：返回空数据

## 问题调试过程

### 第一阶段：API 调用对比

首先，我创建了一个调试脚本来对比不同分类的 API 响应：

```javascript
// scripts/debug-api-categories.js
async function testCategory(category, params = {}) {
  const searchParams = new URLSearchParams()
  searchParams.set('category', category)

  Object.entries(params).forEach(([key, value]) => {
    searchParams.set(key, value.toString())
  })

  const url = `http://localhost:3003/api/articles?${searchParams}`
  console.log(`测试分类: ${category}`)
  console.log(`URL: ${url}`)

  const response = await fetch(url)
  const data = await response.json()
  console.log(`文章数: ${data.data?.articles?.length || 0}`)
}
```

**调试结果发现：**

```bash
# 正常的分类
📂 测试分类: xuehui-news
📊 API响应: { success: true, articlesCount: 8, totalCount: 169 } ✅

# 异常的分类
📂 测试分类: project-notice (带分页参数)
📊 API响应: { success: true, articlesCount: 0, totalCount: 0 } ❌

📂 测试分类: project-notice (不带分页参数)
📊 API响应: { success: true, articlesCount: 1, totalCount: 1 } ✅
```

**关键发现**：`project-notice` 在带分页参数时返回空数据，但不带分页参数时正常！

### 第二阶段：build 时日志分析

在 build 时的日志中发现了关键信息：

```
🔄 [getNotificationsData] 第 1 次尝试（带分页参数）
📊 API响应: { success: true, articlesCount: 0, totalCount: 0 } ❌

🔄 [getNotificationsData] 第 2 次尝试（仅category参数）
📊 API响应: { success: true, articlesCount: 1, totalCount: 1 } ✅
```

这说明问题不在于 API 服务本身，而在于**特定参数组合下的请求处理**。

### 第三阶段：深入后端代码分析

我查看了后端 API 服务器的实现（Express.js）：

```javascript
// scripts/wechat-api-server.js
app.get('/api/articles', (req, res) => {
  const { category, page = 1, limit = 10, refresh } = req.query

  let articles = scanner.getArticles(refresh === 'true')

  // 按分类筛选
  if (category) {
    articles = articles.filter((article) => article.category === category)
  }

  // 分页
  const pageNum = parseInt(page)
  const limitNum = parseInt(limit)
  const startIndex = (pageNum - 1) * limitNum
  const endIndex = startIndex + limitNum

  const paginatedArticles = articles.slice(startIndex, endIndex)
  // ...
})
```

后端逻辑看起来完全正常，分页计算也没有问题。

### 第四阶段：网络请求对比

最关键的发现来自于对比 curl 请求和前端 fetch 请求：

```bash
# 直接 curl 请求 - 完全正常
curl "http://localhost:3003/api/articles?category=project-notice&page=1&limit=8"
# 返回：{"success":true,"data":{"articles":[...1篇文章...]}}

# 前端 build 时的 fetch 请求 - 返回空数据
fetch("http://localhost:3003/api/articles?category=project-notice&page=1&limit=8")
# 返回：{"success":true,"data":{"articles":[]}}
```

**这说明问题出现在前端 fetch 请求的处理上！**

## 解决方案

### 根本原因分析

经过深入分析，我发现问题出在**网络请求的缓存机制**上：

1. **build 时的并发请求**可能触发了缓存冲突
2. **前端 fetch 请求**可能使用了错误的缓存数据
3. **curl 请求**默认无缓存，所以始终正常

### 最终修复

在前端 `wechatApi.ts` 中添加了明确的请求头，特别是 `Cache-Control`：

```javascript
const response = await fetch(requestUrl, {
  signal: AbortSignal.timeout(this.config.apiTimeout),
  headers: {
    Accept: 'application/json',
    'Content-Type': 'application/json',
    'User-Agent': 'NextJS-Build-Client',
    'Cache-Control': 'no-cache', // 🔑 关键修复
  },
})
```

### 辅助改进

1. **改进了响应解析**：

```javascript
// 从直接 JSON 解析改为先获取文本再解析
const responseText = await response.text()
const result = JSON.parse(responseText)
```

2. **添加了重试机制**：

```javascript
// 多种参数组合的重试逻辑
if (retryCount === 0) {
  // 第一次：完整参数
  result = await wechatApi.getArticles({
    category: 'project-notice',
    limit: ITEMS_PER_PAGE,
    page: currentPage,
  })
} else if (retryCount === 1) {
  // 第二次：仅 category 参数
  result = await wechatApi.getArticles({
    category: 'project-notice',
  })
}
```

3. **增强了调试能力**：

```javascript
console.log(`📝 [wechatApi.getArticles] 原始响应:`, responseText.substring(0, 500))
console.log(`🔗 [wechatApi.getArticles] 响应头:`, Object.fromEntries(response.headers.entries()))
```

## 修复效果

修复后的 build 日志：

```
🔧 [getNotificationsData] 使用完整参数: category=project-notice, limit=8, page=1
📡 [wechatApi.getArticles] 响应状态: 200 OK
📊 [wechatApi.getArticles] API响应: { success: true, articlesCount: 1, totalCount: 1 } ✅
🎯 [getNotificationsData] 第 1 次尝试成功获取到文章，跳出重试循环
```

现在 `project-notice` 分页功能在 build 时完全正常工作！

## 经验总结

### 这个 Bug 的特殊性

1. **环境特异性**：只在 build 环境下出现，本地开发正常
2. **分类特异性**：只影响特定分类，其他分类正常
3. **请求方式特异性**：curl 请求正常，fetch 请求异常
4. **参数特异性**：带分页参数异常，不带分页参数正常

### 调试技巧

1. **分层调试**：从前端到后端，从网络到缓存，逐层排查
2. **对比调试**：正常功能 vs 异常功能，找出差异点
3. **请求拦截**：详细记录网络请求的完整信息
4. **日志详化**：在关键节点增加详细的调试日志

### 关键教训

1. **缓存是双刃剑**：能提升性能，也可能导致诡异问题
2. **网络请求头很重要**：`Cache-Control` 等头部对行为有重大影响
3. **环境差异需重视**：开发环境正常不代表生产环境正常
4. **重试机制是保险**：为边界情况提供容错能力

这个 Bug 的解决过程展现了前端调试的复杂性，也提醒我们在处理网络请求时要格外注意缓存机制的影响。最终通过 `Cache-Control: 'no-cache'` 这个简单的修复解决了这个困扰已久的问题。
