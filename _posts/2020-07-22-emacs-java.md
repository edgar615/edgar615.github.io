---
layout: post
title: 使用Spacemacs搭建Java开发环境
date: 2020-07-22
categories:
    - linux
comments: true
permalink: emacs-java.html
---

因为特殊原因，需要在shell下写点Java项目，选择了Spacemacs搭建环境

**emacs一定要装最新版本，我是26.3，在24，25,26.0版本上都出现了错误**

安装emacs和Spacemacs很简单，不写了，首次运行emacs会安装插件并创建Spacemacs的配置文件`.spacemacs`

# 1. 添加国内源

默认源比较慢，可以使用清华大学的源

在`dotspacemacs/user-init`中添加

```
(setq configuration-layer-elpa-archives
    '(("melpa-cn" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/melpa/")
      ("org-cn"   . "http://mirrors.tuna.tsinghua.edu.cn/elpa/org/")
      ("gnu-cn"   . "http://mirrors.tuna.tsinghua.edu.cn/elpa/gnu/")))
```

# 2. 显示行号
在`defun dotspacemacs/init`中找到 `dotspacemacs-line-numbers nil`，将`nil`改为t

# 3. 显示80字符的column

官网 https://github.com/alpaker/Fill-Column-Indicator

在`dotspacemacs-additional-packages`中添加 `(require 'fill-column-indicator)`

对所有文件开启：在`dotspacemacs/user-init`中添加`(add-hook 'after-change-major-mode-hook 'fci-mode)`

在`defun dotspacemacs/init`修改默认参数，将分隔符改为120

```
fci-mode t
   fci-rule-column 120
```

# 4. 增加Java的layer
在`dotspacemacs-configuration-layers`中添加 `(java :variables java-backend 'meghanada)`

# 5. 关闭自动更新
每次启动都要更新，浪费时间

`dotspacemacs-check-for-update nil`改为 `dotspacemacs-check-for-update t`

没起作用

# 6. 语法检查
在`dotspacemacs-configuration-layers`中开启`syntax-checking`

(setq-default dotspacemacs-configuration-layers
  '((syntax-checking :variables syntax-checking-enable-tooltips t)))

# 7. 运行Java程序

两种方式

- 通过maven ` mvn exec:java -Dexec.mainClass=com.github.edgar615.HelloWorldServer`
- 通过alt+x: meghanada-exec-main 运行main方法(两次SPC呼出)

# 快捷键

- SPC 	space
- C 	ctrl
- M 	alt
- S 	shift
- DEL 	backspace
