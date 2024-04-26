# Git指令

```shell
#添加远端
git remote add origin <远程主机名>
#拉代码
git pull <远程主机名> <远程分支名>:<本地分支名>
git pull origin dev	#拉取远端dev分支

#推代码
git push <远程主机名> <本地分支名>:<远程分支名>

#本地库将文件加入暂存
git add <file>
git add .#将所有文件加入暂存

#查看文件暂存状态
git status

#提交暂存区修改
git commit -m "commit message"

#分支
git branch#查看分支情况
git checkout <分支>#切换分支
git branch <branch_name>#在当前分支的基础上创建新的分支

				  
```

