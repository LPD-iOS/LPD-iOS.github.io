---
layout: post
title: '简介我的 Git Work Flow'
date: '2017-05-08'
header-img: "img/post-bg-gitflow.jpg"
tags:
     - Git
author: '小鱼周凌宇'
---

# 重要性

我们从重要性说起。

团队开发中要重视有洁癖的人，这种人往往对糟糕的工作流不断提出意见、对 Git 的使用方式提出要求。如果你的团队中这种人正在不断的被忽视，那么你的团队一定出现了管理混乱、代码质量不高等等等等问题。

统一的工作流程是至关重要的，不管对于哪一个行业的作业来说都一样。对于我们开发人员，工作流包含了开发时 Git 的使用规范、Repo 管理的规范、测试过程的规范、设计交互的管理规范等等。由于测试、交互等设计到更多的人员，本篇文章暂且不表，重点说 Git 的使用规范和 repo 管理的规范。

本篇文章将讲述我在工作中一直使用的 Work Flow，希望对大家有帮助。

<!-- More -->

# 常见 Git 使用规范

先举一个例子，放上几张 Network 的图形截图。为了你的工程不变成下面这个样子，请善待 Git 的使用：

![示例1](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-01.png-w375)

![示例2](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-02.png-w375)

![示例3](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-03.png-w375)

这样一个乱糟糟的 Git，你们能忍我不能忍。

先来说说几张图种的问题：

1. 反向拉取 `develop` 分支
2. 不经过 `Pull Request` 的合并
3. 重复使用已经合并的分支
4. 没有意义的 `Commit Message`

**对于问题 1**，敢问这位同学，能不能用 `rebase`？很多同学在开发分支过程中，发现 `develop` 有更新，就去拉取 `develop` 的内容 Merge 进自己当前分支。对于 `rebase` 和 `merge`，请参考 [这篇文章](https://github.com/geeeeeeeeek/git-recipes/wiki/5.1-%E4%BB%A3%E7%A0%81%E5%90%88%E5%B9%B6%EF%BC%9AMerge%E3%80%81Rebase%E7%9A%84%E9%80%89%E6%8B%A9) 。引用里面的话：

> 每次合并上游更改时 `feature` 分支都会引入一个外来的合并提交。如果 `develop` 非常活跃的话，这或多或少会污染你的分支历史。虽然高级的 `git log` 选项可以减轻这个问题，但对于开发者来说，**还是会增加理解项目历史的难度。**

**对于问题 2**，完全就是习惯问题，对于所有的合并，如果是你自己的两个分支之间，如果非要直接 Merge 也不是不可，但是如果是两个人分别开发的分支，直接的 Merge 是不负责任的。`Pull Request` 或者 `Merge Request` 可以更早的帮你发现合并冲突，并且**强制你 review 代码**，这在保证代码质量方面起着至关重要的作用。

**对于问题 3**，已经合并入 `develop` 分支的分支，最好在 `Pull Request` 合并时直接勾选移除原分支，更容易保持和 `develop` 的同步。

**对于问题 4**，是最最最常见的问题，看看自己的项目，里面有多少个连续的 `Commit Message` 是『bug fix』、『update』、『pod add』、『修复』等等这样完全看不出啥内容的描述。敢问这样写的同学，你们的项目 Owner 看到 `Network` 时候是不是心里充满了 WTF？`Commit Message` 应当简短干练的描述这个 `commit` 做了什么。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-17.jpeg)

下面再看一个正面的示例，无比清爽的 Network：

![示例3](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-04.png-w375)

关于我的 Work Flow，我们从基本的 Git 开发流接着介绍。

## Git Flow

广为人知的 Git Flow 定义了一套标准的 Git 开发流。[这里是经典文章](http://nvie.com/posts/a-successful-git-branching-model/)。

大致的意思为：

1. `master` 是长期分支，一般用于管理对外发布版本，每个 commit 对一个 tag，也就是一个发布版本
2. `develop` 是长期分支，一般用于作为日常开发汇总，即开发版的代码
3. `feature` 是短期分支，一般用于一个新功能的开发
4. `hotfix` 是短期分支 ，一般用于正式发布以后，出现 bug，需要创建一个分支，进行 bug 修补。
5. `release` 是短期分支，一般用于发布正式版本之前（即合并到 `master` 分支之前），需要有的预发布的版本进行测试。`release` 分支在经历测试之后，测试确认验收，将会被合并的 `develop` 和 `master`

具体的也可以参考 [Git 工作流程](http://www.ruanyifeng.com/blog/2015/12/git-workflow.html)。

## Github Flow

在开发中 Git 往往搭配持续交付平台，Github 也好，GitLab 也好，都提供了完备的持续交付管理功能。配合这些就有了 [Github Flow](https://guides.github.com/introduction/flow/index.html)

![Github Flow](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-05.png)

大致意思为：

1. `master` 开出新分支，不区分功能分支或补丁分支。
2. 新分支开发完成后，或者需要讨论的时候，就向 `master` 发起一个 `Pull Request`。
3. 项目内人一起评审和讨论你的代码。对话过程中，你还可以不断提交代码。
4. `Pull Request` 通过，合并进 `master`，原分支就被删除。

可以看出 Github Flow 是 Git Flow 的简化版本，但是加上了一些合作关节的把控。

# 我的 Git Work Flow

我通常希望团队中的开发流程类似 Git Flow，但更为详细，大致为：

## 一、Git 分支部分

### master

长期分支，每个 `commit` 对一个 `tag`（一个发布版本）

![master](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-06.png-w375)

### develop

长期分支，日常开发汇总（开发版的代码）。

开发一个新的 feature 直接新在 `develop` 新开一个临时的 `feature` 分支，开发完成向 `develop` 提 `Pull Request``Pull Request`。

![develop](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-07.png-w375)

### `release`

短期分支，feature 开发完成后从 `develop` 拉出的分支，用于测试阶段，期间添加的 `commit` 基本都是 bug fix，开发结束后同时和并进 `develop` 和 `master`，`master` 打上发布 tag。

![release](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-08.png-w375)

### `hotfix`

短期分支，正式发布以后，进行 bug 修补

## 二、Git 操作部分

### commit 规范

1. `commit message` 言简意赅，不要写无用信息。不要出现 『update』，『Bug Fix』，这样让别人不能领其意的描述
2. 添加一个新的 `Pod` 库或 `pod update` 后，单独提交一个 `commit`，统一 `commit message` 为『pod add xxx』或 『pod update』
2. `commit` 之间保持独立，不要有修改同一个文件的情况。比如一个 `Pull Request` 中 commit1 在 FileA 中改了一个变量名， commit2 改回了变量名。原因是：**审核代码时，审核人通常会逐个 `commit`查看，而不是直接看 `Changes`（可以直接忽略掉 pod update 这样的 `commit` 不看）**

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-09.png)

### rebase

**不要出现反向拉取代码**的情况，即文章开头第一张、第二张图片中的情况——看到 `develop` 有更新，就将 `develop` 的代码拉取 merge 进自己的分支。

原因是：

1. `merge` 会导致你的分支都会引入一个外来的合并提交。如果 `develop` 非常活跃的话，或多或少会污染你的分支。
2. 丑，Network 复杂，增加理解项目历史的难度。

如何解决当前 `develop` 有更新的情况？

**请使用 `rebase`！**

`rebase` 用中文直译就是 `变基`。上张图帮助大家理解：

![rebase](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-10.png)

`rebase` 在进行时，需要选择一个 `commit` 点，将当前分支从根基整个移到指定 `commit` 点，名副其实——`变基`。

这样你既可以得到一个好看的 `Network`，又可以及时控制冲突。不过在多人开发中你需要多多关注 `develop` 的情况，及时 `rebase`，避免长时间不更新代码突然 `rebase` 到最新后发现了大量冲突。当然，控制和分配比较好的项目本身也很少产生冲突。

## 三、GitLab 管理规范

### issue

日常 Github 玩的转的同学都知道 `issue` 可以做很多事，比如：意见管理、Bug 管理、任务管理，可以只做一种功能，也能通过不同的 Label 同时使用所有的功能。

我的 Work Flow 中，`issue` 用来做任务管理，因为 `issue` 可以方便的指派，及时收到邮件通知。

每次新版本迭代开始，PRD 审核通过时，组内协商好任务分配后将任务拆成最小单元，由 Owner 分别建立 issue，大家自行领取。

如果有开发过程中发现的需要改进的地方，同样可以建立 issue。

关于 Label，我通常会分为：`optimizing`、`bug fix`、`feature`

![issue 管理任务](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-11.png)

一个任务完成时，通常会提 `Pull Request`，如果该 `Pull Request` 中完成了所有的任务，`Pull Request` 的 Title 应当类似以下格式：

> 『close #13 首页滚动栏切换效果完善』


GitLab 和 Github 都能识别 `close #{issue id}`，如果在 Title 中这样写，在 `Pull Request` 通过审核时，相应的 issue 会自动被关闭。

### Milestones

`Milestones` 即里程碑，`issue` 在建立的时候可以选择 `Milestones`，如果合理的使用了 `Milestones`，在 Milestones 页面，就可以得到一个清晰的项目进度。

![Milestones 页面](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-12.png)

### Pull Request

所有的合并都需要提 `Pull Request`，包括自己的分支合并到自己的分支，可以更好的帮助大家养成 Code Review 的好习惯。

`Pull Request` 的标题应该简介的介绍该次合并所做的事。更详细的内容应当在 `Description` 中逐条列出。如有相关文档链接也应列出。

![Milestones 页面](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-13.png)

注意选择合适的 `Milestone` 和 `Labels`。选择一位 Assignee 来审核，如果觉得该 `Pull Request` 内容过多，或有需要大家共同讨论的地方，再 `Pull Request` 提交后，在 `Discussion` 区域 `@` 其他人，所有人都会及时收到邮件。

![Milestones 页面](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-14.png)

### Code Review

`Code Review` 是一个很值得说的点。很多时候大家会以为 `Code Review` 是一定要读懂别人的代码，然后进行分析、审核。其实 `Code Review` 更多的是扮演了团队内经验传递的作用。

举个例子，代码规范这样的东西，就算是一个团队有了很详细的文档，但大家也不一定会去完整记下。对于新人，完成了 feature 后提 `Pull Request`，交由其他人 `Code Review` 时，由其他人审核代码规范，不合规要求继续修正，来回四五次，就再基本不会有问题了。

这也是我的亲身经历，我之前的一位 leader 对于 Work Flow 管理非常有经验，我在最初时提了几次 `Pull Request`，很快就熟悉了团队内的代码规范。

所以 `Code Review` 审核人应当检查的内容不是硬性的，但至少应当包括：

1. 代码规范
2. 基本语法和基本逻辑错误
3. 业务逻辑的一些经验
4. ...

在发现错误时，应当及时的添加 `comment`。

![Milestones 页面](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-15.png)

当审核人全部审核完毕，添加完所有的 `comment` 之后需要在 `Discussion` 区域 `@提交人 review done`，通知提交人。

同样，提交人在按照 `comment` 修改完后，也应当在 `Discussion` 区域 `@相关审核人 修改完成，请重新审核`。

需要着重说明的是：提交人的**所有修改，不允许新提交 `commit`**，应当在本地修改完成后，`ammend` 追加到最后 `commit`。

![Milestones 页面](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-16.png)

但是这一点，只是我一直使用的方式，原因同样是遵循『`commit` 之间保持独立』，如果提交新的 `commit` 导致两个 `commit` 修改了同一个文件。

当然也有人认为新加 `commit` 可以更清晰的看到提交者的新变动，也更符合 `Github Flow`。关于这里，就没有什么强制了，更喜欢什么就什么。

# 总结

最后，希望大家都收获一个清爽如风的 Network。

![示例3](http://7xt4xp.com1.z0.glb.clouddn.com/blog_work-flow-04.png-w375)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
