---
title: >-
  Java.lang.NoClassdefFoundError: Could Not Initialize Class
  Sun.AWT.X11.XtoolKit异常解决
date: 2020-05-30 23:16:11
tags: Java
categories: 
- 踩坑
---

该问题的解决方法：
在Tomcat/bin/catalina.sh 中增加-Djava.awt.headless=true

```shell
如下：
JAVA_OPTS="$JAVA_OPTS -Djava.awt.headless=true"
```

![img](http://andiycc.com/wp-content/uploads/2019/05/B74248D8-2721-4eab-B9EB-5DFFB5F40332.png)

原因：
对于一个Java服务器来说经常要处理一些图形元素，例如地图的创建或者图形和图表等。这些API基本上总是需要运行一个X-server以便能使用AWT（Abstract Window Toolkit，抽象窗口工具集）