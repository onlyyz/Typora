对应文件夹输入" CMD " 

#### Clone

```C++
git clone "  "
```

![image-20230713161908587](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230713161908587.png)

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

![image-20230713201925971](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230713201925971.png)

![image-20230713201656467](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20230713201656467.png)