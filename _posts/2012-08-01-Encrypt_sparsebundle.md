---
title: Encrypt a sparsbundle file in os X 
last_modified_at: 2012-08-01 13:34:41 +0200
---

The following command can be used to encrypt a sparsebundle in OS X.

```console
$ sudo hdiutil convert -format UDSB \
  -encryption AES-128 \
  -o <new.sparsebundle> \
  <old.sparsebundle>
```
