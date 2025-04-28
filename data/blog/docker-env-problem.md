---
title: Docker .env 不更新问题
date: 2025-04-23
tags: [docker]
---

## docker .env 不更新问题

缓存或镜像未更新：

即使执行了 docker compose down，Docker Compose 可能会复用之前构建的镜像（如果有 build 配置）。镜像构建时使用的环境变量可能已经缓存，.env 文件的更新不会影响已构建的镜像。
如果你的环境变量是在构建镜像时使用的（例如通过 ARG 或 ENV 在 Dockerfile 中定义），仅执行 down 和 up 不会重新构建镜像。