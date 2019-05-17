---
title: Git基本命令详解
date: 2018-06-16 16:44:12
toc: true
comments: true
tags:
  - git使用
  - 技术
categories:
  - 项目工具
---


最近刚开始学习git，总结一下git的各个命令，方便以后查阅。  
学习环境：windows10   
 **参考链接：**
>* Pro Git（中文版）[http://git.oschina.net/progit/](http://git.oschina.net/progit/)
> * 沉浸式学 Git[http://igit.linuxtoy.org/contents.html](http://igit.linuxtoy.org/contents.html)

## **1. git的安装及初始配置**  
### **1.1 git 安装**   
&emsp; &emsp;windows下安装git很方便，github上提供了安装包，链接： [http://msysgit.github.com/](http://msysgit.github.com/)   

### **1.2 git 初始化配置**
&emsp; &emsp; 1.2.1&emsp;配置用户名和户邮箱：
```
    git config --global user.name "deng wen" 
	git config --global user.email 156XXXXXXX@163.com 
```
&emsp; &emsp; 1.2.2&emsp;查看初始配置：
```
    git config --list 
```
## **2.git的基础命令**
### **2.1 新建仓库**
&emsp; Git 新建项目仓库的方法有两种。分别为：  
&emsp; &emsp;  2.1.1&emsp; 第一种：在现存的目录下，用如下命令得到一个.git仓库目录，资源对应添加到其中：
```
    git init
```
&emsp; &emsp;  2.1.2&emsp; 第二种：从已有的 Git 仓库克隆出一个新的镜像仓库来。[URL]       如：http://uestclab307.kmdns.net:808/dengwen/SAIC_SecMonitor.git，mygitname可省略：
```
    git clone [URL] mygitname
```
### **2.2 文件基本处理**
&emsp; &emsp;  2.2.1&emsp;检查当前文件状态：
```
    git status
``` 
 &emsp; &emsp;  2.2.2&emsp; 将新文件或更新文件加入跟踪：
```
    git add  filename
    git add  —A //将所有新文件一次加入跟踪
    git checkout -- filename // 对所做的更改进行忽略
    git reset HEAD filename //撤销加入跟踪的文件 
```  
 &emsp; &emsp;  2.2.3&emsp; 文件提交：
```
    git commit  —m  "your comment" //-m表示注释
    git commit --amend //撤销刚做的提交
    git commit  —a  //所有跟踪文件一次提交
```  
 &emsp; &emsp;  2.2.4&emsp; 删除文件：
```
    git rm --cached filename //跟踪目录删除、本地不删除
    git rm -f filename //跟踪目录、本地目录皆删除：
```  
 &emsp; &emsp;  2.2.5&emsp; 在仓库中移动文件：
```
    git mv file_from file_to
```  
 &emsp; &emsp;  2.2.6&emsp; 查看提交历史，- -pretty按固定格式显示,--graph 选项用 ASCII 字符串形象地展示了每个提交所在的分支及其分化衍合情况：
```
    git log
    git log --pretty=format:"%h - %an, %ar : %s"
    git log --pretty=format:"%h %s" --graph
```  
>**选项 说明**
    %H 提交对象（commit）的完整哈希字串
    %h 提交对象的简短哈希字串
    %T 树对象（tree）的完整哈希字串
    %t 树对象的简短哈希字串
    %P 父对象（parent）的完整哈希字串
    %p 父对象的简短哈希字串
    %an 作者（author）的名字
    %ae 作者的电子邮件地址
    %ad 作者修订日期（可以用 -date= 选项定制格式）
    %ar 作者修订日期，按多久以前的方式显示
    %cn 提交者(committer)的名字
    %ce 提交者的电子邮件地址
    %cd 提交日期
    %cr 提交日期，按多久以前的方式显示
    %s 提交说明
    
### **2.3 远程仓库的使用**
 &emsp; &emsp;  2.3.1&emsp; 查看当前的远程库,-v 选项显示对应的克隆地址：
```
    git remote //查看当前仓库对应的远程库,一般为origin
    git remote -v //查看当前仓库对应的远程库及相应地址
``` 
 &emsp; &emsp;  2.3.2&emsp;远程仓库处理：
```
    git remote add yourname [url] //yourname是你的本地仓库名，相当于赋值yourname为URL
    git fetch yourname  //从远程仓库抓取数据到本地，到如果要查看需要合并到当前分支
    git remote show [remote-name] //查看远程仓库信息，如显示了有哪些远端分支还没有同步到本地等
    git remote rename old-name new-name //重命名
    git remote rm paul //删除
``` 
### **2.4 标签**
 &emsp; &emsp;  2.4.1&emsp; 新建标签：
```
    git tag //查看已有标签
    git tag -a yourtagname -m 'your comment' //打标签
    git show yourtagname //查看版本信息 
    git tag -a yourtagname hist //后期加标签,hist表校验和
```
 &emsp; &emsp;  2.4.2&emsp; 标签远程共享：
```
    git push origin yourtagname //推送标签
    git push origin --tags //推送所有标签
```
### **2.5 Git 命令别名**
 &emsp; &emsp;  2.5.1&emsp; 简写git命令：
```
     git config --global alias.shortname gitcommandname 
     eg: git config --global alias.unstage 'reset HEAD'
         git unstage filename //撤销加入跟踪的文件 
```

## **3.git分支处理**
### **3.1 Git 查看分支**
 &emsp; &emsp;  3.1.1&emsp; 查看分支：
```
     git branch 
     git branch -a //查看所有分支，包括远程分支
     git branch -v //查看分支最后一个提交对象的信息
     git branch --merged/--no--merge //查看已经（或尚未）合并的分支
```
### **3.2 Git 分支切换、合并和删除**
 &emsp; &emsp;  3.2.1&emsp; 切换分支：
```
     git branch branchname  //在当前分支下创建分支
     git checkout branchname //切换到已有的分支
     git checkout -b 'branchname' //创建分支并切换
```
 &emsp; &emsp;  3.2.2&emsp; 合并分支：
```
     git merge branchname //将分支合并到当前分支
```

 &emsp; &emsp;  3.2.3&emsp; 删除分支：
```
     git branch -d branchname  //删除分支
```
### **3.3 Git 远程分支处理**
 &emsp; &emsp;  3.3.1&emsp;跟踪远程分支：
```
     git checkout -b [分支名] [远程仓库名]/[分支名]  //跟踪分支是一种和某个远程分支有直接联系的本地分支
     git pull  //新建跟踪分支后用该命令直接将远程分支合并进来
     git push  //将本地跟踪分支推送到远程分支
     git push --set-upstream [远程仓库名] [分支名]  //将当前的分支设置为跟踪某个远程分支

```
 &emsp; &emsp;  3.3.2&emsp;抓取和合并远程分支：
```
     git fetch origin  //同步远程origin/master数据到本地，指针移到它最新的位置上。
     git merge origin/remotename  //将远程分支的内容合并到当前分支，用于远程分支已同步而又不能直接访问时。
```

 &emsp; &emsp;  3.3.3&emsp;推送分支和删除远程分支：
```
     git push origin name1:name2  //把本地分支name1推送到远程分支name2中，如果远程仓库没有这个分支，会生成这样一个新的分支。用这种方式可以远程创建分支。  
     git push origin name //将本地分支推到远程同名分支。
     git push origin :remotename //把空白远程远程分支，即删除远程分支。

```

<br>
<br>
  
***这些就是基本的git命令，更多待进一步学习***