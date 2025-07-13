---
title: '纯IPv6机器NAT64 DNS配置'
date: 2025-07-08
tags: ['IPv6', 'NAT64', 'DNS']
---

## 纯IPv6机器NAT64 DNS配置

如纯IPv6机器，请设置NAT64，可执行下方命令：

```bash
sed -i "1i\nameserver 2a00:1098:2b::1\nnameserver 2a00:1098:2c::1\nnameserver 2a01:4f8:c2c:123f::1\nnameserver 2a01:4f9:c010:3f02::1" /etc/resolv.conf
```

如果添加下方命令还是不可用，请尝试更换其他NAT64服务器。
