---
title: 5种 git 工作流
date: 2020-08-28 10:55:26
thumbnail: /img/git-workflow.png
tags: 
- git
categories:
- 开发技巧
toc: true
---
团队协作开发中合并代码是件很令人头疼的事情，尤其是在快要上线的时候发现有很多冲突需要解决。所以使用一种合适的 git 工作流可以为您的开发流程带来很大的好处。

当然，即使是用了合适的 git 工作流，也不能解决所有的问题，但这是正确方向的一部。毕竟，当多人开发同一特性时，版本管理工具的使用是至关重要的。

如何选择合适的 git 工作流取决于您的项目类型，团队规模，发布频率等因素，在本文中，我们将介绍5种 git 工作流的优缺点，以及适用场景，let's go !
<!-- more -->
本文翻译自 [https://zepel.io/blog/5-git-workflows-to-improve-development/](https://zepel.io/blog/5-git-workflows-to-improve-development/)

# 基础工作流
最简单的 git 工作流是只有一个分支 - master 分支，开发人员直接提交到 master 分支上，并使用 master 分支部署。

{% asset_img Basic-git-workflow-3.png 简单 git 工作流 %}

这种工作流通常不推荐，除非你的团队/项目规模非常小，并且你们想快速起步。

因为只有一个分支，所以不涉及合并分支，这是的 git 使用变的很简单。但是这样会带来以下几个缺点：
1. 多人合作时，需要不停的 rebase / merge 代码。
2. 很有可能将别人不想上线的代码部署到线上环境。
3. 代码缺少 review，后续代码会很难维护。

# 功能分支工作流
当有多个开发人员开发项目时，就必须使用 git 分支了。
假设有多个开发者在同一分支上开发不同功能，他们向同一分支提交代码，这将使代码库变的非常混乱，并产生大量冲突。

{% asset_img Feature-Branch-git-workflow-4.png 功能分支工作流 %}

为了解决这个问题，不同的开发人员可以从 master 分支创建两个独立的分支，并分别开发他们的功能，当一个人完成自己的功能时，可以分别合并到 master 分支并发布到线上，无需等待第二个人开发完功能。
这个工作流程的优点是，你在开发自己功能的时候不必担心代码冲突。

# 功能分支工作流改进版（增加开发分支）

该工作流是开发团中比较流行的工作流之一，它类似于前一种工作流，但是相比前一种工作流多了一个 develop 分支，该分支是和主分支并行的。

这个工作流中，线上部署的代码始终是 master 分支的。团队想要部署到生产环境中，必须从 master 分支进行部署。

develop 分支上是下一个交付版本的功能叠加代码，开发人员从 develop 分支创建不同分支进行开发，开发测试完毕后，合并到 develop 分支。当所有功能都开发合并到 develop 分支后，在 develop 分支进行测试，通过之后再合并到 master 分支，然后进行 测试，发布，上线。

{% asset_img feature-branch-with-develop-git-workflow-2.png 功能分支工作流改进版（增加开发分支） %}

这个工作流的优点是，线上代码会很明确，出现线上问题时，不会因为有新代码影响到排查问题。但是这个流程对于一些团队来说比较冗长乏味。

# gitflow 工作流

gitflow 工作流相比前一种工作流，又增加了 hot-fix 和 release 两个分支。

## hot-fix 分支

hot-fix 分支可以从主分支 checkout 出来，开发完成后直接合并到 master 分支，而不是 develop 分支。它仅在有必须快速修复的线上问题时使用，这种分支的优点是，允许快速的部署到线上环境，不中断其他人的开发流程。

一旦 hot-fix 分支被合并到 master 分支并部署，它也应该被合并到 develop 分支和 release 分支，这样做是为了确保后续开发的功能是在最新 master 分支代码基础上开发的。

## release 分支

release 分支是在 develop 分支和 master 分支中间的一层。这个分支存放一些文档，修复 bug 等非功能性代码。一旦这个分驻与主分支合并并部署到生产环境中后，也需要合并到 develop 分支里，这样保障了后续开发等新功能都是在最新等 master 分支代码基础上开发等。

{% asset_img GitFlow-git-workflow-2.png gitflow 工作流 %}

[Vincent Driessen](http://nvie.com/posts/a-successful-git-branching-model/) 首先发布了这个工作流，随后这种工作流慢慢流行起来了。

git-flow 是一个基于 git 开发的工具，你可以在现有的代码库里安装 git-flow 来帮助你创建分支。
- mac 上使用 `brew install git-flow` 安装。
- window 上需要(下载安装)[https://git-scm.com/download/win]。
安装完成后，使用 `git flow init` 初始化工作流。