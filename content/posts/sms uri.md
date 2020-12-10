---
title: "sms uri"
date: 2020-12-10T22:39:54+08:00
summary: "sms uri实现访问网页发送短信"
featured_image: ""
draft: false
toc: true
tags: ['docker']
author: "bitmyth"
---

# sms uri 



```html
<!DOCTYPE html>
<html lang="zh_CN">
	<body>
    <script>
		window.location.href='sms:18911115533,13922224444&body=hello world,你好世界';
    </script>
</html>
```

参考

[http://tools.ietf.org/html/rfc5724](http://tools.ietf.org/html/rfc5724)

