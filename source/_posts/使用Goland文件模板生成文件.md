---
title: 使用Goland文件模板生成文件
categories: Goland
tags:
  - Golang
  - Goland
  - Template
date: 2021-05-13 19:29:52
---



打开 Goland 设置找到 `File and Code Templates` 新建模板

velocity 语法：http://velocity.apache.org/engine/devel/user-guide.html#string-concatenation

```template
package ${GO_PACKAGE_NAME}

#set($HUMP_NAME = ${StringUtils.removeAndHump(${NAME}, "_")})
type ${HUMP_NAME} struct {
}

func (${HUMP_NAME}) TableName() string {
    return "${NAME}"
}
```

## Ref

https://www.coder.work/article/6393553

https://www.codota.com/code/java/methods/org.apache.velocity.util.StringUtils/removeAndHump

