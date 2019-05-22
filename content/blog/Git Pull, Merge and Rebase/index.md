---
title: Git Pull, Merge and Rebase
date: "2019-05-22T17:07:00.000Z"
description: "Git Pull, Merge and Rebase"
---

三种操作都可以进行代码的整合。

## Merge

### Merge by Fast-forward

![Fast-forward](https://git-scm.com/book/en/v2/images/basic-branching-4.png)
这种类型就相当于把主线直接提前到和merge近来的支线的位置（master -> hotfix），是最简单的。

### Merge by Three-way

所谓三方，如下所示

![ThreeWay1](https://git-scm.com/book/en/v2/images/basic-merging-1.png)

通过生成一个真正的改变的集合体(snapshot C6)来完成整合。

![ThreeWay2](https://git-scm.com/book/en/v2/images/basic-merging-2.png)

```bash
$ git checkout master
Switched to branch 'master'
$ git merge iss53
Merge made by the 'recursive' strategy.
index.html |    1 +
1 file changed, 1 insertion(+)
```

## Rebase

对于三方（基方，分之一，分支二）整合，通过rebasing同样可以做到。本质就是把从基方开始一方的所有更改都重新加到另一方上面。

```bash
$ git checkout experiment
$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: added staged command
```
在这里，就是把C4的更改加到C3上面进而创造出C5并指向它， 也就是先把你所在的那个分支上所有的更改都保存到临时文件上去，然后把当前分支指向你要base的那个分支(名义上你当然还在自己的分支上)，并把临时文件的更改都加上去。

![Rebasing1](https://git-scm.com/book/en/v2/images/basic-rebase-3.png)

然后你可以把master分支fast forward到最新
```bash
$ git checkout master
$ git merge experiment
```

![Rebasing2](https://git-scm.com/book/en/v2/images/basic-rebase-4.png)

### 一个复杂点的例子 

![Rebasing3](https://git-scm.com/book/en/v2/images/interesting-rebase-1.png)

告诉系统把client与Server不一样的东西找到并rebase到master上去。

```bash
$ git rebase --onto master server client
```

![Rebasing3](https://git-scm.com/book/en/v2/images/interesting-rebase-2.png)

### 优点
很清晰的提交历史，线性的。

### 禁忌

不要在别人工作的分支上使用rebase。

> Do not rebase commits that exist outside your repository and people may have based work on them.

换句话说，千万不要在公共分支上使用rebase。 

> Once you understand what rebasing is, the most important thing to learn is when not to do it. The golden rule of git rebase is to never use it on public branches.

## Pull

git pull  == git fetch + git merge [origin/master] 
git pull --rebase == git fetch + git rebase

[有人](https://coderwall.com/p/7aymfa/please-oh-please-use-git-pull-rebase)认为在本地应该永远使用rebase，这样使得log非常清晰。如果你也赞同，可以通过[如下设置](https://coderwall.com/p/yf5-0w/like-git-pull-rebase-make-it-default)使其变成默认行为。

```bash
git config branch.autosetuprebase always
```
如果是已有分支

```bash
git config branch.YOUR_BRANCH_NAME.rebase true
````

如果临时想用merge

```bash
git pull --no-rebase
```
## 致谢

* https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging#_basic_merging

* https://git-scm.com/book/en/v2/Git-Branching-Rebasing

* https://stackoverflow.com/questions/5601931/what-is-the-best-and-safest-way-to-merge-a-git-branch-into-master

* https://www.quora.com/What-are-the-differences-between-git-pull-git-pull-origin-master-and-git-pull-origin-master

* https://www.atlassian.com/git/tutorials/merging-vs-rebasing


