##  前言

在平时的开发中避免不了使用 git 来管理代码，尤其是多人协作开发时更是如此。git 使用不好轻则一堆乱七八糟的追踪，重则 git push --force 覆盖队友代码（那些年的强制推送与删库跑路），这一篇主要介绍一下代码合并的两个重要命令，`git merge` 与 `git rebase`。

## 1. 操作前的准备

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100441.png)

**master** 分支内容如下

```txt
index.txt
// 内容
master-1 // commit 内容 master-1
master-2 // commit 内容 master-2
master-3 // commit 内容 master-3
```

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100442.png)

**feature** 分支内容如下

```txt
index.txt
// 内容
master-1 // commit 内容 master-1
master-2 // commit 内容 master-2
|-- feature
|   |-- index.txt
// index.txt 内容
feature-1 // commit 内容 feature-1
feature-2 // commit 内容 feature-2
```

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100443.png)

## 2. git merge

首先我们切到 master 分支，然后在控制台运行 `git merge feature` 命令。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100445.png)

这里就是 commit 的信息了，我们修改为 master-4，然后保存。然后再查看 git 记录会发现如下图。

> git merge feature 会从重现 feature 分支上做的更改，从 master-2 到 feature-2，然后将其结果记录成一个新的 commit 保存到 master 分支，同时也会以另一条线记录 feature 分支的提交记录。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100446.png)

如果我们不想有这么杂乱的提交，只想保持一个干净整洁的记录，而且也不需要 feature 分支的提交记录，那么我们可以使用一个参数 --squash，这个参数可以帮助我们压缩分支线。

首先还原到上次提交: `git reset HEAD^`

然后再删掉从 feature 分支 merge 过来的文件，再 merge 一下：`git merge feature --squash`，于是就有了下图的结果。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100448.png)

然后 `git status` 查看。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100449.png)

再 `git commit -m 'master-4'` 提交。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100450.png)

这样就达到了我们想要的效果了。

## 3. git rebase

在使用 rebase 之前我们也还是切换回到上一次提交，`git reset HEAD^`，删除多余的文件，然后切换到 feature 分支，`git reset HEAD^` 回到 feature-1 的提交。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100452.png)

此时 master 分支的提交是 master-1、master-2、master-3；feature 分支的提交是 master-1、master-2、feature-1。然后现在要在 feature 上开发 feature-2，但是缺少了一个 master-3 的提交。

通过 rebase 来将 master-3 给拉下来。

输入命令 `git rebase master`，然后看 git 的提交记录。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100453.png)

神奇的事情发生了，master-3 居然出现在了 feature-1 前面，这是为什么？

> git-rebase - Reapply commits on top of another base tip

当前分支在其他目标分支基础上重新提交，通俗地讲就是先讲 feature-1 提交“取出来”，然后同步 master 分支，再讲 feature-1 提交“放回去”。

此时我们再加入 feature-2，`git add feature/index.txt`，`git commit -m 'feature-2'`，就得到了下图所示的结果。

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100454.png)

我们合并到 master 分支，`git checkout master`，`git rebase feature`。

为了更明显一点，我们在 master 分支 rebase 前再进行一个 master-4 的提交。

在 master 分支里的根目录 /index.txt 里新增一行 master-5，然后 `git add index.txt`、`git commit -m 'master-4'`，得到如下结果：

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100455.png)

然后 `git rebase feature`

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100456.png)

![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100458.png)

这里就相当于把 master-4 提交先“取出来”，然后同步 feature 的提交到 master，再将 master-4 的提交“放回去”。最后得到上图的结果，追踪了所有的提交，并且分支树也是干净整洁的。

## 总结

总的来说，git merge 是从两个分支分离的时候开始合并，然后将功能分支的提交重现到主分支上，形成新的提交，同时也会在分支树形成枝叶来记录功能分支的提交，可以通过 --squash 参数来压缩。

git rebase 则是会先“取出”当前提交，再改变原有的分支基础（会将两个分支分离的地方进去往前的推进，达到同步），最后再将新的提交给“放回”到同步后的提交上。不会产生新的提交记录，也不会歪曲分支树。

**如何选择哪一种方式呢？**

这里我在工作中的 git 结构是

+ master——主分支，需要追踪每一次 develop 以及 hotfix 的提交。
+ develop——开发分支，需要记录每一个功能的开发，但是不需要 feature 分支各种小的提交，只需整合成一个 feature 即可。
+ feature—— 功能分支，个人在本地创建（多人协作时可能在远端也有），用来开发单个功能，功能完成后合并到 develop 分支。在这里可能本地会有很多的小的提交改动，但是 develop 分支不需要记录追踪。

于是就有了以下的选择：

feature 分支上有大量杂乱细小无关紧要的提交，在功能完成后，我们需要整合到一个 commit 提交给 develop 分支。

**1. 基础**
![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100459.png)
**2. feature 分支 git rebase develop —— 変基同步 develop 可能其他同事的提交。**
![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100500.png)
**3. develop 分支 git merge feature --squash —— 合并 feature 分支到 develop 分支，并压缩提交形成整洁分支树。**
![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100504.png)
**4. master 分支 git merge develop —— 合并 develop 分支到 master 分支，追踪每一个 develop 分支的功能提交。**
![i](https://yywc-image.oss-cn-hangzhou.aliyuncs.com/2019-11-27-100505.png)
