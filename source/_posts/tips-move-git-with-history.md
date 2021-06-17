---
title: 在迁移 Git 仓库时保留历史
catalog: true
header-img: /img/header_img/tips-header.jpg
tags:
  - Git
categories:
  - 经验
date: 2021-06-17 15:41:39
subtitle:
---



公司做“组件化”的时候热爱新建 Git 仓库，但是有些代码被搬来搬去之后丢掉了 Git 历史，导致不好追溯代码是怎么被写残的。所以在这里简短的存档一下文明有爱的迁移步骤。

## 操作步骤

假定要将仓库 A 下面的 "/module_0" 目录迁移到新建的仓库 B 中。

### 在本地仓库 A 挑选 Git 历史

这一步的目标是从仓库 A 中挑选仅仅和 "/module_0" 目录有关的 Git 历史。

* 在初始化本地仓库 A

> `git clone <git repository A url> repoA`
> `cd repoA`

* (optional) 删除远端，防止将远端仓库写坏了。实际上在生产环境中，远端的重要分支会被 "protected"，可以忽略这个步骤。

> `git remote rm origin`

* 删除掉与 "/module_0" 目录无关的历史。注意这一步将破坏本地的 Git 历史，如果还有没同步到远端的提交，建议先 push 到远端再进行这一步。

> `git filter-branch --subdirectory-filter <directory e.g. module_0> -- --all`


### 同步本地仓库 A 的 Git 历史到本地仓库 B 

* 在初始化仓库 B 的本地仓库

> `git clone <git repository B url> repoB`
> `cd repoB`

* 为本地仓库 B 添加一个远端，指向本地仓库 A

> `git remote add remote-a <directory e.g. repoA>`

* 同步本地仓库 A 特定分支的文件和 Git 历史到本地仓库 B

> `git pull remote-a <branch> --allow-unrelated-histories`

* 删除指向本地仓库 A 的远端

> `git remote rm remote-a`

### push 仓库 B 到远端

> `git push origin HEAD:<remote branch>`

## 闲话

* `git-filter-branch` 是一个关键命令, [Understanding Git Filter-branch](https://manishearth.github.io/blog/2017/03/05/understanding-git-filter-branch/)。
*  在使用 `git-filter-branch` 时，命令行弹出了 Warning 建议使用 [git-filter-repo](https://github.com/newren/git-filter-repo/) 来 替代 `git-filter-branch` 命令。
