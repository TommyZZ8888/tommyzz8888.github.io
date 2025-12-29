---
title: https
date: '2025-12-28T09:38:23+08:00'

categories:
- web
---
生成https key和pem:
openssl genrsa -out ./server.key 2048
openssl req -new -x509 -key ./server.key -out ./server.pem -days 365