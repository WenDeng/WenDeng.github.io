---
title: Git 中的SSH key的生成
date: 2019-05-16 16:44:12
toc: true
comments: true
tags:
  - git使用
categories:
  - 项目工具
---

## **Git 中的SSH key的生成**  
     
#### 1.1&emsp;安装git：  
&emsp; &emsp;windows下安装git很方便，github上提供了安装包，链接： [http://msysgit.github.com/](http://msysgit.github.com/) 

#### 1.2&emsp;查看是否经有SSH key：  
```
    cat ~/.ssh/id_rsa.pub //git bash中输入这个命令
```

#### 1.3&emsp;生成SSH key：
```
    ssh-keygen -t rsa -C "your.email@example.com" -b 4096 //git bash中输入这个命令，修改对应的邮箱
```
  
  
  ***如果已经存在SSH key,则直接复制即可，否则需要重新生成。***   
  ***生成SSH key时需要设置对应的文件存放路径和密码，为了方便，直接回车默认即可。***   
  
  #### 1.3&emsp;查看生成的SSH key，复制到git中即可：
  ```
    xclip -sel clip < ~/.ssh/id_rsa.pub //GNU/Linux (requires the xclip package)
    cat ~/.ssh/id_rsa.pub | clip //Git Bash on Windows / Windows PowerShell
    type %userprofile%\.ssh\id_rsa.pub | clip //Windows Command Line
```