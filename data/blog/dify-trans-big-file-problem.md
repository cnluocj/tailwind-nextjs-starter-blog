---
title: 'dify http 传输大文件 Server disconnected without sending a response.'
date: '2025-04-24'
tags: ['dify']
---

https://github.com/langgenius/dify/issues/12098
https://github.com/langgenius/dify/issues/12216
After I removed the SSRF configuration, it worked.

```
  #SSRF_PROXY_HTTP_URL: ${SSRF_PROXY_HTTP_URL:-http://ssrf_proxy:3128}
  #SSRF_PROXY_HTTPS_URL: ${SSRF_PROXY_HTTPS_URL:-http://ssrf_proxy:3128}
```

![remove-ssrf](/static/images/remove-ssrf.png)
  
注释掉 docker-compose.yaml 中的 ssrf proxy 配置

很神奇的是，如果dify是https就没问题