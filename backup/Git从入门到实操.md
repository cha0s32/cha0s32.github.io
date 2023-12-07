## Git 基础



### 创建版本库

```shell
git init 
```

### 添加及提交到仓库

```shell
git add xxx.file
git commit -m "xxxx"

## 看看工作区和暂存区的区别
git diff xxx.file
```

### 版本回退

```shell
# 查看最近三次commit 记录

git log --pretty=oneline
```

HEAD 表示当前版本， HEAD^ 表示上一个版本， HEAD～100表示前100个版本

```shell
git reset --hard HEAD^ # 回退到上一个版本
```

Git在内部有个指向当前版本的`HEAD`指针，当你回退版本的时候，Git仅仅是把HEAD从指向指定版本

想从旧版本恢复到新版本，但是新版本的`commit id`用`git log`找不到了：

```shell
git reflog
```

可以看到每一步HEAD指针的变更记录，从而找到新版本的`commit id`

```shell
git reset --hard commit_id
```





### 撤销修改

```shell
git checkout -- file_name
```

- 自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态
- 已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态



如果修改已经提交到暂存区了，使用`git reset HEAD file_name`可以把暂存区的修改撤销掉，重新放回工作区

然后再`git reset -- file_name`把工作区的文件还原



### 远程仓库

第一步，在 github上创建远程仓库

本地文件夹关联远程仓库 （origin也可以改成其他的）

```shell
git remote add origin git@github.com:cha0s32/xxx.git
```

第一次把本地内容推送到远程仓库

```shell
git push -u origin master # -u 会把本地master分支和远程master分支关联起来
```

查看远程库信息

```shell
git remote -v
```

接触远程库和本地的绑定关系

```shell
git remote rm 远程库的名字
```



从远程库上克隆项目

```shell
# ssh

git clone git@github.com:cha0s32/xxx.git

# https
https://github.com/cha0s32/xxx.git # 比较慢

```



## 分支管理

### 创建与合并分支

```shell
git checkout -b dev # 创建名为dev的分支,并切换过去
# git switch -c dev
```

改动 dev 分支的文件，然后提交

```shell
git add .
git commit -m "a new branch"
```

切换回master分支

```shell
git branch master # git switch master
```

合并 dev 分支

```shell
git merge dev
```

删除 dev 分支

```shell
git branch -d dev
```



### 解决冲突

当两个分支同时改动同一个文件，并且都提交了以后，要 merge 需要手工解决冲突

新建新分支

```shell
git switch -c test
```

原 master 中有一个 readme 文件，内容为 

```shell
aaaaaaaa
bbbbbbbb
```

且已提交

新分支中，也有readme文件，内容为

```shell
aaaaaaaaa
bbbbbbbba
```

也已提交，现打算在主分支用`git merge`合并，会出现冲突的提示，使用`cat readme`可以看到两个文件的区别

在主分支的`readme`中作改动，然后再次提交，解决冲突

```shell
git add .
git commit -m "conflict commit"
```

然后可以删掉 `test`分支



### 分支管理策略

通常，合并分支时，如果可能，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

<img src="https://cos.kevinc.ltd/file/download?fileId=969" style="zoom: 33%;" />



<img src="https://cos.kevinc.ltd/file/download?fileId=970" style="zoom:33%;" />

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

简单得说，就不需要手动解决冲突了，直接生成新的`commit`。

命令如下：

```
 git merge --no-ff -m "merge with no-ff" dev
```



### Bug 分支

情况： 有 Bug 的情况下，要新建一个分支进行bug修复，但当前的工作还不能提交

先把当前工作现场存储起来

```shell
git stash
```

切换到需要改bug的分支，新建新分支

```shell
git checkout master
git checkout -b bug-1.0.1
```

修改bug，并提交（记住commit id，切换到开发的分支后可以将修改合并）

```shell
git add .
git commit -m "xxxx"
```

合并bug分支

```shell
git merge --no-ff -m "merge" bug-1.0.1
```

回到开发分支，恢复工作现场

```shell
git switch dev

git stash list
git stash pop # 恢复同时把stash删掉

/*
也可以这样
git stash apply
git stash drop
*/

# 恢复指定的stash
git stash apply stash@{0}
```

把前面bug修改复制到dev分支

```shell
git cherry-pick {bug-commit-id}
```



## 好用的github工作流



### 1. 三个部分

- remote  
- local
- disk



### 2. 流程

- 拉取远端仓库

```shell
git clone repo_address
```

- 在local git 处 新建一个 分支 feature branch

```shell
git  checkout -b my-feature
```

这个时候`local git` 上就存在两个分支，`main branch` 和 `my-feature branch`

- 修改 硬盘 代码， 修改完以后使用 `git diff`看看硬盘上文件和git上保存的branch有什么区别
- 使用 `git add file-name`将改动存在暂存区里，然后使用`git commit` 将代码提交到`local  git `上
- 使用 `git push origin my-feature`, 此时 `github` 项目上多了一个`my-feature`分支



### 3. 同步

如果此时发现main 直线里 多了一个 commit 名为 update

想要把 update 的commit 也同步到 branch 里面

- 执行 `git checkout main`， 将 `disk` 的代码还原为 `init` 的状态
- 执行 `git pull origin master` , 把远端的`update`同步到`local git` 的`main`分支下
- 执行 `git checkout my-feature`，切换回 `my-feature`分支
- `git rebase main`, `rebase`的意思是先把 `my-feature`对`init`的改动先扔到一边，然后`main`最新修改拿过来，最后把`my-feature`修改尝试弄回去，有可能会出现`commit conflict`，出现冲突要手动选择
- `git  push -f origin my-feature`， 把最新改动 `push`到`github`上



### 4. Pull Request

将 `my-feature`的改动尝试合并在`main`分支里

- `pull request`
- 主分支如果接受合并，则执行 `squash and merge`,意思就是把其他分支的各种commit合并为一个改变，并把这个commit 放到 main branch
- 合并后一般把远端的feature-branch 给删掉
- local git 上 还存在 feature-branch
- `git checkout main`, 首先切换到 local git 的 main 分支下
- `git branch -D my-feature`, 把 my-feature branch 从local git 上也删掉
- `git pull origin master`, 把最新的更新拉到 local 的git 和 disk 上

