---
title: IDE配置GCI
date: 2025-04-17 09:24:13
tags:
  - GO
  - Goland
  - VSCode
  - GCI
  - 配置
  - 代码格式化
categories: IDE配置
---

# 在IDE中配置GCI
## 安装GCI
```bash
go install github.com/daixiang0/gci@latest
```

## Goland

### 使用GCI格式化导入包
1. 打开`Settings` -> `Tools` -> `File Watchers` -> 点击`+`，添加一个`File Watcher`
   
   {% img [img] /img/ide-config-gci/setting-file-watcher.png '"text"' %}
   
2. `Name`中输入`gci` -> `File type`选择`Go files` -> `Scope`中选择`Current File` -> `Program`中输入`gci` -> `Arguments`中输入`write -s standard -s blank  -s "prefix(github.com),prefix(go.uber.org),prefix(golang.org),prefix(gorm.io)"  -s blank -s "prefix(替换自己本地项目名)" -s blank -s dot --skip-generated $FileName$` -> 点击`OK`
   
   {% img [img] /img/ide-config-gci/goland-add-gci-file-watcher.png '"text"' %}

3. 在`Setting`中点击`Apply` -> `OK`
   
   {% img [img] /img/ide-config-gci/goland-setting-apply.png '"text"' %}

### 使用golangci-lint(使用GCI格式化导入包)

1. 创建一个`.golangci.yml`文件
   ```yaml
    run:
    verbose: true
    linters:
    enable:
        - gci
    linters-settings:
    gci:
        sections:
        - standard
        - blank
        - prefix(github.com),prefix(go.uber.org),prefix(golang.org),prefix(gorm.io)
        - blank
        - default
        - prefix(自己的包名)
        - blank
        - dot
        skip-generated: true
   ```
1. 添加一个`golangci-lint`的`File Watcher`
   
   {% img [img] /img/ide-config-gci/goland-add-golangci-file-watcher.png '"text"' %}

2. 修改配置
   参数(Arguments)调整 `run --fix --allow-parallel-runners --disable-all --enable=gci $FileName$`

   {% img [img] /img/ide-config-gci/goland-set-golangci-file-watcher-args.png '"text"' %}

### 参数说明
- `-s standard`：标准库分组排序
- `-s blank`：添加一个空行
- `-s "prefix(github.com),prefix(go.uber.org),prefix(golang.org),prefix(gorm.io)"` ：第三方库分组排序, 如果有其他不同前缀的第三方库, 可以添加到这里
- `-s default`：默认分组排序
- `-s "prefix(替换自己本地项目名)"`：本地项目分组排序
- `--skip-generated`：跳过生成的文件
- `$FileName$`：当前文件路径，也可以选择项目目录
  
   {% img [img] /img/ide-config-gci/goland-replace-macros.png '"text"' %}