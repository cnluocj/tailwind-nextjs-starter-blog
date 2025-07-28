---
title: 'Dify 和 Coze 同时启动时 Redis 容器名称冲突解决方案'
date: '2025-07-28'
tags: ['dify', 'coze', 'redis', 'docker', '容器冲突']
---

## 背景

在本地开发环境中，当需要同时运行 Dify 和 Coze 两个 AI 应用时，遇到了一个常见的 Docker 容器冲突问题。

## 问题表现

两个应用都使用 Redis 作为缓存和消息队列，当先启动 Dify，再启动 Coze 时，会出现：

1. Coze 的 Redis 容器会直接替换掉 Dify 的 Redis 容器
2. Dify 应用无法再访问 Redis，导致功能异常
3. 没有明显的错误提示，但前面启动的应用会静默失效

## 问题分析

- Dify 和 Coze 的 `docker-compose.yml` 文件中，Redis 服务使用相同的默认容器名称
- Docker Compose 在遇到同名容器时，会停止并替换旧容器
- 后启动的应用会"接管" Redis 容器，导致先启动的应用失去 Redis 连接
- 这种覆盖行为是静默的，不会产生明显的错误提示

## 解决方案

使用 Docker Compose 的项目名称隔离功能，通过 `-p` 参数指定不同的项目名称：

### 1. 启动 Dify

```bash
docker-compose -p dify up -d
```

### 2. 启动 Coze

```bash
docker-compose -p coze up -d
```

### 3. 验证隔离效果

查看运行中的容器：

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
```

此时可以看到容器名称变为：

- `dify_redis_1` 或 `dify-redis-1`
- `coze_redis_1` 或 `coze-redis-1`

## 其他管理命令

使用项目名称后，相关的管理命令也需要指定项目名：

```bash
# 停止 Dify
docker-compose -p dify down

# 停止 Coze
docker-compose -p coze down

# 查看 Dify 服务状态
docker-compose -p dify ps

# 查看 Coze 日志
docker-compose -p coze logs -f
```

## 总结

通过使用 `docker-compose -p <project-name>` 参数，可以有效隔离不同项目的容器资源，避免：

1. 容器名称冲突
2. 端口映射冲突
3. 网络命名空间冲突
4. 卷名称冲突

这是在本地开发环境中同时运行多个使用相同基础服务（如 Redis、PostgreSQL 等）的应用的最佳实践。

建议在日常开发中养成为每个项目指定唯一项目名称的习惯，避免潜在的资源冲突问题。
