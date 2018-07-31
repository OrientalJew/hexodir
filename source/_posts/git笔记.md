---
title: git笔记
date: 2018-07-27 14:24:10
tags:
- git
categories:
- git
---

<!-- toc -->

#### 安装

```
sudo apt-get install git
```

#### 配置

```
$ git config --global user.name "Your Name"
$ git config --global user.email "email@example.com"
```
<!--more-->
#### 创建Git空仓库

cd到特定目录下：

```
$ git init
```

#### 提交文件

```
$ git add readme1.txt
$ git add readme2.txt
...

# -m 后面跟提交的信息说明
$ git commit -m "第一次条readme文件"
```

#### 查看当前仓库的修改信息

git status能够告诉我们当前**本地仓库**所处的状态，哪些文件被添加、删除、修改等。
```
$ git status
```

如果需要查看详细的修改信息可以通过：
```
$ git diff readme.txt
```

#### 提交日志

查看提交日志：
```
$ git log | more

# 查看精简形式的日志
$ git log --pretty==oneline
```

#### 回退版本
HEAD表示当前本地仓库的版本；

HEAD^ 表示本地仓库的上一个版本；

HEAD^^ 表示本地仓库的上上个版本；

HEAD~num 表示本地仓库的上num个版本；
```
$ git reset --hard HEAD^
```
_注意：增删改并不意味这进入新的版本，也就是说，在当前版本进行了增删改操作后，使用上面的方式进行版本回退并不是回退到增删改之前的版本(这个是当前版本)，而是回退到上次从仓库更新时的版本(比仓库落后一个版本)，**如果想要内容回退到最后一次更新时的版本内容，应通过撤销本地修改记录来实现**；_

回退到指定ID的版本
```
$ git reset --hard commitID
```

查看每次回退操作记录(便于查找ID)：
```
git reflog | more
```


#### 工作目录(Working Directory)和版本库(Repository)

> 工作目录即我们创建仓库的目录。

> 版板库即工作目录下的.git目录。


#### git提交更改内容的步骤

把文件往Git版本库里添加的时候，是分两步执行的：

第一步是用git add把文件添加进去，实际上就是把文件修改添加到暂存区；

第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支;

默认情况下，git会为我们创建一个master分支，所以内容默认情况下会被提交到此。

![image](/images/git/1.jpeg)
![image](/images/git/2.jpeg)

_注意：修改的内容只有通过 git add 添加到暂存区，才能在git commit时被添加到仓库中。_


#### 撤销修改

- 撤销工作区的修改

```
# 分为两种情况：
# 一种是readme.txt自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
# 一种是readme.txt已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态；
$ git checkout -- readme.txt
```
_注意，只有在文件通过git add添加进过暂存区，文件才能被git进行跟踪管理，此时才能对其进行撤销操作，也就是说，对于工作区中新建的文件，上面的撤销命令是无效的。_


- 将修改从暂存区撤销回工作区

对于还没有提交的修改，可以通过以下命令，将内容从暂存区撤销会缓存区：
```
$ git reset HEAD readme.txt
```

- 对于已经commit的内容，可以通过**回退版本**来实现内容回退到上个版本的内容(当前无论是否做过修改，都是属于最新的版本，**通过更新是无法使内容回退到最近更新时的状态的，只能通过上面两种情况来撤销修改，或者先回退到上一个版本，再更新为最新版本；**)；


#### 删除文件

对于已经git add或git commit到仓库中的文件，已经被版本库跟踪变化，此时如果删除了文件：

1、如果要撤销删除，通过
```
$ git checkout --  readme.txt
```
2、如果是确实想删除文件，不想git再跟踪，则可以通过：
```
$ git rm readme.txt
```

### 远程仓库


- 创建SSH Key

进入用户家目录：
```
$ ssh-keygen -t rsa -C "youremail@example.com"
```
之后会在家目录下生成.ssh文件夹，里面有id_rsa和id_rsa.pub两个文件，id_rsa是私钥，不能泄露出去，id_rsa.pub是公钥，用于远程仓库识别用。

可以将id_rsa.pub内容复制给github的ssh key 管理，这样就能够与Github仓库建立连接。


- 与远程GitHub仓库关联

```
1、首先，创建一个github远程仓库，选择SSH类型的地址；

2、 根据github下面的提示，在本地仓库目录下进行链接：
$ git remote add origin git@github.com:OrientalJew/gitdir.git

3、将本地内容推送到远程仓库(第一次推送需要加上-u参数，用于关联本地和远程仓库)：
$ git push -u origin master
```
origin为远程仓库的名字，master表示远程仓库的分支，表示推送到远程仓库的master。由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。

以后推送只需要输入：
```
$ git push origin master
```

#### 分支管理1

每一个分支实际上代表着一个指针，每个分支的指针指向当前分支的最新版本。比如主分支master，其实际也是一个指针，指向当前master分支的最新版本。

HEAD指针则是指向当前我们正在编辑的分支的指针，比如如果当前分支是master，则HEAD指针指向的是master指针。

创建新分支实际上就是创建一个新的指针，并让HEAD指针指向当前的新指针：

1、创建dev分支，此时当前分支改为了dev分支，HEAD指针也会指向dev指针；
![image](/images/git/3.png)

2、接下来的修改都是针对当前dev分支的：

![image](/images/git/4.png)

3、合并master分支实际上就是让master指针指向当前dev分支的最新版本：
![image](/images/git/5.png)

4、删除分支实际就是把指针删除掉：

![image](/images/git/6.png)

#### 分支管理2

- 创建分支并切换分支
```
$ git checkout -b dev
```
等价于：
```
$ git branch dev
$ git checkout dev
```
- 查看分支
```
$ git branch
```
- 切换分支
```
$ git checkout master
```
- 合并指定分支内容到当前分支

```
$ git merge dev
```
- 删除分支
```
$ git branch -d dev
```
- 删除远程分支
```
$ git push origin --delete dev
```

- 未合并导致删除分支失败

如果在当前分支要删除一个没有合并内容到当前分支的分支，此时git将会报删除失败：
```
error: 分支 'temp' 没有完全合并。
如果您确认要删除它，执行 'git branch -D temp'。
```
此时我们可以：

1、切换到有合并删除分支内容的分支上，再尝试删除分支；

2、合并删除分支内容，再执行删除分支操作；

3、强制执行删除分支操作：
```
$ git branch -D temp
```

#### 解决分支冲突

当我们在两个分支同时修改了同一个文件时，在合并分支时就会出现冲突，此时需要我们手动修改分支的冲突；

输入：
```
$ git status
```
可以查看发生冲突的文件。

直接用编辑器打开冲突的文件，可以看到git已经为我们标记了冲突的内容：
```
Git tracks changes of files.
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```
> <<<<<<< HEAD ：表示当前分支存在冲突的内容。

> =======：用来隔开不同分支。

> '>>>>>>>' feature1：表示合并分支与当前分支冲突的内容。

直接修改冲突文件，并提交即可解决冲突：

```
$ git add readme.txt
$ git commit -m "conflict fixed"
[master cf810e4] conflict fixed
```
- 查看分支合并情况：
```
$ git log --graph
或
$ git log --graph --pretty=oneline --abbrev-commit
```

#### 提交带合并分支历史的记录

合并分支时，git默认会采用Fast forward模式，这种模式下，分支历史会被删除掉，不会展现在Log中；

如果想要在合并时，把分支的合并历史也添加进来，可以通过添加--no-ff参数来禁用Fast forward模式：
```
$ git merge --no-ff -m "merge with no-ff" dev
```
此时再查看log，可以看到合并历史也被添加进来：
```
git log --graph --pretty=oneline --abbrev-commit | more


*   08f8563 merge with no-ff
|\  
| * 9a8f00f 再次修改readme
|/  
* 00b7418 modify readme
* dd98a26 第三次修改
* 41eab06 修改了readme
* c919600 第一次条readme文件
```

#### 暂存某个分支的修改内容

有时，我们在某个分支上修改内容，此时任务还没有完成，又来了一个优先级更高的任务需要处理，我们只能先放下手头上的内容，先去处理该问题。
此时，我们需要把当前分支的内容进行保存，来到出问题的分支上开一个子分支处理问题。

- 保存当前分支内容
```
$ git stash
```
输入
```
$ git status
```
可以看到当前分支已经变得干净了，然后我们可以切换到问题分支上，假设问题分支是master分支：

```
$ gti checkout master
```
创建一个处理问题的子分支：
```
$ git checkout -b issue-1
```
处理完后提交，合并到主分支：
```
$ git checkout master

$ git merge --no-ff -m "merge issue-1" issue-1

$ git branch -d issue-1
```
此时，问题已经处理完成,可以切换回dev分支继续刚才的任务。

- 恢复分支原来的内容

输入：
```
$ git stash list
```
可以查看到git为我们保存的分支内容。

```
# 恢复分支内容
$ git stash apply

# 删除stash内容
$ git stash drop
```
或者直接通过：
```
# 恢复并删除stash
$ git stash pop
```

- 恢复指定的stash

如果存在多个stash，则需要根据名字来恢复对应的stash：
```
# 查看stash列表
$ git stash list

# 恢复指定的stash
$ git stash apply stash@{0}
```

#### 远程分支多人协作

- 查看当前仓库对应的远程分支
```
$ git remote

# 显示详细的推送和抓取分支信息地址
$ git remote -v
```

- 推送分支内容到对应的远程仓库上
```
# 推送主分支
$ git push origin master

# 推送开发者分支
$ git push origin dev
```

- 抓取分支到本地

```
$ git clone git@github.com:michaelliao/gitlearn.git [/workdir]
```

默认情况下，clone下来的内容只包含了主分支，如果需要其他分支的内容，需要自己手动创建同名的分支，并关联到远程的其他分支上：

```
$ git checkout -b dev origin/dev
```
之后在该分支修改的内容就可以直接commit，并且push到远程了：

```
$ git add env.txt

$ git commit -m "添加env文件"

$ git push origin dev
```

- 推送到远程分支之前需要先进行git pull

在多人协作开发远程仓库时，提交分支前一定要先进行git pull 操作，拉取远程仓库内容同步本地，解决可能的冲突后，再进行push。

```
$ git pull
```

如果本地分支没有与远程分支建立连接，git pull 失败，则通过：

```
$ git branch --set-upstream-to=origin/dev dev
```
建立连接，然后可以正常的git pull，之后才可以进行commit和push操作。

#### Rebase

当前远程仓库存在其他人提交记录时，我们进行pull操作之后，本地穿插入了远程的新记录内容，此时进行提交push操作，远程插入的记录会被记录到我们的提交记录中，如果不想这样，让我们每次提交都是基于远程仓库最新状态的基础上进行提交的，可以通过：

```
$ git rebase
```
进行整理。这样分叉的提交历史会被整理为一条记录。


#### 创建标签

我们可以为某个commit记录打上标签，以便后面回顾这个commit提交记录的相关信息。

- 创建标签
```
$ git tag v1.0
```
还可以对应具体的commit id打tag。

```
$ git tag v0.9 f52c633

# 带说明信息
$ git tag -a v0.1 -m "version 0.1 released" 1094adb
```
注意，标签是打在当前分支最新的commit上，与分支是无关的，只与某个commit id挂钩，在任何分支都能查到所有的tag。

- 查找所有的tag
```
$ git tag
```
- 显示具体的tag信息
```
$ git show v0.1
```

- 删除标签
```
$ git tag -d v0.1
```
#### 操作远程标签

- 推送指定标签到远程仓库
```
$ git push origin v1.0
```

- 推送所有标签到远程仓库

```
$ git push origin --tags
```

- 删除远程标签

```
$ git tag -d v0.9

$ git push origin :refs/tags/v0.9
```

#### 定义忽略文件

git可以定义仓库下不被识别提交的文件，这些文件在git status是不会被识别的。

只需要在仓库下创建一个.gitignore的隐藏文件，在其中可以定义忽略文件的文件名规则。

- Android下的部分忽略规则：
```
# Built application files
*.apk
*.ap_

# Files for the ART/Dalvik VM
*.dex

# Java class files
*.class
```
具体的各种忽略规则可参考：https://github.com/github/gitignore


- 提交.gitignore中存在的忽略文件

对于.gitignore中定义的忽略文件，也是可以强制提交的：
```
$ git add -f App.class
```

- 查看.gitignore文件中相对当前文件定义的忽略规则

```
$ git check-ignore -v App.class
```

#### 设置git命令别名

git支持对命令定义别名。

```
# 定义status命令别名
$ git config --global alias.st status

$ git st

# 定义checkout命令别名
$ git config --global alias.co checkout

$ git co
```
如果命令的别名需要带参考，则需要使用字符串的形式进行定义：

```
# 定义git log 带参数的别名
$ git config --global alias.lg "log --graph --pretty=oneline --abbrev-commit"

$ git lg
```

#### git配置文件

通过git config对git仓库进行配置时，如果带上--global则是对所有仓库进行配置。

每个仓库都有自己的配置，放在仓库下的.git/config文件中。
```
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@github.com:OrientalJew/gitlearn.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```


如果不带有--global，则只是针对当前仓库起作用。


#### 搭建git服务器

https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137583770360579bc4b458f044ce7afed3df579123eca000

#### git怎么查看两个版本具体的不同信息呢？

通过git diff只能查看当前修改与最新版本的不同信息；
