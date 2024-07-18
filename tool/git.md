

# 版本管理
使用git的时候，想要恢复到git的某一个版本，就需要使用版本回退的功能


## 版本提交

```shell
#把修改的文件保存到暂存区当中
git add xxx(file) / .

#把暂存区的内容提交到当前分支
git commit -m "xxx" 

#把当前分支的内容推送到远程
git push origin master
```



## 版本回退
使用 `git reset --hard ` 命令来回退版本，`HEAD`表示当前版本， `HEAD^` 表示回退到上个版本，两个^就是回退两个版本，如果要回退多个，可以用`HEAD~100`回退一百个版本



## 撤销修改

使用`git checkout -- file`的命令可以撤销对某个文件的修改


# 远程仓库的管理

```shell
#查看本地与远程仓库的管理
git remote -v 

#关联本地和远程仓库(origin通常为默认名称，可以改成其他的)
git remote add origin git@github.comxxxx

#第一次git推送
git push -u origin master
```





# 分支管理

## 创建分支
```shell
#创建并切换分支
git checkout -b xxxx
git swich -c  xxxx

#创建分支
git branch    xxx

#切换分支
git checkout  xxxxx
git switch    xxx
```




## 工作区暂存
修复bug时，我们会通过创建新的bug分支进行修复，然后合并，最后删除；

当手头工作没有完成时，先把工作现场`git stash`一下，然后去修复bug，修复后，再`git stash pop`，回到工作现场；

在master分支上修复的bug，想要合并到当前dev分支，可以用`git cherry-pick <commit>`命令，把bug提交的修改“复制”到当前分支，避免重复劳动。




# 状态查看


使用`git log`命令可以查看当前提交的版本信息，可以看到git提交的信息，包括`commit id`（`--pretty=oneline`参数可以简化输出）


使用`git reflog`命令可以查看执行的所有命令，其中包括head命令回退的版本，使用该命令可以查看到撤销以前的版本，可以再次复原到未来


使用 `git status` 命令可以查看当前git的状态


# 场景案例

## fork后的仓库如何同步原仓库


### 1. 暴力解法

删除fork的原仓库，重新进行fork

### 2.版本merge


1. 进入到本地仓库，使用`git remote -v`命令，查看当前仓库的远程分支

2. 使用`git remote add upstream xxxx`命令，把原仓库地址添加到本地当中，upstream通常就表示上游代码库

3. 使用git  commit 一套命令保存本地的更改

4. 使用`git fetch upstream` 命令抓取远程仓库的该更，随后 `git checkout master`更换到master分支当中去，最后使用`git merge upstream/master` 来进行本地代码和远程代码的合并，解决完冲突之后，再使用`git push` 命令来把命令推送到fork后的仓库当中，随后再进行mr


