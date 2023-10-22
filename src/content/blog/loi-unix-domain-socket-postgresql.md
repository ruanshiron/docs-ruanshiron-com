---
author: ruanshiron
pubDatetime: 2022-03-17T02:37:53Z
title: Lỗi unix domain socket ở PostgreSQL
postSlug: loi-unix-domain-socket-postgresql
tags:
  - code
  - error
  - postgresql
description: Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
---

Việc đầu tiên nên làm là kiểm tra log chứ không phải search google!

```shell
tail -f /usr/local/var/log/postgres.log
```
