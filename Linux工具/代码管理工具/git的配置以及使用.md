```table-of-contents
```
# 配置

# 使用
## git clone
## git branch
## git log

### 查看某个函数的所有修改
```bash
git log -L :函数名:文件名

比如：
git log -L :kbuff_malloc:src/eal/kbuff.c
```


## git tag

## git status
## git remote
## git add
## git pull
## git rebase
### 常见使用
```bash

（1）git rebase -i COMMIT_ID
	1.1> 2个commit的合并；
	1.2> 已经提交commit的删除
	1.3> 已经提交cimmit的重新编辑
	1.4> 2个commit的先后顺序调整
```


### 提取已经提交的几个commit中对于某个目录的修改，为新的commit
#### 背景
比如：存在多个分支。
发布给线上用的分支为：1.0的分支；
开发的分支为：devel的分支。正常在devel分支上进行开发。

devel分支上可能存在一个新的功能模块（在一个新的目录下，比如： feature_new），1.0分支上没有这个功能模块。
每次commit的时候，可能修改了很多了文件，有的位于 feature_new 目录下，有的位于其他的目录下(其他的目录在1.0分支上也有)。

那么，这个commit可能就无法被1.0分支进行cherry-pick。

#### 需求
比如：已经在devel分支上提交了3个commit，修改了3个bug。但是这3个commit也涉及到 新的功能模块（在一个新的目录下，比如： feature_new）文件下的修改。

那么，这3个commit就无法直接的cherry-pick到1.0分支上。这就带来了麻烦。

#### 解决方法

**思路**：
从最近 3 个提交中，把 某个目录的改动提取出来，单独作为一个新的 commit，  并将原有 3 个提交中对该目录的改动清理掉。

**流程**：

（1）备份分支（防止出错可随时回退）
```bash
git branch backup-before-rebase
```

（2）生成并保存该目录的所有改动
回来原来分支，在开始前，先把最近 3 个提交中 `src/module/` 的所有变更保存成 patch 文件：
```bash
git diff HEAD~3 HEAD -- src/module > /tmp/module.patch

可以用 `git apply --stat /tmp/module.patch` 看看 patch 内容是否符合预期。
```

（3）启动交互式 rebase
```bash
git rebase -i HEAD~3

编辑器打开后会看到类似：
pick a1c2d34 Commit 1
pick b2e3f45 Commit 2
pick c3f4a56 Commit 3

把它改成：
edit a1c2d34 Commit 1
edit b2e3f45 Commit 2
edit c3f4a56 Commit 3

保存退出。
```

在 rebase 暂停时清理该目录的改动：
```bash
每次 Git 暂停在一个 commit 时执行以下命令：

# 从当前提交中移除 src/module 目录的改动
git reset HEAD^ -- src/module
git checkout -- src/module

# 保存该 commit（不再包含 src/module 改动）
git commit --amend --no-edit

# 继续下一个 commit
git rebase --continue

你只需重复这组命令三次（每次 rebase 停下来时都执行）。
```


（4）rebase 完成后，创建新的独立 commit
当 rebase 全部完成后，你的提交历史中最近 3 个 commit 已不再包含 `src/module` 的修改。此时重新应用并提交该目录的改动：
```bash
git apply /tmp/module.patch
git add .
git commit -m "Extract: 从最近3个提交中提取 src/module 改动为独立提交"

```

（5）强制推送远程仓库（可选）
由于 rebase 改写了历史，如果这些提交已推送远程，需要强制推送：
```bash
git push -f origin <branch-name>

or

git push -f
```


**小结**

|步骤|命令|说明|
|---|---|---|
|1️⃣ 保存目录变更|`git diff HEAD~3 HEAD -- src/module > /tmp/module.patch`|提前导出 patch|
|2️⃣ 启动 rebase|`git rebase -i HEAD~3`|对最近3个提交交互式编辑|
|3️⃣ 清理每个 commit 的改动|`git reset HEAD^ -- src/module``git checkout -- src/module``git commit --amend --no-edit``git rebase --continue`|重复三次|
|4️⃣ 重新创建新提交|`git apply /tmp/module.patch && git add src/module && git commit -m "Extract..."`|新建独立 commit|
|5️⃣ 推送|`git push -f`|改写历史|




## git diff
### 使用场景
如果开发环境和编译的环境不在一个机器上，那么就涉及到如何在这2个环境中进行更改代码的同步。

### 方法
通过 `git diff > xxx.diff` 产生 diff 文件，然后再目标机器上通过`git apply xxx.diff` 来应用diff。这样就可以实现2个设备上的更改同步。
注：`git diff` 要求更改还在工作区（即`git add .`操作之前）。

## git reset
### 常见使用
```bash
git reset --hard COMMIT_ID; git pull

git reset HEAD^

git reset --soft HEAD^
```
### 使用场景： 回退提交到工作区
#### 背景
如果开发环境和编译的环境不在一个机器上，那么就涉及到如何在这2个环境中进行更改代码的同步,「将上面的`git diff`」

`git diff` 需要更改在 工作区「即：`git add .`」之前，但是此时别人可能也提交了代码，此时需要通过`git pull --rebase REMOTE_NAME BRANCH_NAME` 来合并别人的更改。
在合并之前，需要将本地工作区的代码给保持下来，可能需要`git add .; git commit -m "xxxx"`；

合并之后，需要将自己的更改再回退到工作区，然后产生`diff` 文件，上传到目标的编译机器上。
#### 需求
用`Git`提交了最新的`commit`，但是现在想要回退到提交之前的状态，同时保留所有的更改。也就是说，他们不希望丢失当前的修改，只是撤销那次提交，让工作区回到提交前的状态，但保留那些改动，以便重新修改或再次提交。

#### 分析
要保留本地更改并回退到**最近一次commit提交「即完成了`git commit -m xxx操作`」** 之前的状态，可以按照以下步骤操作：

**（1）保留更改在暂存区（git add 后的状态）**
```bash
git reset --soft HEAD^
```
- 效果：撤销最后一次提交，但保留所有更改在暂存区（即 `git add` 后的状态）。
- 适用场景：需要重新修改提交信息或补充文件后再次提交。

**(2) 保留更改在工作区（未暂存状态）**
```bash
git reset HEAD^ # 或 git reset --mixed HEAD^
```
- 效果：撤销最后一次提交，保留所有更改在工作目录，但未暂存（即 `git add` 前的状态）。
- 适用场景：需要重新审查更改或选择性暂存部分文件后再提交。


**(3)注意事项**
- 如果已推送提交到远程仓库，回退后需用 `git push -f` 强制推送（慎用，确保协作成员知晓）。
- `HEAD^` 表示上一个提交，也可用 `HEAD~1` 替代，二者等效。

#### 小结
整体流程如下：
```bash
1> git add .
2> git commit -m "xxxx"

3> git pull --rebase REMOTE_NAME BRANCH_NAME
4> 如果有冲突，解决冲突

5> 将自身的更改，在回退到工作区。
   git reset --soft HEAD^

6> git diff > xxx.diff

7> 拷贝 xxxx.diff 到目标机器

8> 目标机器上执行：
git pull --rebase REMOTE_NAME BRANCH_NAME， 
然后 git apply xxx.diff  或者 patch -p1 < xxx.diff

```

## git blame
### 需求背景

### 范例

如下所示，查询`rdma-core`的哪个版本中开始支持了`ibv_query_gid_ex`函数 或者存在了`struct ibv_gid_entry`结构体，之前都是`ibv_query_gid`函数。

```bash
git clone https://github.com/linux-rdma/rdma-core.git 
cd rdma-core/libibverbs
git blame verbs.h  查看具体的哪个 commit中提交了
```
![](attachments/Pasted%20image%2020250624143215.png)


```bash
查看该提交在哪个版本中被包含。
git tag --contains  61f72f4768
```
![](attachments/Pasted%20image%2020250624142856.png)

# 参考
```bash

```