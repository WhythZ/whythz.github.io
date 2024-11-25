---
# author:
title: 对比分支管理中的merge与rebase
description: >-
  介绍在合并分支时使用`git merge`指令和`git rebase`指令的区别，并介绍多人协作中使用`rebase`需要遵守的黄金准则，以及若不遵守有怎样的后果
date: 2024-11-25 17:50:00 +0800
categories: [技术经验, Git]
tags: [Git]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
# math: true
# mermaid: true
image:
  path: /resources/2024-08-15-Git入门基础知识汇总/GitLogo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

>关于Git的基本操作讲解，参考我[对应的博客](/_posts/2024-08-15-Git入门基础知识汇总.md)

## 一、对比`rebase`与`merge`
- `rebase`又称变基，其和`merge`都是用于合并分支的指令，但二者的合并方式有着较大的差异
	- `git merge`
		- 完整保留所有提交记录及其分支结构，并保持提交ID不变
		- 可追溯性较高，但复杂的结构会导致难以进行代码审核与漏洞追查
	- `git rebase`
		- 会在合并后以分支的共同根提交记录作为基底，将分支结构融合重组为一条线性结构，这会导致某些提交ID的改变
		- 使得提交记录较为整洁，但可能会导致丢失部分重要信息，尤其是分支冲突时

![CompareMergeOrRebase.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebase.png)

- 如上图所示，其相当于是在`feature`分支上执行了`git rebase develop`，原有的记录C、D、E都被新建为新的分支变基到了B记录上，形成了一条线性结构，这条线性记录仍然归属于调用这条指令的`feature`分支上，`develop`分支仍保留原有的提交记录不变

## 二、搭建测试用例

- 我在第一次提交中创建了下图的`test.txt`文件，提供了对`Function-1`和`Function-2`的`A`实现

![CompareMergeOrRebaseP1.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP1.png)

- 此时创建一个新分支`bran`，然后进行如下修改提交
	- 在`master`分支上将`Function-1`的实现先后改为`Implementation-X`、`Implementation-XX`
	- 在`bran`分支上将`Function-2`的实现先后改为`Implementation-Y`、`Implementation-YY`

![CompareMergeOrRebaseP2.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP2.png)

![CompareMergeOrRebaseP3.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP3.png)

![CompareMergeOrRebaseP4.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP4.png)

![CompareMergeOrRebaseP5.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP5.png)

## 三、进行`merge`会怎样
- 此时切换到`master`分支上使用`git merge bran`进行合并（将`bran`合并到`master`上），结果如下图所示，可以看到所有的提交记录都被保留了下来，并且所有commit的ID编号都不会被改变

![CompareMergeOrRebaseP6.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseP6.png)

## 四、进行`rebase`会怎样
- 回退到`cbe2319`版本，然后切换到`master`分支上

![CompareMergeOrRebaseBeforeTest.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseBeforeTest.png)

- 调用`git rebase bran`指令后，我们需手动将被换基的提交记录逐一与新基进行桥接，体现为冲突的处理（即便前后被桥接的两个版本一模一样，没有冲突，也需要进行手动处理该过程）
- 处理完毕后执行`git add`暂存，然后使用`git rebase --continue`表示完成桥接（实际上这个过程就相当于创建了一个新的提交记录），然后就可以继续下一个提交记录的桥接了，直到所有被转移的提交记录都被处理完毕，他们在这个过程中被创建为新的提交记录
- 我的测试用例中的`master`上仅有两个需要被转移的提交记录，所以仅需进行如下的两次处理

![CompareMergeOrRebaseConflictP1.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseConflictP1.png)

![CompareMergeOrRebaseConflictP2.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseConflictP2.png)

- 完成上述两步后即完成了整个变基合并流程，由于我们是在`master`分支上执行的`git rebase bran`指令，所以最终这条线性的记录是保存在`master`分支上的

![CompareMergeOrRebaseConflictP3.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseConflictP3.png)

- 分支`master`中除了基底`0b4e138`记录未被改变ID，其余两条则是被新建为了新的相同信息的提交记录变基到了另一个分支`bran`的最新提交记录上（可以使用`git reflog`来寻找原有的提交记录，而不是使用`git log`）
- 而`bran`分支的提交记录并未被改变，如下所示，`bran`仍维持在其原本最新的提交记录版本

![CompareMergeOrRebaseConflictP4.png](/resources/2024-11-25-对比分支管理中的merge与rebase/CompareMergeOrRebaseConflictP4.png)

## 五、黄金准则
- 黄金准则指的是多人开发场景下，使用`rebase`时必须遵守的一个规范

```
>> "No one shall rebase a shared branch"
>> "不应当在共享的分支上执行git rebase another_branch指令操作"
```

- 什么是共享的分支？如下图所示，两位开发者Bob和Anna都需要在`feature`分支上进行修改，那么`feature`分支就是共享分支
- 可以看到下图中的Bob违反了黄金准则，此时Bob还未将修改推送到远程仓库，所以暂未发生问题，他的本地仓库中原先D记录的ID已经被新记录D'替代了，而Anna的本地仓库的D未发生改变，这就埋下了隐患，待我们逐步分析Bob这样做的严重后果

![黄金准则P1.png](/resources/2024-11-25-对比分支管理中的merge与rebase/黄金准则P1.png)

- Bob将他的本地仓库推送到远程仓库后，远程仓库就如下图所示，当Anna此时也想要推送时，就需要先进行同步

![黄金准则P2.png](/resources/2024-11-25-对比分支管理中的merge与rebase/黄金准则P2.png)

- Anna执行`git pull feature`指令后，自己的本地仓库就会被变成下面这个鬼样子，但是冲突确实解决了，可以进行推送操作

![黄金准则P3.png](/resources/2024-11-25-对比分支管理中的merge与rebase/黄金准则P3.png)

- 假设Anna此时确信Bob会遵守黄金准则而不检查的话，那么在她将此时的本地仓库直接推送到远程仓库的话，那么远程仓库也会变成这个鬼样子，乱七八糟不说，并且由于存在大量重复的提交版本，还会导致远程仓库空间的浪费

![黄金准则P4.png](/resources/2024-11-25-对比分支管理中的merge与rebase/黄金准则P4.png)

- 这还仅仅是两个人协作中发生的`rebase`误用，尚且如此严重，此时若出现第三个共享`feature`分支的人想要将其实现的`F`推送到远程仓库，就需要先拉取，拉取后就变成了下图的样子

![黄金准则P5.png](/resources/2024-11-25-对比分支管理中的merge与rebase/黄金准则P5.png)