---
title: "Docker DNS 解析失败与 Let's Encrypt 证书申请问题解决记录"
date: 2025-06-05
tags: ['docker', 'dns', 'letsencrypt', '故障排查']
---

## 背景

在一台新配置的 VPS 上，使用 Docker 部署服务时遇到了两个问题：

1. Docker 容器内 DNS 解析失败，导致容器内无法访问域名。
2. 使用 `kejilion.sh`（实为基于 Docker 的 nginx 和 acme.sh）申请证书失败，提示域名解析失败。

## 问题表现

### Docker DNS 错误表现

容器内执行如下命令：

```bash
docker run --rm busybox nslookup google.com
```

报错为：

```
nslookup: can't resolve 'google.com'
```

### Let's Encrypt 报错内容

申请证书时报错：

```
An unexpected error occurred:
requests.exceptions.ConnectionError: HTTPSConnectionPool(host='acme-v02.api.letsencrypt.org', port=443): Max retries exceeded with url: /directory (Caused by NameResolutionError("<urllib3.connection.HTTPSConnection object at 0x7f...>: Failed to resolve 'acme-v02.api.letsencrypt.org' ([Errno -3] Try again)"))
```

## 问题排查

- 宿主机网络正常，能 ping 通 google.com。
- 但 Docker 容器中的 DNS 无法解析域名。
- 怀疑是 `/etc/docker/daemon.json` 配置问题导致 DNS 配置无效。

## 解决方案

### 1. 修复 `/etc/docker/daemon.json`

查看配置发现语法错误，例如：

```json
{
  "dns": ["8.8.8.8"] "log-level": "error"
}
```

应该改为：

```json
{
  "dns": ["8.8.8.8"],
  "log-level": "error"
}
```

修复后重启 Docker：

```bash
sudo systemctl restart docker
```

### 2. 验证 DNS 是否恢复正常

```bash
docker run --rm busybox nslookup google.com
```

如果能正确解析出 IP 地址，说明 DNS 设置成功。

### 3. 再次申请证书

运行 `kejilion.sh`，即可顺利申请 Let's Encrypt 证书。

## 总结

本次问题主要由 Docker 配置文件中的 JSON 语法错误导致 DNS 解析失败，连带影响了证书申请过程。排查重点应放在：

- 容器内网络与 DNS 可用性验证
- `/etc/docker/daemon.json` 配置的合法性
- 配置变更后的 Docker 服务重启

建议每次修改 Docker 配置后及时使用容器测试网络连通性，防止隐性故障。
