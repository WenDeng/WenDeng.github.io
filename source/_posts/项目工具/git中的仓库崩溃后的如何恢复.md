---
title: git中的仓库崩溃后的如何恢复
date: 2018-06-16 18:44:12
toc: true
comments: true
tags:
  - git使用
categories:
  - 项目工具
---

# 解决git仓库崩溃问题
----------------------------------------------------------------
> 不知道是不是虚拟机的问题，最近修改代码后git仓库总崩溃，导致的结果就是很多时候自己刚修改的代码不得不放弃，最近找到一种比较好的解决方式，链接如下：https://stackoverflow.com/questions/11706215/how-to-fix-git-error-object-file-is-empty

### 1.git仓库崩溃表现
```
dengwen@ubuntu:~/project_DW/selog$ git status
error: object file .git/objects/a9/761932a220991b0490c2715f218f814d39b876 is empty
error: object file .git/objects/a9/761932a220991b0490c2715f218f814d39b876 is empty
fatal: loose object a9761932a220991b0490c2715f218f814d39b876 (stored in .git/objects/a9/761932a220991b0490c2715f218f814d39b876) is corrupt
```
### 2.常规解决方案
git仓库崩溃后，常规的解决方案是在其他目录git clone之前版本的项目，然后将
当前版本的项目拷贝过去进行覆盖，再进行提交，但是这样做的结果就是可能会丢失部分git commit信息，除此之外基本没什么问题。


### 3.推荐方法
这种方法的好处在于可以恢复git log信息，同时也不用重新clone项目、切换分支、替换等操作，相对来说，git管理的完整度和效率会更高，具体步骤如下：
* （1）删除全部空文件: **注意在.git目录下进行**
```
dengwen@ubuntu:~/project_DW/selog/.git$ find . -type f -empty -delete -print
./objects/0d/e32d3b8d0399414c0c8fc47a56069e9821615a
./objects/14/540f9dda3c30044e2dbe4629d22c715145f212
./objects/19/b98c74bc6c2e372887af410301a0a80495725c
./objects/55/14f9022e0e39a29d0e25cdf15cecac1f2f479c
./objects/84/0103bdd9538473baab19520eda11b88b40c953
./FETCH_HEAD
```
* （2）获取最后两条reflog：注意自己要恢复的分支，此处为develop
```
dengwen@ubuntu:~/project_DW/selog$ tail -n 2 .git/logs/refs/heads/develop
41867ca4ab8d60979e804ee7f4640a2e9231d96b f815821a9c4e4833be898dace675916f3cad0124 dengwen <15680482464@163.com> 1539335482 +0800        commit: add manage
```
* （3）恢复对应的日志   
  由上一步我们知道最新的日志节点为f815821a9c4e4833be898dace675916f3cad0124，我们可以查看这个节点的信息：
```
dengwen@ubuntu:~/project_DW/selog$ git show f815821a9c4e4833be898dace675916f3cad0124
commit f815821a9c4e4833be898dace675916f3cad0124
Author: dengwen <15680482464@163.com>
Date:   Fri Oct 12 17:11:22 2018 +0800

    add manage
```
接下来要做的就是恢复日志,同样，需要注意分支和日志节点。
```
dengwen@ubuntu:~/project_DW/selog$ git update-ref develop f815821a9c4e4833be898dace675916f3cad0124
```

* （4）提交最新的git log
  执行上述步骤后，用git status可以查看仓库的状态了，也就意味着git仓库恢复成功了。