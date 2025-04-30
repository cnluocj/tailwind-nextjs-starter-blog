---
title: "Firecrawl can't accept connection"
date: 2025-04-30
description: "Firecrawl can't accept connection"
tags: ['firecrawl']
---

Previously, the services were experiencing "Can't accept connection" errors in docker-compose logs due to unmanaged resource allocation. These new environment variables help prevent resource exhaustion by limiting RAM and CPU usage to 95% by default.

之前，服务在 docker-compose 日志中出现了 "Can't accept connection" 错误，原因是未管理的资源分配。这些新的环境变量有助于防止资源耗尽，默认限制 RAM 和 CPU 使用率为 95%。

https://github.com/devflowinc/firecrawl-simple/pull/24
https://github.com/devflowinc/firecrawl-simple/pull/24/commits/da1ef45cb8d41df0d3530d12ba7e5290b60e9010
