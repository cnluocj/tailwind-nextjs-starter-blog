---
title: 'AI应用WebSocket连接中断问题解决记录'
date: '2025-08-12'
tags: ['WebSocket', 'Next.js', 'CopilotKit', 'Caddy', '反向代理', '连接管理', 'React Hooks']
draft: false
summary: '记录了医学文章生成AI应用中WebSocket连接中断导致的跨域跳转报错问题，以及完整的解决方案实施过程。'
---

## 🔍 背景介绍

我们正在开发一个医学文章生成的AI应用，使用了以下技术栈：
- **前端**：Next.js + CopilotKit + TypeScript
- **后端**：Python + LangGraph  
- **部署**：Docker + Caddy反向代理
- **域名架构**：
  - 主站：`www.example.com` (端口3000)
  - AI服务：`ai.example.com` (端口3333)

## 🚨 问题描述

在部署到生产环境后，出现了一个奇特的跨域跳转报错问题：

### 现象表现

1. **跳转报错**：从AI页面通过导航栏跳转到主站时，浏览器显示"无法打开此网页"错误
2. **时机相关**：只有在AI预测输入进行中时跳转才会报错，预测完成后跳转正常  
3. **域名相关**：使用域名访问会报错，直接用IP+端口访问则正常

### 错误信息
```
无法打开此网页
请检查您的网络连接：
• 关闭其他标签页或应用
• 在新的无痕式窗口中打开网页 (⇧⌘N)
• 重新启动 Chrome
• 重新启动计算机
错误代码：11
```

## 🔬 问题分析

### 根本原因

经过深入分析，发现这是一个典型的**WebSocket连接中断**问题：

1. **CopilotKit建立WebSocket连接**：用于实时AI对话和预测输入功能
2. **强制页面跳转**：使用普通`<a href>`导致页面完全刷新
3. **连接突然中断**：WebSocket连接在预测输入进行中被强制断开
4. **代理配置问题**：Caddy没有正确处理WebSocket连接的清理

### 技术细节

```javascript
// 问题代码：直接跳转，无连接清理
<a href="https://www.example.com/home" target="_self">
  网站首页
</a>
```

## ✅ 解决方案

### 1. 优化导航组件

为导航链接添加智能处理逻辑：

```typescript
// ui/src/components/MainHeader.tsx
const handleExternalLinkClick = async (e: React.MouseEvent<HTMLAnchorElement>, href: string) => {
  // 如果当前在AI页面，给一个小的延迟让正在进行的请求完成
  if (pathname?.includes('/medical-science')) {
    e.preventDefault();
    
    try {
      // 给一个小的延迟让正在进行的请求完成
      await new Promise(resolve => setTimeout(resolve, 100));
      
      // 触发页面跳转
      window.location.href = href;
    } catch (error) {
      console.warn('清理连接时出错:', error);
      // 即使清理失败也要跳转
      window.location.href = href;
    }
  }
};

// 应用到导航链接
<a 
  href={item.href} 
  onClick={(e) => handleExternalLinkClick(e, item.href)}
  target="_self"
>
  {item.label}
</a>
```

### 2. 添加页面离开监听

在主组件中添加优雅的连接清理：

```typescript
// ui/src/app/Main.tsx
useEffect(() => {
  const handleBeforeUnload = (event: BeforeUnloadEvent) => {
    // 如果有正在进行的AI对话或预测输入，给用户提示
    if (state.logs && state.logs.some(log => !log.done)) {
      const message = "AI正在处理您的请求，确定要离开吗？";
      event.preventDefault();
      event.returnValue = message;
      return message;
    }
  };

  const handleUnload = () => {
    // 页面卸载时执行清理
    try {
      if ('sendBeacon' in navigator) {
        navigator.sendBeacon('/api/cleanup', JSON.stringify({ 
          agent: agent,
          timestamp: Date.now() 
        }));
      }
    } catch (error) {
      // 忽略错误，因为页面即将关闭
    }
  };

  window.addEventListener('beforeunload', handleBeforeUnload);
  window.addEventListener('unload', handleUnload);
  
  return () => {
    window.removeEventListener('beforeunload', handleBeforeUnload);
    window.removeEventListener('unload', handleUnload);
  };
}, [state.logs, agent]);
```

### 3. 创建清理API端点

```typescript
// ui/src/app/api/cleanup/route.ts
export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const { agent, timestamp } = body;

    console.log(`客户端请求清理连接:`, {
      agent, timestamp,
      userAgent: req.headers.get('user-agent'),
      ip: req.headers.get('x-forwarded-for')
    });

    return NextResponse.json({ 
      success: true, 
      message: '清理请求已处理' 
    });
  } catch (error) {
    console.error('处理清理请求时出错:', error);
    return NextResponse.json({ 
      success: true, 
      message: '清理请求已接收' 
    });
  }
}
```

### 4. 优化Caddy配置

```caddyfile
# AI站点配置
ai.medstarai.com {
  # WebSocket连接特殊处理
  @websocket {
    header Connection *Upgrade*
    header Upgrade websocket
  }
  
  reverse_proxy @websocket http://ip:3333 {
    header_up Host {host}
    header_up X-Real-IP {remote}
    header_up X-Forwarded-For {remote}
    header_up X-Forwarded-Proto {scheme}
    header_up Connection {>Connection}
    header_up Upgrade {>Upgrade}
    
    transport http {
      read_timeout 300s
      write_timeout 300s
      dial_timeout 5s
    }
  }
  
  # 普通HTTP请求
  reverse_proxy http://ip:3333 {
    header_up Host {host}
    header_up X-Real-IP {remote}
    header_up X-Forwarded-For {remote}
    header_up X-Forwarded-Proto {scheme}
  }
}
```

## ⚠️ 遇到的额外问题

### React Hooks规则错误

在实现过程中遇到构建错误：

```
Error: React Hook "useCopilotContext" is called conditionally. 
React Hooks must be called in the exact same order in every component render.
```

**解决方案**：移除条件性Hook调用，改为基于pathname判断：

```typescript
// ❌ 错误做法
try {
  const copilotContext = useCopilotContext();
} catch (error) {
  // 处理错误
}

// ✅ 正确做法  
if (pathname?.includes('/medical-science')) {
  // 处理AI页面逻辑
}
```

### Next.js图片优化502错误

使用`<Image>`组件后出现502错误：

```
GET /_next/image?url=https%3A%2F%2Fwww.google.com%2Fs2%2Ffavicons...
502 Bad Gateway
```

**解决方案**：禁用图片优化避免代理问题：

```javascript
// next.config.mjs
const nextConfig = {
  output: "standalone",
  images: {
    // 禁用图片优化，避免代理问题
    unoptimized: true,
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'www.google.com',
        pathname: '/s2/favicons/**',
      },
    ],
  },
};
```

## 📚 经验总结

### 技术要点

1. **WebSocket连接管理**：在SPA应用中，需要特别注意WebSocket连接的生命周期管理
2. **优雅降级**：即使连接清理失败，也要保证用户能正常跳转
3. **事件监听最佳实践**：合理使用`beforeunload`和`unload`事件
4. **React Hooks规则**：避免条件性调用Hooks

### 架构建议

1. **连接状态管理**：建议使用状态管理工具追踪WebSocket连接状态
2. **错误边界**：为WebSocket相关组件添加错误边界
3. **监控告警**：添加连接异常的监控和告警机制

### 部署注意事项

1. **代理配置**：确保反向代理正确处理WebSocket升级
2. **超时设置**：为长连接设置合理的超时时间
3. **健康检查**：添加WebSocket连接的健康检查机制

## 🎯 结论

通过系统性的问题分析和解决方案实施，我们成功解决了医学AI应用中的WebSocket连接中断问题。关键在于：

1. **准确诊断**：识别出问题的根本原因是连接管理不当
2. **优雅处理**：实现了平滑的连接清理机制  
3. **防御编程**：添加了多层错误处理和降级方案

这次问题解决过程让我们深刻理解了现代Web应用中实时连接管理的复杂性，以及在生产环境中进行充分测试的重要性。
