---
title: Android Studio 3.2 Canary4 在编辑 build.gradle 文件时出现卡顿严重
date: 2018-03-11 19:24:05
tags:
---

Android Studio 3.2 Canary4 在编辑 build.gradle 文件时出现卡顿严重
问过小伙伴，说设置AS的http代理指向本机127.0.0.1，没有效果还是一样
之后查到这篇文章[Macbook Pro Android Studio 编辑 build.gradle 卡顿严重](http://blog.csdn.net/longyc2010/article/details/53180491)
恍然大悟，原来编辑build.gradle的时候AS会将我们的行为上报到google的网站

http://search.maven.org/solrsearch/select?q=g:%22com.google.android.support%22+AND+a:%22wearable%22&core=gav&rows=1&wt=json

http://search.maven.org/solrsearch/select?q=g:%22com.google.android.gms%22+AND+a:%22play-services%22&core=gav&rows=1&wt

木有错，就是上面这俩货
之后，在host文件设置search.maven.org指向127.0.0.1就好多了

> 127.0.0.1 search.maven.org
