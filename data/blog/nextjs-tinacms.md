---
title: Next.js 15 + TinaCMS 可视化编辑完整配置指南
date: '2025-06-18'
tags: ['Next.js', 'TinaCMS', '可视化编辑']
draft: false
summary: 本文档记录了如何在 Next.js 15 (App Router) 项目中完整配置 TinaCMS，实现真正的可视化编辑功能。
---

## 概述

本文档记录了如何在 Next.js 15 (App Router) 项目中完整配置 TinaCMS，实现真正的可视化编辑功能。包括：

- TinaCMS 基础配置
- Next.js Draft Mode 集成
- 可视化编辑组件实现
- 预览模式管理

## 技术栈

- **框架**: Next.js 15 (App Router)
- **CMS**: TinaCMS 2.7.8
- **样式**: Tailwind CSS v4
- **语言**: TypeScript

## 1. 项目初始化和依赖安装

### 安装 TinaCMS 相关包

```bash
npm install tinacms @tinacms/cli @tinacms/auth
```

### 关键依赖说明

- `tinacms`: 核心包，提供 `useTina`、`tinaField` 等 hooks
- `@tinacms/cli`: 命令行工具，用于生成类型和构建
- `@tinacms/auth`: 认证相关功能

## 2. TinaCMS 配置

### 创建 TinaCMS 配置文件

`tina/config.ts`:

```typescript
import { defineConfig } from 'tinacms'

const branch =
  process.env.GITHUB_BRANCH || process.env.VERCEL_GIT_COMMIT_REF || process.env.HEAD || 'main'

export default defineConfig({
  branch,
  clientId: process.env.NEXT_PUBLIC_TINA_CLIENT_ID,
  token: process.env.TINA_TOKEN,

  build: {
    outputFolder: 'admin',
    publicFolder: 'public',
  },

  // 配置预览模式
  admin: {
    auth: {
      onLogin: async () => {
        window.location.href = `/api/preview/enter?secret=${process.env.TINA_PREVIEW_SECRET}&slug=/`
      },
      onLogout: async () => {
        window.location.href = '/api/preview/exit'
      },
    },
  },

  media: {
    tina: {
      mediaRoot: '',
      publicFolder: 'public',
    },
  },

  schema: {
    collections: [
      {
        name: 'global',
        label: 'Global',
        path: 'content/global',
        format: 'json',
        ui: {
          global: true,
        },
        fields: [
          {
            type: 'object',
            name: 'header',
            label: '网站头部',
            fields: [
              {
                type: 'string',
                name: 'title',
                label: '网站标题',
                required: true,
              },
              {
                type: 'string',
                name: 'subtitle',
                label: '网站副标题',
                required: true,
              },
              {
                type: 'image',
                name: 'logo',
                label: 'Logo图片',
              },
              {
                type: 'object',
                name: 'contact',
                label: '联系信息',
                fields: [
                  {
                    type: 'string',
                    name: 'hotline',
                    label: '咨询热线',
                    required: true,
                  },
                  {
                    type: 'string',
                    name: 'hotlineLabel',
                    label: '热线标签',
                    required: true,
                  },
                  {
                    type: 'image',
                    name: 'qrcode',
                    label: '二维码图片',
                  },
                ],
              },
            ],
          },
          {
            type: 'object',
            name: 'navigation',
            label: '导航菜单',
            list: true,
            fields: [
              {
                type: 'string',
                name: 'name',
                label: '菜单名称',
                required: true,
              },
              {
                type: 'string',
                name: 'href',
                label: '链接地址',
                required: true,
              },
              {
                type: 'boolean',
                name: 'enabled',
                label: '启用',
                description: '是否显示此菜单项',
              },
            ],
          },
        ],
      },
    ],
  },
})
```

### Package.json 脚本配置

```json
{
  "scripts": {
    "dev": "TINA_PUBLIC_IS_LOCAL=true tinacms dev -c \"next dev --turbopack\"",
    "build": "tinacms build && next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

## 3. 环境变量配置

### .env.example

```bash
# TinaCMS Configuration
NEXT_PUBLIC_TINA_CLIENT_ID=your_tina_client_id_here
TINA_TOKEN=your_tina_token_here

# Preview Mode Secret
TINA_PREVIEW_SECRET=your_preview_secret_here

# Next.js Configuration
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

## 4. Next.js Draft Mode 配置

### 预览模式 API 路由

`src/app/api/preview/enter/route.ts`:

```typescript
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const secret = searchParams.get('secret')
  const slug = searchParams.get('slug') || '/'

  if (secret !== process.env.TINA_PREVIEW_SECRET) {
    return new Response('Invalid token', { status: 401 })
  }

  const draft = await draftMode()
  draft.enable()

  console.log('🎭 [Preview] 启用预览模式，重定向到:', slug)
  redirect(slug)
}
```

`src/app/api/preview/exit/route.ts`:

```typescript
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'

export async function GET() {
  const draft = await draftMode()
  draft.disable()

  console.log('🚫 [Preview] 退出预览模式')
  redirect('/')
}
```

## 5. TinaProvider 组件

`src/components/TinaProvider.tsx`:

```tsx
'use client'

import { ReactNode } from 'react'
import { useTina } from 'tinacms/dist/react'

interface TinaProviderProps {
  children: ReactNode
  data: any // TinaCMS数据
}

export default function TinaProvider({ children, data }: TinaProviderProps) {
  // 在预览模式下启用TinaCMS编辑功能
  if (data) {
    const { data: tinaData } = useTina({
      query: data.query || '',
      variables: data.variables || {},
      data: data.data || {},
    })

    return <>{children}</>
  }

  return <>{children}</>
}
```

## 6. 可视化编辑组件实现

### Header 组件示例

`src/components/Header.tsx`:

```tsx
'use client'

import React from 'react'
import Image from 'next/image'
import { useTina, tinaField } from 'tinacms/dist/react'

interface HeaderConfig {
  title: string
  subtitle: string
  logo?: string
  contact: {
    hotline: string
    hotlineLabel: string
    qrcode?: string
  }
}

interface HeaderProps {
  config: HeaderConfig
  tinaData?: any // TinaCMS原始数据
}

const Header: React.FC<HeaderProps> = ({ config, tinaData }) => {
  console.log('🎨 [Header] 组件开始渲染')

  // 在预览模式下使用TinaCMS编辑功能
  const { data: editableData } = useTina({
    query: tinaData?.query || '',
    variables: tinaData?.variables || {},
    data: tinaData?.data || { global: null },
  })

  // 在预览模式下优先使用实时数据，否则使用SSG配置
  let headerConfig = config

  if (editableData?.global) {
    console.log('🎭 [Header] 检测到预览模式，使用实时TinaCMS数据')
    const globalData = editableData.global

    headerConfig = {
      title: globalData.header?.title || config.title,
      subtitle: globalData.header?.subtitle || config.subtitle,
      logo: globalData.header?.logo || config.logo,
      contact: {
        hotline: globalData.header?.contact?.hotline || config.contact.hotline,
        hotlineLabel: globalData.header?.contact?.hotlineLabel || config.contact.hotlineLabel,
        qrcode: globalData.header?.contact?.qrcode || config.contact.qrcode,
      },
    }
  }

  // 获取TinaCMS数据用于编辑标记
  const globalNode = editableData?.global

  return (
    <header className="border-b border-gray-300 bg-white px-4 py-6">
      <div className="container mx-auto flex items-center justify-between">
        {/* Logo */}
        <div
          className="mr-6 flex h-20 w-20 items-center justify-center rounded-full bg-blue-100"
          data-tina-field={globalNode && tinaField(globalNode, 'header.logo')}
        >
          {headerConfig.logo && headerConfig.logo.trim() !== '' ? (
            <Image
              src={headerConfig.logo}
              alt="Logo"
              width={64}
              height={64}
              className="rounded-full"
            />
          ) : (
            <div className="flex h-16 w-16 items-center justify-center rounded-full bg-blue-500">
              <span className="text-sm font-bold text-white">医</span>
            </div>
          )}
        </div>

        {/* 标题 */}
        <div className="text-left">
          <h1
            className="mb-2 text-4xl font-bold text-gray-800"
            data-tina-field={globalNode && tinaField(globalNode, 'header.title')}
          >
            {headerConfig.title}
          </h1>
          <p
            className="text-base tracking-wide text-gray-600"
            data-tina-field={globalNode && tinaField(globalNode, 'header.subtitle')}
          >
            {headerConfig.subtitle}
          </p>
        </div>

        {/* 联系信息 */}
        <div className="flex items-center space-x-6">
          {/* 二维码 */}
          <div
            className="flex h-20 w-20 items-center justify-center border-2 border-gray-400 bg-gray-100"
            data-tina-field={globalNode && tinaField(globalNode, 'header.contact.qrcode')}
          >
            {headerConfig.contact.qrcode && headerConfig.contact.qrcode.trim() !== '' ? (
              <Image
                src={headerConfig.contact.qrcode}
                alt="咨询二维码"
                width={76}
                height={76}
                className="object-cover"
              />
            ) : (
              <div className="h-16 w-16 bg-gray-300"></div>
            )}
          </div>

          <div className="text-right">
            <p
              className="mb-1 text-sm text-gray-500"
              data-tina-field={globalNode && tinaField(globalNode, 'header.contact.hotlineLabel')}
            >
              {headerConfig.contact.hotlineLabel}
            </p>
            <p
              className="text-2xl font-bold text-blue-600"
              data-tina-field={globalNode && tinaField(globalNode, 'header.contact.hotline')}
            >
              {headerConfig.contact.hotline}
            </p>
          </div>
        </div>
      </div>
    </header>
  )
}

export default Header
```

## 7. Layout 集成

`src/app/layout.tsx`:

```tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import Header from '../components/Header'
import Navigation from '../components/Navigation'
import TinaProvider from '../components/TinaProvider'
import client from '../../tina/__generated__/client'
import { draftMode } from 'next/headers'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',
})

export const metadata: Metadata = {
  title: '广东省肿瘤康复学会',
  description: '广东省肿瘤康复学会官方网站',
}

async function getGlobalData() {
  try {
    const result = await client.queries.global({ relativePath: 'index.json' })
    const global = result.data.global

    // 转换数据格式
    const headerConfig = {
      title: global.header?.title || '广东省肿瘤康复学会',
      subtitle: global.header?.subtitle || 'GUANGDONG CANCER REHABILITATION SOCIETY',
      logo: global.header?.logo || undefined,
      contact: {
        hotline: global.header?.contact?.hotline || '020-84829789',
        hotlineLabel: global.header?.contact?.hotlineLabel || '7*24小时咨询热线',
        qrcode: global.header?.contact?.qrcode || undefined,
      },
    }

    const navigationConfig =
      global.navigation
        ?.filter((item) => item !== null)
        .map((item) => ({
          name: item!.name,
          href: item!.href,
          enabled: item!.enabled ?? true,
        })) || []

    return {
      data: result.data,
      query: result.query,
      variables: result.variables,
      header: headerConfig,
      navigation: navigationConfig,
      tinaData: result, // 保存完整的TinaCMS数据
    }
  } catch (error) {
    console.error('Error fetching global data:', error)
    // 返回默认数据...
  }
}

export default async function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode
}>) {
  const globalData = await getGlobalData()
  const { isEnabled: isPreview } = await draftMode()

  console.log('🎭 [RootLayout] 预览模式:', isPreview)

  const content = (
    <>
      <Header config={globalData.header} tinaData={globalData.tinaData} />
      <Navigation items={globalData.navigation} tinaData={globalData.tinaData} />
      <main>{children}</main>
    </>
  )

  return (
    <html lang="zh-CN">
      <body className={`${inter.variable} font-sans antialiased`}>
        {isPreview ? <TinaProvider data={globalData.tinaData}>{content}</TinaProvider> : content}
      </body>
    </html>
  )
}
```

## 8. 内容数据文件

`content/global/index.json`:

```json
{
  "header": {
    "title": "广东省肿瘤康复学会",
    "subtitle": "GUANGDONG CANCER REHABILITATION SOCIETY",
    "logo": "",
    "contact": {
      "hotline": "020-84829789",
      "hotlineLabel": "7*24小时咨询热线",
      "qrcode": ""
    }
  },
  "navigation": [
    {
      "name": "首页",
      "href": "/",
      "enabled": true
    },
    {
      "name": "关于学会",
      "href": "/about",
      "enabled": true
    }
  ]
}
```

## 9. 使用流程

### 开发环境设置

1. 复制 `.env.example` 到 `.env.local`
2. 填入 TinaCMS 凭据和预览密钥
3. 运行 `npm run dev`

### 编辑流程

1. 访问 `/admin` 进入 TinaCMS 管理界面
2. 编辑内容后，TinaCMS 自动启用预览模式
3. 在预览模式下，页面元素显示编辑标记
4. 点击编辑标记可直接在页面上编辑
5. 编辑完成后内容实时同步

### 可编辑元素

- **文本内容**: 使用 `data-tina-field={tinaField(node, 'fieldPath')}`
- **图片内容**: 使用 `data-tina-field={tinaField(node, 'imagePath')}`
- **列表内容**: 使用 `data-tina-field={tinaField(node, 'listPath')}`

## 10. 关键技术要点

### 1. 模式区分

- **生产模式**: 使用静态数据，性能最佳
- **预览模式**: 使用 TinaCMS 实时数据，支持可视化编辑

### 2. 数据流

```
TinaCMS Admin → Draft Mode → useTina Hook → tinaField → 可视化编辑
```

### 3. 组件设计原则

- 支持 SSG 静态数据和 TinaCMS 实时数据
- 只在预览模式下添加编辑标记
- 保持原有交互功能（如导航点击）
- 提供合理的默认值和占位符

### 4. 性能优化

- TinaProvider 只在预览模式下包装组件
- 使用条件渲染避免不必要的 TinaCMS 代码加载
- 静态数据优先，实时数据作为增强

## 11. 常见问题和解决方案

### 图片路径问题

如果图片路径被 TinaCMS 编码导致错误，可以：

1. 使用 `skipPaths` 配置跳过特定字段
2. 在组件中对图片路径进行额外处理
3. 使用原始数据作为 fallback

### 类型安全

- 使用 TinaCMS 生成的类型文件
- 为组件 props 定义明确的接口
- 使用可选链操作符处理可能为空的数据

### 调试技巧

- 在组件中添加 console.log 查看数据流
- 使用浏览器开发工具检查 `data-tina-field` 属性
- 通过预览模式 API 路由测试状态切换

## 总结

通过以上配置，可以实现：

1. **真正的可视化编辑**: 点击页面元素直接编辑
2. **性能优化**: 预览模式和生产模式分离
3. **用户体验**: 保持原有交互功能
4. **开发体验**: 类型安全和调试友好

这套方案适用于任何需要内容管理功能的 Next.js 项目，可以根据具体需求调整 schema 配置和组件实现。
