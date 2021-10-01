---
title: 避免rm -rf *的悲惨命运
date: 2020-02-17 21:44:07
tags: 杂
---

# 避免rm -rf *的悲惨命运

今天又是悲惨的一天,没认真看,rm -rf *错地方了

为了再次避免悲惨命运

1. 还是不要老是用root用户了,创个日常的低权限用户,有必要时再登陆root

2. 别用rm 啦,改用trash吧



## 安装trash-cli

`apt-get install trash-cli`

`echo alias rm=trash >> ~/.zshrc`

`source ~/.zshrc` 

## trash-cli 命令

| command         | explain                |
| --------------- | ---------------------- |
| trash-put/trash | 将文件或目录放进回收站 |
| trash-empty     | 清空回收站             |
| trash-list      | 列出回收站中的文件     |
| restore-trash   | 还原回收站中的文件     |
| trash-rm        | 删除回收站中的单个文件 |



## 垃圾站所在位置

`~/.local/share/Trash/files`