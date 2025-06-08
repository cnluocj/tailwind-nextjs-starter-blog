---
title: '解决 Next.js + TinaCMS + SSG 页面更新缓存问题'
date: '2025-06-08'
tags: ['Next.js', 'TinaCMS', 'SSG', 'Cache', 'Troubleshooting']
draft: false
summary: '记录在使用 Next.js 配合 TinaCMS 做可视化编辑时，遇到的 SSG 页面无法正常更新的缓存问题及其解决方案。'
---

## 问题背景

在开发一个使用 Next.js + TinaCMS 的项目时，我遇到了一个令人困扰的问题：通过 TinaCMS 可视化编辑器修改内容后，前端页面无法正常更新，始终显示旧的内容。这个问题在 SSG（静态站点生成）模式下尤为明显。

## 问题分析

这个问题的根本原因在于多层缓存机制的冲突：

1. **Next.js ISR 缓存**：Next.js 的增量静态再生成会缓存页面
2. **TinaCMS 构建缓存**：TinaCMS 在构建过程中会生成自己的缓存
3. **浏览器缓存**：浏览器端的网络缓存策略
4. **构建工具缓存**：构建过程中产生的临时缓存文件

当内容通过 TinaCMS 更新后，这些缓存层没有正确失效，导致页面显示过期内容。

## 解决方案

经过反复测试和调试，我总结出了以下 4 步解决方案：

### 1. 启用页面重新验证 (Revalidate)

首先需要在 Next.js 页面中正确配置 `revalidate` 选项：

```javascript
// pages/[...slug].js 或 app/[...slug]/page.js
export async function getStaticProps({ params }) {
  // 获取页面数据
  const pageData = await getPageData(params.slug)

  return {
    props: {
      pageData,
    },
    // 设置重新验证时间（秒）
    revalidate: 1, // 1秒后允许重新生成
  }
}
```

或者在 App Router 中：

```javascript
// app/[...slug]/page.js
export const revalidate = 1 // 1秒

export default async function Page({ params }) {
  const pageData = await getPageData(params.slug)
  return <PageComponent data={pageData} />
}
```

### 2. 优化 TinaCMS 构建命令

修改 TinaCMS 构建脚本，跳过客户端构建但保留构建缓存：

```bash
# 正确的 TinaCMS 构建命令
tina build --no-client-build-cache
```

或者在 package.json 中配置：

```json
{
  "scripts": {
    "tina:build": "tina build --no-client-build-cache",
    "build": "tina build --no-client-build-cache && next build"
  }
}
```

参数说明：

- `--no-client-build-cache`：跳过客户端构建但保留构建缓存，这样可以加快构建速度同时避免客户端缓存问题

### 3. 设置网络优先策略

在 TinaCMS 获取数据时设置网络优先策略：

```javascript
const aboutPageResponse = await client.queries.aboutPageConfigConnection(
  {},
  {
    fetchOptions: { fetchPolicy: 'network-only' },
  }
)
```

### 4. 清理 TinaCMS 缓存目录 .cache
