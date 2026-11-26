---
title: Fork 工作流的概念
date: '2026-11-25T11:23:47+08:00'
summary: 'Fork 工作流解决了什么问题？'
draft: false
---

## 概念

开源领域中，Fork 是最常用于多人协作开发一个共同项目的工作流。在参与一个开源项目之前，有必要理解 Fork 的一些概念。

想象一下，你开发了一个很酷的应用，本来是放在自己的电脑上，开发，测试，运行，以及相关的文档编写，随着应用的不断完善，你想要倾听下其他人的意见和看法，可以通过邮件或者论坛的方式把这个应用源码发布到互联网上。

其他人看到了你的项目，想要一份源码在本地编译运行或者修改，最古老的分发方式是通过真实的快递，邮寄一个刻录有源码的光盘或者U盘，稍微互联网一些的方式是共享目录，网盘，邮箱附件。这些好奇的开发者中，总会有几个积极的人想要为你的开源项目添砖加瓦（修复 BUG，添加新的功能），如果把补丁代码继续通过邮寄或者电子邮件发送，那么对于维护者本身的压力是很大的。

收到了补丁代码，你需要手动录入或者复制到项目中，然后进行一系列的测试，如果幸运的话，补丁代码不用修改或者仅需要少量改动就可以合并到这个开源项目中，当然合并的过程也是手动的，合并好后就需要再次发布，形成了一个完整的开源项目的循环。

以上，就是一个没有 Git 或者 GitHub 的古法开源项目的工作流，这对于一个非常小的开源项目来说也许行得通，但是对于一些由全世界开发者共同参与的大型项目（编译器、数据库、浏览器内核、操作系统内核等等）就很痛苦了：贡献者提交的补丁代码很可能在提交后就已经落后于源码好几个版本，维护者需要手动收集和追踪每一个补丁的状态。大型项目的贡献者一般是没有直接写入到中央仓库的权限的，只有维护者确认没有问题了，贡献者的代码才由少数几个维护者手动合并到中央仓库中。

2005 年 Git 诞生（为了管理 Linux 内核），带来了一个关键性质：每份克隆都是完整的仓库，带着全部历史，没有技术上的"中心"。所谓"主仓库"，只是大家约定去认的那一份，是一种社会共识，不是技术限制。对于开源项目的贡献者和维护者而言，最重要就是能否创造一个分布式的工作流，解决“能为大型、自发性的团队（包括了不受信的第三方）提供灵活的方式来安全协作”的问题。

Git 的天然性质支持一种做法：每个人都有自己的完整仓库，如果你想把改动交给别人，不是"写进对方的仓库"（这样可以避免不受信任的第三方贡献者搞乱仓库），而是让项目的维护者主动来拉取我的仓库里的改动，这种思想被 GitHub 做成了 Pull Request 功能：向主仓库发起合并请求，由维护者审核后合入。

Git 解决了分布式版本管理的问题，但是对于贡献者来说，要搭建自己的仓库，不仅有技术问题，还有成本问题，我想要参与一个开源项目，我还得先搭建自己的 Git 服务器，这未免有些麻烦。2008 年 GitHub 上线，把搭建自己的仓库并开放给他人的问题解决了。

重新回到刚刚的例子，你不再需要通过邮件或者论坛来发布你的代码和产品，在 GitHub 开源你的项目，愿意为你项目贡献的其他人 Fork 你的主仓库到他们自己名下，他们可以在 Fork 出来的仓库中随意改动代码，当贡献者觉得可以合并到你的主仓库中时，他们会调用 GitHub PR 的功能：一键向原项目提出"请看看我的改动，愿意的话请合并"，并附带讨论、评审的界面。于是"陌生人贡献代码"从邮件补丁时代的体力活，变成了几分钟的流程，这是开源协作大爆发的重要原因之一。

Fork（中文解释：（公路、河流等的）分岔处，分流处，岔口；岔路），之所以选用这个名字是来源于 Unix 的 fork() 系统调用（进程复制自身产生子进程）是程序员最熟悉的"fork"。路径分叉的意象也很直观：从同一个点出发，之后各走各的路，颇有一些另起炉灶的感觉。事实上，Fork 也确实不仅仅用于贡献，因为社区矛盾或路线分歧，Fork 也有可能意味着和主仓库的“分道扬镳”，比如：MySQL → MariaDB，OpenOffice.org → LibreOffice，Elasticsearch → OpenSearch。

---

## GitHub Fork

理解了为什么需要 Fork，再来看一次完整的贡献流程是什么样的。假设你想给一个开源项目 octocat/hello-world 修复一个 BUG，其中 octocat 是原项目（upstream，上游仓库）的所有者，yourname 是你自己的 GitHub 用户名。

### Fork：在服务端复制一份到自己名下

打开原项目页面，点击右上角的 Fork 按钮。GitHub 会在你的账号下创建一个完整的副本 yourname/hello-world，包含全部代码和提交历史。这份副本完全属于你，你拥有它的写权限，同时 GitHub 会记录它来自哪个上游仓库。

### Clone：把自己的 Fork 克隆到本地

注意要克隆的是你自己的 Fork，而不是原仓库，因为你对原仓库没有推送权限：

```bash
git clone https://github.com/yourname/hello-world.git
cd hello-world
```

### 添加 upstream：保持与原项目的联系

此时本地仓库里只有一个远程地址 origin，指向你的 Fork。为了日后能获取原项目的最新代码，需要再添加一个指向原仓库的远程地址，约定俗成命名为 upstream：

```bash
git remote add upstream https://github.com/octocat/hello-world.git
git remote -v   # 检查：origin 指向你的 Fork，upstream 指向原仓库
```

### 创建功能分支：不要直接在 main 上改

先同步一下上游，再基于最新的 main 开一条新分支：

```bash
git fetch upstream
git checkout -b fix-login-bug upstream/main
```

为每个改动单独建分支是个好习惯：一个 PR 对应一个分支，互不干扰，之后同步上游时也不容易产生冲突。

### 开发并提交

在分支上正常修改、测试、提交：

```bash
git add .
git commit -m "fix: handle empty password on login"
```

提交前建议先阅读项目的 CONTRIBUTING.md，很多项目对提交信息格式、代码风格、测试要求都有明确规定。

### Push：推送到自己的 Fork

```bash
git push origin fix-login-bug
```

推送的目标是 origin（你的 Fork），而不是 upstream，这正是 Fork 工作流的关键：你从头到尾都没有向原仓库写入任何东西。

### 发起 Pull Request

推送成功后，打开你的 Fork 页面，GitHub 通常会提示 Compare & pull request，点击后选择：

base repository：octocat/hello-world 的 main 分支（要合并进去的目标）
head repository：yourname/hello-world 的 fix-login-bug 分支（你的改动）

然后填写标题和说明，讲清楚改了什么、为什么改、如何验证。提交之后，你的 PR 就出现在原项目的 Pull Requests 列表里了。

### 评审与迭代

维护者会审阅你的代码，可能留下评论、要求修改，项目的 CI 也会自动跑测试。如果需要修改，直接在同一个分支上继续提交并推送，PR 会自动更新，无需重新发起：

```bash
git add .
git commit -m "fix: address review comments"
git push origin fix-login-bug
```

### 合并与收尾

维护者认可后会点击 Merge，你的改动正式进入原仓库。此后可以清理现场，并把你的 Fork 同步到最新状态：

```bash
git checkout main
git fetch upstream
git merge upstream/main      # 或 git rebase upstream/main
git push origin main         # 让自己 Fork 的 main 也保持最新

git branch -d fix-login-bug               # 删除本地分支
git push origin --delete fix-login-bug    # 删除 Fork 上的远程分支
```

---

参考

1. https://gitee.com/opensource-guide/guide/participating/first-pull-request.html
2. https://www.atlassian.com/git/tutorials/comparing-workflows/forking-workflow
3. https://docs.gitlab.com/user/project/repository/forking_workflow/
4. https://docs.github.com/en/pull-requests/reference/forks
