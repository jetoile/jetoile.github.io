---
layout: post
title: "Utilisation du remote debugger"
date: 2010-02-12 22:59:01 +0100
comments: true
sharing: true
footer: true
categories: java
---
<img border="0" src="http://4.bp.blogspot.com/_XLL8sJPQ97g/S3VtYOIAnBI/AAAAAAAAAIg/vs6qhSaG6Gc/s320/blog_fabrice.png" alt="left"/>
Juste parce que j'ai la flemme de toujours rechercher le lien et parce que je trouve que cet article explique et résume très bien l'utilisation du remote debug, je me permets de mettre un lien vers l'excellent blog de Fabrice Dewasmes :

http://jtruc.dewasmes.net/?p=162

```bash
java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=n MonMachin
```