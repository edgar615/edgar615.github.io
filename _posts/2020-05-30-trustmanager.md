---
layout: post
title: TrustManager
date: 2019-03-14
categories:
    - java
comments: true
permalink: java-trustmanager.html
---

最近实现GRPC的TLS认证，遇到一些坑，记录一些，这篇文章不完善，后面有时间补充

```
CertificateFactory cf = CertificateFactory.getInstance("X.509");
                    Certificate cfet = cf.generateCertificate(new FileInputStream("C:\\Users\\Administrator\\Desktop\\nginx.crt"));
String keyStoreType = KeyStore.getDefaultType();
KeyStore keyStore = KeyStore.getInstance(keyStoreType);
keyStore.load(null, null);
keyStore.setCertificateEntry("ca", cfet);
 
 
//初始化TrustManagerFactory
String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
                    TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
                    tmf.init(keyStore);

```
