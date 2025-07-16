---
title: 'Oracle Cloud 实例开通并打通网络端口全流程'
date: '2025-07-10'
tags: ['Oracle Cloud', 'OCI', 'VCN', '网络配置', 'iptables', '云服务器']
draft: false
summary: '详细记录在 Oracle Cloud（OCI）上从实例创建到成功开放网络端口的完整步骤，包括 VCN 配置、Security List 设置和实例内部防火墙配置，便于今后复用或写成文档。'
---

## ✅ Oracle Cloud 实例开通并打通网络端口全流程

### 🟢 第一步：创建实例

1. 登录 Oracle Cloud 控制台
2. 进入「Compute → Instances」点击 Create instance
3. 选择：
   - 实例名称
   - 镜像（通常选 Ubuntu 或 Oracle Linux）
   - 形状（Shape）：比如 VM.Standard.A1.Flex
   - SSH 公钥
   - 创建时选择或新建 VCN（虚拟云网络）

---

### 🟢 第二步：配置 VCN 及网络组件

#### 🔹1. 配置 Subnet

- 保证 Subnet 类型为 Public Subnet
- CIDR 一般为 10.0.0.0/16

#### 🔹2. 创建 Internet Gateway

- 在 VCN 中 → Gateways → 创建 Internet Gateway
- 命名为 Internet Gateway for VCN-xxxx（你已创建成功）

#### 🔹3. 配置 Route Table

- 默认 Route Table 中新增路由规则：

| Destination CIDR | Target           |
| ---------------- | ---------------- |
| 0.0.0.0/0        | Internet Gateway |

---

### 🟢 第三步：配置 Security List（网络安全组）

在 VCN 里找到对应的 Security List，添加 Ingress Rules：

#### 基础端口开放：

| 协议 | 来源      | 端口范围   | 用途                         |
| ---- | --------- | ---------- | ---------------------------- |
| TCP  | 0.0.0.0/0 | 22         | SSH 登录                     |
| ICMP | 0.0.0.0/0 | 类型 8     | ping 请求（Echo）            |
| TCP  | 0.0.0.0/0 | 自定义端口 | 例如：VPN、Web 等服务端口    |
| UDP  | 0.0.0.0/0 | 自定义端口 | 例如：WireGuard、sing-box 等 |

你这次添加了：

- TCP: All → 全端口
- UDP: All → 全端口
- ICMP: type 8 → 支持 ping
- ✅ 已完全放开访问

---

### 🟢 第四步：实例内部防火墙配置（iptables）

OCI 默认镜像可能仍启用 iptables 拦截部分端口，即使云端放行了也无法访问。

#### 检查：

```bash
sudo iptables -L -n
```

发现存在：

```
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

#### 临时放行服务端口：

```bash
sudo iptables -I INPUT -p tcp --dport 17871 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 17871 -j ACCEPT
```

#### （可选）永久关闭 iptables：

```bash
sudo iptables -F
# Ubuntu:
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## 总结

通过以上四个步骤，你已经成功完成了 Oracle Cloud 实例的网络配置：

1. **实例创建** - 选择合适的配置和 VCN
2. **VCN 网络组件** - 配置 Subnet、Internet Gateway 和 Route Table
3. **Security List** - 开放必要的端口和协议
4. **实例防火墙** - 处理 iptables 规则

### 关键要点

- **双重防火墙**：Oracle Cloud 有云端 Security List 和实例内 iptables 两层防护
- **完整路由**：确保 Internet Gateway 和 Route Table 配置正确
- **端口开放**：根据实际需要开放特定端口，或临时全开进行测试
- **持久化配置**：重要的 iptables 规则需要持久化保存

这套流程适用于大多数 Oracle Cloud 实例的网络配置需求，可以作为标准操作流程参考。
