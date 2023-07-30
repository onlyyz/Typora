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



```
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

```
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
