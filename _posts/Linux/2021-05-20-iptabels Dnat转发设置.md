---
layout: post
category: Linux
title: iptables dnat转发设置
tagline: by 噜噜噜
tags: 
  - xx
published: true
---



<!--more-->

```
iptables -t nat -A PREROUTING -p tcp -m tcp --dport 30443 -j DNAT --to-destination 100.64.3.155:30443
```

