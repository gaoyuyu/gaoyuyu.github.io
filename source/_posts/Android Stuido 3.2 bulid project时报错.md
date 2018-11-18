---
title: Android Stuido 3.2 bulid project 时报错Manifest merger failed with multiple errors, see logs
date: 2018-03-03 22:20:04
tags:
---

Android Stuido 3.2 bulid project 时报错Manifest merger failed with multiple errors, see logs
<!--more-->
![这里写图片描述](https://img-blog.csdn.net/20180303220216123?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
以下是过程
1.搜的CSDN的一篇文章[Manifest merger failed with multiple errors, see logs问题处理](http://blog.csdn.net/Picasso_L/article/details/53085299)

> 命令行 输入 gradlew processDebugManifest --stacktrace
> 提示：Could not reserve enough space for 1572864KB object heap

接着去修改内存值等等...没效果，放弃，不是问题的原因
2.然后是这篇文章[Androidstudio常见错误"Manifest merger failed with multiple errors, see logs"](http://blog.csdn.net/lby159951/article/details/51082761)

> 注册文件 在application节点添加
> tools:replace=“android:icon, android:label, android:theme”

没效果，放弃，也不是问题的原因

3.之后也是各种搜资料，找到了
google官方链接：https://developer.android.com/studio/build/manifest-merge.html

![这里写图片描述](https://img-blog.csdn.net/20180303221219458?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这里的图片很明了了，第一时间查看AndroidManifest.xml文件，果然有报错
![这里写图片描述](https://img-blog.csdn.net/20180303221412356?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

大意了，由于项目是2个app project和2个library库的依赖关系，JPush的配置在第一个app project的bulid.gradle给了，第二个app project的bulid.gradle还没给，就导致了合并注册文件出错，添加完成后问题解决，无报错。



**总结 Manifest merger failed with multiple errors, see logs 错误处理**
**查看AndroidManifest.xml的树形结构看有无报错信息，有报错对照着处理即可**
