---
title: Git 命令笔记
date: 2018-10-06 16:24:59
tags: Git
categories: Git
comments: true
---

- 初始化仓库
``` bash
# 初始化一个 git 仓库并且建立工作目录
git init
# 初始化一个干净的仓库，使用 bare 选项不带有工作目录，使用 shared 选项，提供组可写的权限
git init --bare --shared
```
- 克隆仓库
``` bash
# 在未指定协议的时候优先会采用 ssh
git clone [地址]
```
- 暂存文件
将文件提交至暂存区（进入 staged 状态）
``` bash
git add [路径]
# 只暂存所有已经跟踪的文件，这通常可以减少多余文件的提交
git add -u
```
- 查看当前工作区以及暂存区的文件状态
``` bash
# 该命令会展示所有文件的状态，包括它是否已经修改，是否被跟踪，是否被暂存
git status
```
- 查看工作区和暂存区文件差异
``` bash
# 查看工作区和暂存区快照间的差异（文件与上一次 add 时的差异）
git diff
# 查看工作区和上次提交时的差异（文件与上一次 commit 时的差异）
git diff --staged
git diff --cached
# 可以携带版本号，可以指定文件名产看，版本间指定文件的差异
# 第二版本号省略时默认为当前版本，文件路径省略时默认所有文件
git diff [版本号] [版本号]
git diff [版本号] [版本号] -- [文件路径]
# 查看在之前祖先基础上的差异需要在源版本号前添加'...'，如：
git diff master...feature
```
- 提交更新
``` bash
# 在暂存区准备妥当之后，把暂存区内容提交
git commit
# 在提交时会要求输入提交备注，可以直接在命令行键入
git commit -m [提交备注]
# 偶尔跳过暂存步骤，将工作区全部的修改直接提交
git commit -a
```
- 文件删除
``` bash
# 删除动作会将文件删除，并且在完成动作时修改暂存区
git rm [路径]
# 如果该文件已经被本次修改后暂存（staged）需要强制删除
git rm -f [路径]
# 从仓库中删除而不删除工作区文件，即放弃跟踪
git rm --cached [路径]
```
- 文件移动
``` bash
# 如同文件删除，移动命令会在移动后对暂存区修改
git mv [源路径] [目标路径]
```
- 查看提交历史
``` bash
# 查看历史时会展示历次提交的哈希值、作者、时间以及提交备注
git log
# 使用 p 选项展示内容差异
git log -p
# 可以指定展示条目数量
git log -[N: int]
# 展示增删改行数统计
git log --stat
# 按单词维度检测修改内容，需要同时指定显示的上下行数
git log --word-diff -U[N: int]
# 支持图形化展示
git log --graph
# 支持格式化记录的显示
git log --pretty=[FormatMode]
# FormatMode 常用形式
# oneline 单行显示，仅显示提交的哈希值和备注
# fuller 完整显示，包括作者，创建时间，提交者，提交时间，提交备注等
# format:[格式字符串] 按指定格式输出，具体占位符含义查看手册
# 查看在 A 分支上而不在 B 分支上的提交，有多种写法
git log B..A
git log ^B A
git log A --not B
# 查看在多个分支上仅不在 B 分支上的提交
git log A C --not B
git log A C ^B
# 查看在 A 或 B 上，但不同时存在于 A 和 B 上的提交，使用 left-right 选项区分提交所在分支位置
git log --left-right A...B
```
- 文件修改
    - 提交修改
    ``` bash
    # 如果遗漏部分提交，可以重新更新暂存区后使用 amend 选项修改之前次的提交，比如如下操作
    git commit -a -m 'first commit'
    touch a.txt
    git add a.txt
    git commit --amend -m 'first commit with a.txt'
    ```
    - 撤销暂存区快照
    ``` bash
    # 取消暂存状态的修改
    git reset HEAD [路径]
    ```
    - 撤销工作区修改
    ``` bash
    # 这会使工作区的内容还原至上一次提交时的状态，慎用
    git checkout -- [路径]
    ```
    - 还原代码至指定的版本（也可以向上）
    ``` bash
    # mixed 默认选项，回退 HEAD 并还原暂存区，所有修改保留在工作区
    # soft 选项，仅回退 HEAD，保留暂存区和工作区的改动
    # hard 选项，将回退 HEAD 并还原暂存区和工作区内容，彻底的回退
    git reset [--soft/hard/mixed] [版本号]
    ```
- 远程仓库
``` bash
# 远程仓库列表，默认仅显示仓库名称，v 选项列出具体地址
git remote -v
# 添加远程仓库
git remote add [远端名称] [远端仓库地址]
# 展示远端具体信息，包括未跟踪的分支、提交的信息等
git remote show [远端名称]
# 重命名远程名称
git remote rename [源远程名称] [新远程名称]
# 删除关联的远程仓库
git remote rm [远程名称]
```
- 标签处理
``` bash
# 展示所有标签，
git tag
# 使用 l 选项过滤标签，可以使用通配符
git tag -l 'v1.*'
# 创建轻量级标签，如果版本哈希值省略，则为当前版本打标签
git tag [标签名称] [版本哈希值或分支名或标签名]
# 创建携带附注的标签，使用 m 选项可以直接输入标签备注
git tag -a [标签名称] -m [标签备注]
# 使用 GPG 签名创建标签
git tag -s [标签名称]
# 验证标签，仅对于签名的标签
git tag -v [标签名称]
```
- 分支处理
``` bash
# 查看所有分支，默认展示本地分支，使用 r 选项展示远端分支，a 选项全部展示
git branch -a
# 新建分支，如果版本哈希值省略，默认在当前版本新建分支
git branch [分支名称] [版本哈希值或分支名或标签名]
# 新建分支后并不会直接切换至该分支，需要进行切换
git checkout [分支名称]
# 新建并检出一个分支
git checkout -b [新分支名称] [版本哈希值或分支名或标签名]
# 使用 track 选项简化上述命令，直接新建同名分支并跟踪远端分支
git checkout --track [远程名]/[分支名称]
# 使用 d 选项删除分支
git branch -d [分支名称]
```
- 拉取和推送
``` bash
# 大部分时候，远程名称可以省略，默认使用 origin
# 拉取远程仓库的数据（这不会直接进行合并
git fetch [远程名称]
# 向远端推送数据，远程分支名称省略时推送至已关联的远程分支上，本地分支名称和远程分支名称一起省略时可以推送当前分支至远程对应分支上
git push [远程名称] [本地分支名称]:[远程分支名称]
# 删除远程分支
git push [远程名称] :[远程分支名称]
```
- 变基操作
``` bash
# 可以将某一些特性在指定的基分支上重演，省略特性分支时默认为当前分支
git rebase [基分支] [特性分支]
# 有时候需要变基的特性分支和并不直接基于基分支，在需要跳过中间分支的情况下需要使用 onto 选项
git rebase --onto [基分支] [跳过分支] [特性分支]
# 变基过程中可能遇到冲突，git 会自动停下等待处理
# 冲突处理完成后继续
git rebase --continue
# 放弃处理冲突，放弃变基
git rebase --abort
```
- 使用补丁
``` bash
# 使用 diff 创建一个简单的补丁，diff 的输出就是一个标准的 patch 内容，例子如下：
git diff master >> [补丁路径]
# 打上补丁，之后需要手动提交
git apply [补丁路径]
# 添加补丁前最好测试是否存在冲突
git apply --check [补丁路径]
# 使用 git format-patch 创建补丁，该补丁已邮件方式存在，并包含提交者信息，可以添加额外的描述，通常使用 M 选项检测重命名
git format-patch [比较的版本号]
git format-patch HEAD^^
# 使用 format-patch 创建的补丁需要使用 am 指定来确认
git am [补丁路径]
# 如同变基操作一样，在遇到冲突或者无法快速合并时候，git 会停下让你解决冲突或者放弃
# 冲突解决完成，继续
git am --[continue/resolved/r]
# 放弃并还原
git am --abort
# 跳过该补丁
git am --skip
# 在存在公共祖先的时候，git 可以更智能地合并
git am -3 [补丁路径]
```
- 发布
``` bash
# 发布前准备，需要打上一个标注标签（使用 a 或 s 选项）
# 查看内部版本号，通常用来做归档压缩包的名称
git describe [分支或标签名称]
# 进行归档，可以使用 prefix 增加根目录，使用 format 指定格式
git archive [分支或标签名称] --prefix='[增加的根目录名称/]' --format=[zip/tar/tar.gz/...]
# 上述命令仅输出二进制内容，需要将其输入至归档文件，如
git archive master --prefix='project/' --format=zip > `git describe master`.zip
```
- 储藏内容
git 可以帮助储藏一份临时性的改动便于之后恢复
``` bash
# 将已跟踪的文件的改动进行储藏
git stash
git stash push [描述]
# 查看储藏栈内所有的储藏记录
git stash list
# 将储藏内容还原至工作区和暂存区，名称省略时使用栈顶记录
git stash apply [stash@{N}]
# 删除储藏栈中的记录
git stash drop stash@{N}
# 清除全部储藏站中的记录
git stash clear
# 将储藏内容还原，并立即移除记录
git stash pop stash@{N}
# 还原应用的储藏内容，通过 show -p 选项展示 diff patch 内容，并使用 apply -R 来撤销应用
git stash show -p [stash@{N}] | git apply -R
# 直接从储藏中新开分支来避开冲突解决，使用栈顶记录时可省略储藏名称，完成时会自动移除储藏栈记录
git stash branch [新分支名称] [stash@{N}]
```