---
title: '在 mac 中使用 host.docker.internal 访问容器'
date: '2025-03-15'
tags: ['docker']
---

在 mac 中默认使用 host.docker.internal 就能容器间就能相互访问

而 Linux 中并没有这个默认设置，可以使用 extra_hosts 进行配置来实现，两边服务都需要添加，这样可以做到相互访问

```yaml
version: '3'
services:
  api:    # 或者是 backend 等服务名称
    # ... 其他配置 ...
    extra_hosts:
      - "host.docker.internal:host-gateway"
  
  web:    # 如果有前端服务的话
    # ... 其他配置 ...
    extra_hosts:
      - "host.docker.internal:host-gateway"
```