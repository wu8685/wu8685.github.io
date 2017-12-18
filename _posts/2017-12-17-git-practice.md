---
layout: post
title: Git Practice
categories: [git]
tags: [git]
---

# GIT版本控制实践

## 流程控制

![git practice]({{ "/img/posts/2017-12-17-git-practice.md/git-practice.png" | prepend: site.baseurl }})

* 最稳定的代码放在master分支上，不要直接在master分支上提交代码，只能在该分支上进行代码合并操作，例如将其它分支的代码merge到master分支上。

* 伴随master分支，checkout一条develop分支供所有人访问。但一般情况下，我们也不会直接在该分支上提交代码，develop上的代码主要是从其他的分支merge过来。

* 在开发新的功能时，我们从develop分支checkout新的功能开发分支，如feature-1，feature-2。然后与此功能相关的代码都提交到此分支上。

* 当各个功能开发完毕后，准备发布某个版本了。此时需要从develop分支上拉出一条release分支，例如release-1.0.0，并将需要发布的功能从相关feature分支一同合并到release分支上。然后将release分支发布到测试环境，测试工程师在该分支上做功能测试，开发工程师在该分支上做bug fix。

  * 测试过程中，可以定期对release分支打tag。如每一期的测试结束后，可以在release分支上打上tag 1.0.0-rc1，1.0.0-rc2，以此类推。

* 待测试工程师完成测试时，我们可将该release分支部署到预发环境，再次验证无任何bug后，可将release分支部署到生产环境。待上线完成后，将release分支上的代码同时合并到develop分支与master分支，并在master分支上打一个最终的tag，例如1.0.0。

  * 若上生产时发现bug，可以继续在release上修复，并继续打release-rc的tag

* 在生产运行时发现bug需要修复时，可以从tag 1.0.0上checkout一个hot fix分支，例如hotfix-1.0.1，然后在此分支上进行bug fix。在提交并完成测试后，则merge到master和develop上。然后在master打上新的tag 1.0.1，并将其发布到生产。

版本号格式为x.y.z，其中：

x为大的功能发布或重构时会升级；
y为一般功能发布时会升级；
z为有了bug被修复后会升级。

## 常用命令

* 查看/修改git信息

```
git config --list

git config --global user.name ke.wu
git config --global user.email ke.wu@163.com
```

* 创建/删除新branch

```
git checkout -b <branch-name>
# 删除本地分支
git branch -D <branch-name>
# 删除远程分支
git branch origin :<branch-name>
```

* 设置/交互upstream，比如github上的开源项目

```
git remote add upstream https://github.com/coreos/jetcd.git

# 更新所有上游分支
git fetch upstream
# 更新某个上游分支，例如master
git fetch upstream/master
# rebase上游分支
git rebase upstream/master
```

* push/pull当前分支

```
git push origin <branch>
git pull origin <branch>
```

* 回到之前的commit

```
git checkout <commit-id>
```

* 重制到制定commit

```
git reset <commit-id>

# 此时当前commit之后的提交都还在本地，且可查看
git status
```

* squash压缩多个commit

```
# 压缩n个commit
git rebase -i HEAD~n

# 在弹出的编辑器中选择commit，此处将c6 dd 6d三个commit压缩到1f中
pick 1fc6c95 do something
squash 6b2481b do something else
squash dd1475d changed some things
squash c619268 fixing typos

# push
git push -o origin <branch>
```

* 从开发分支（feature-1）merge到release-1.0.0分支

```
# git checkout feature-1
git rebase release-1.0.0
# 此处fix conflict并commit到feature-1分支，如果有多个conflict，则会用到continue
git rebase --continue
# 若想放弃
git rebase --abort

# 换到release分支并merge
git checkout release-1.0.0
git merge feature-1
```

## comment中引用

[reference](https://gitlab.com/gitlab-org/gitlab-ce/issues/30204)

| input | reference |
| --- | --- |
| namespace/project#123 | issue |
| namespace/project!123 | merge request |
| namespace/project%123 | milestone |
| namespace/project$123 | snippet |
| namespace/project@9ba12248 | specific commit |
