layout: name
title: Git命令
date: 2015-04-02 17:07:45
categories: Git
tags: Git
---

命令 | 解释 | 备注
--- | --- | ---
git init | 初始化一个Git仓库
git add <file> | 添加 | 反复多次使用，添加多个文件
git commit | 提交
git status | 工作区的状态
git diff | 查看修改内容
git reset --hard commit_id | 返回到某版本 | HEAD指向的版本就是当前版本
git log | 查看提交历史 | 以便确定要回退到哪个版本
git reflog | 查看命令历史 | 以便确定要回到未来的哪个版本
git checkout -- file | 直接丢弃工作区的修改 | 未add
git reset HEAD file | 去掉暂存区，之后还要进行上面步骤 | 已经add，未commit
<!-- more -->
