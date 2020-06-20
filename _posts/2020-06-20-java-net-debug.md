---
layout: post
title: Java开启网络debug
date: 2020-06-20
categories:
    - java
comments: true
permalink: javax-net.debug.html
---

-Djavax.net.debug=ssl
或者
-Djavax.net.debug=all
 
all            turn on all debugging
ssl            turn on ssl debugging


The following can be used with ssl:

```
record       enable per-record tracing
handshake    print each handshake message
keygen       print key generation data
session      print session activity
defaultctx   print default SSL initialization
sslctx       print SSLContext tracing
sessioncache print session cache tracing
keymanager   print key manager tracing
trustmanager print trust manager tracing
pluggability print pluggability tracing

handshake debugging can be widened with:
data         hex dump of each handshake message
verbose      verbose handshake message printing

record debugging can be widened with:
plaintext    hex dump of record plaintext
packet       print raw SSL/TLS packets
```
