对应文件夹输入" CMD " 

#### Clone

```C++
git clone "  "
```

#### 初始化，仓库

```C++
git init
```

#### 暂存区 

```C++
git add -A 			All
git add <file>
    
git commit -m "detail"
```

#### 打回

```
git checkout
```

#### 提交撤回

```
git reset HEAD^1 
```



```c#
从当前节点创建分支
git checkout -b <branch name>
列举所有分支
git branch
切换分支
git checkout <branch name>
删除分支
git checkout -D <branch name>
合并分支
git merge <branch name>
//rename 本地
git branch -m name name
```



#### SSH

```cpp
//查看是否有SSH
ls ~/.ssh
//generate
ssh-keygen -t ed25519 -C "Email"
// 
cat ~/.ssh/id_ed25519.pub
    //添加到git ssh
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIO0oy1nO18EgEqAWzsOFeHIuZ3itzQdvKbGA3q/33Aw8 2713421980@qq.com

```



## 提交项目

```C#
//初始化
git init
	git status

git add .
	git status
	
git commit -m "first commit"
	git status
	
git branch -M main

//link
git remote add origin 
git remote add origin git@github.com:onlyyz/Vulkan_learn.git

git push -u origin main
git push --set-upstream origin main


git commit -m "first commit"
```

#### 忽略文件夹

Create gitignore

VS code set

```C++
*# Directories*

.vs/

bin/

bin-int/

*# Files*

*.user
```

git

```C++
//add
git add *
//设置文件夹
git reset
    
//commit    
git commit -m "Setup basic Application and Entry Point"

//push
git push origin main/master
```

## 引用其他git库

```c++
git submodule add "URLpath" "filePath"
example:E:\dev\Hazel>git submodule add https://github.com/gabime/spdlog.git Hazel/vendor/spdlog

E:\Item\Resource>git submodule add http://192.168.8.104:3300/Art_Resource/Model.git E:\Item\Resource\Model
```

#### 查看文件历史

```c#
git log --pretty=oneline 文件名
```

但是，类似 `--since` 和 `--until` 这种按照时间作限制的选项很有用。 例如，下面的命令会列出最近两周的所有提交：

```console
$ git log --since=2.weeks
```

#### 历史版本

```c#
git show <git提交版本号> <文件名>
git show 34b7ac981a39193ca78e0d4673ce66515c23988dframeworks/base/packages/SystemUI/AndroidManifest.xml
```

 将文件还原到你想要还原的版本

```c#
git checkout ${commit} /path/to/file。
即$ git checkout 052c0233bcaef35bbf6e6ebd43bfd6a648e3d93b /path/to/fileche
```



### 冲突

```c#
$ git checkout --ours <fileName>
$ git add <fileName>   //标记该冲突已解决
$ git rebase --continue 
$ git status //查看当前冲突状态
```

场景1：修改了文件/path/to/file，没有提交，但是觉得改的不好，想还原。
解决：

```c#
git checkout -- /path/to/file
```

 将文件还原到你想要还原的版本

```c#
git checkout ${commit} /path/to/file。
即$ git checkout 052c0233bcaef35bbf6e6ebd43bfd6a648e3d93b /path/to/file
```

#### ours与theirs

在上述两种合并中，都可能会产生冲突，需要通过手动解决。如果想要保留两个分支中的某一个可以使用

```c#
git checkout --ours <fileName>
```

或者

```c#
git checkout --theirs <fileName>
```

这里需要注意的是，一定要知道哪个分支对应ours或theirs。

#### 还原文件夹

要还原某个文件夹，你可以使用 Git 中的 `checkout` 命令。以下是还原文件夹的步骤：

1.  确定要还原的 Git 仓库所在的目录，使用终端进入该目录。
2.  确认要还原的文件夹的名称，以及在哪个分支中还原。
3.  运行 `git checkout <branch_name> -- <path_to_folder>` 命令来还原指定分支中的指定文件夹，其中 `<branch_name>` 为要还原的分支名称，`<path_to_folder>` 为要还原的文件夹路径。

例如，如果要还原分支 `master` 中的文件夹 `src`，可以使用以下命令：

```css
git checkout master -- src
```

这将会将 `src` 文件夹还原到 `master` 分支的最新版本。如果你想还原到之前的某个提交，可以使用该提交的哈希值来代替 `master`。

请注意，运行 `git checkout` 命令将更改你的工作目录和文件系统中的文件。在还原文件夹之前，一定要确保已经将所有未保存的更改保存并提交到 Git 仓库中。

#### 放弃本地所有修改

两个命令的使用
要放弃本地所有修改并还原到最近的提交状态，可以使用以下 Git 命令：

git reset --hard HEAD
1
这个命令将会重置 HEAD 指针和工作目录，将它们恢复到最近的提交状态，并且丢弃所有未提交的修改。



## [合并分支](https://github.com/zeus911/blog-3/blob/master/git/merge_files_or_folders_from_other_branches.md#合并分支)

使用`git merge`命令进行分支合并是通用的做法，但是`git merge`合并的时候将两个分支的内容完全合并，如果想合并部分肯定是不行的。那怎么办？

如何从其他分支合并指定文件到当前分支，`git checkout`是一个合适的工具。

```
git checkout source_branch <path>...
```

#### 忽略

https://www.freecodecamp.org/chinese/news/gitignore-file-how-to-ignore-files-and-folders-in-git/
