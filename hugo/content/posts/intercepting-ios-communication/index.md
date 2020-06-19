---
title: "Intercepting and Decrypting iOS communications"
summary: "How to intercept and decrypt the HTTPS communications for *some* of your iOS applications"
date: "2020-06-19T19:06:00+10:00"
lastmod: "2020-06-19T19:06:00+10:00"
author: Gary Jackson
draft: true
categories:
- "Development"
tags:
- "iOS"
- "Encryption"
- "Decryption"
---

## Overview

Ever wondered how you can view the network communications from your iPhone - me too, let's have a look.

## Goals

There are two things I'd like to do

1. Intercept the network communications
2. Decrypt the communications

### Intercepting Network Communications
I somehow need to inject an intermediary between my phone and the internet that can accept the network communications from my iPhone, and forward the traffic to the intended recipients.

That intermediary is called a proxy - cool, 1 down :heavy_check_mark:

### Decrypting Communications
This one is a bit more complicated, so maybe I need to start off with a quick high level view of how HTTPS works.

```Powershell
docker run --rm -it -p 8080:8080 mitmproxy/mitmproxy
```
Apps that worked
Safari
Weather
Photos
Gmail
LinkedIn

Apps that didn't work
App Store
FaceBook
Instagram

## References
- [mitmproxy](https://mitmproxy.org/)