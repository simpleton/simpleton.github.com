---
author: sunsj1231
comments: true
date: 2011-07-17 08:54:03
layout: post
slug: meego%e4%b8%80%e4%ba%9b%e5%bc%b1%e5%bc%b1%e7%9a%84qml%e7%9f%a5%e8%af%86
title: '[MeeGo]一些弱弱的QML知识'
wordpress_id: 44039
categories:
- coding
---

初学QML，在这里记一些特殊的知识点。要持续更新！！！



	
  1. Any snippet of QML code can become a component, just by placing it in the file "<Name>.qml" where <Name> is the new element name, and begins with an **uppercase** letter.  <quote:[http://doc.qt.nokia.com/4.7/qdeclarativedocuments.html](http://doc.qt.nokia.com/4.7/qdeclarativedocuments.html)>

	
  2. **Components** are reusable, encapsulated QML elements with well-defined interfaces. <quote: [http://doc.qt.nokia.com/4.7-snapshot/qml-component.html#details](http://doc.qt.nokia.com/4.7-snapshot/qml-component.html#details)>


TODO:

	
  1. <del>搞清component的意义<[http://mxr.meego.com/meego.gitorious.org/source/meego-ux-daemon/statusindicatormenu.qml#36](http://mxr.meego.com/meego.gitorious.org/source/meego-ux-daemon/statusindicatormenu.qml#36) > </del>

	
  2. DBUS的基本概念，还有在qt里的使用方法。<del>[http://wiki.meego.com/D-Bus/Overview](http://wiki.meego.com/D-Bus/Overview)     [http://kb.cnblogs.com/page/92194/](http://kb.cnblogs.com/page/92194/)   [http://blog.csdn.net/feiyinzilgd/article/details/6099087](http://blog.csdn.net/feiyinzilgd/article/details/6099087)  [http://hi.baidu.com/cyclone/blog/item/1bf8033bbb7498e514cecb50.html](http://hi.baidu.com/cyclone/blog/item/1bf8033bbb7498e514cecb50.html)</del>

	
  3. <del>qtdbus中的adaptor proxy interface.</del>

	
  4. qml中定义外部接口变量，如同默认的一些color, width之类的。

	
  5. 如何写一个cpp的model给qml用。



