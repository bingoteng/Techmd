# 一、git基础语法

```bash
#查看版本号：验证安装
git --version
```

```bash
# 设置全局用户名（你的GitHub/Gitee昵称）
git config --global user.name "window"

# 设置全局邮箱（代码平台注册邮箱）
git config --global user.email"1840892765@qq.com"

#查看配置是否生效
git config --global --list

#1.名字/邮箱只是标识可以任意，但邮箱最好和远程仓库对应，可以查看提交记录
#2.<--global>全局标识，存在：对本电脑任意仓库生效，不存在：进对当前仓库生效
#3.标记这段代码是谁写的，只存在你本地电脑，和GitHub 平台账号无关。
```

```bash
#切换分支到 main
git checkout main 
#创建并切换到新分支
git checkout -b wb_test

#个人分支只要不合并到主分支，所有的临时修改、注释，都不会影响主干分支main

git branch -r

git brach -v

git branch -d
# 删除本地分支（仅本地，远程分支不受影响）
git branch -D my_debug_note

```

```bash
#初始化当前文件夹为git仓库
git init

#暂存当前文件夹的所有改动
git add  .

#提交一次版本到本地仓库
git commit -m "备注：本次提交标识"

#查看仓库状态
git status
```

```bash
#查看当前文件夹绑定的所有远程仓库
git remote -v

#添加公司远程仓库，别名 origin
git remote add origin git@git.company.com:team/project.git

#添加个人远程仓库，别名 personal
git remote add personal git@gitee.com/yourname/project.git

#删除指定远程仓库关联
# 完整写法
git remote remove origin
# 简写等效
git remote rm personal

# 删除origin远程的wb_best分支
git push origin --delete wb_best
```

```bash
#推送
git push
#第一次推送本地分支到远程（远程不存在该分支）
git push -u origin wb_test

#首次推送/后续推送？
git remote add origin https://github.com/bingoteng/test.git
#输入远程仓库名
#个人密钥
#拉取
git pull

git fetch 
git diff
```

```bash
#克隆远程分支到当前文件夹
git clone

#本地没有 Git 仓库，这时我们可以直接将远程仓库clone到本地。通过clone命令创建的本地仓库，其本身就是一个 Git 仓库了，不用我们再进行init初始化操作啦，而且自动关联远程仓库。我们只需要在这个仓库进行修改或者添加等操作，然后commit即可

# 克隆公司主干代码到本地（拿到完整工程）
git clone git@公司git地址:项目.git
cd 项目文件夹
# 同步主干最新代码
git checkout main
git pull origin main
# 从主干新建并切换到本地分支 my_debug
git checkout -b my_debug

```

```bash
# 暂存当前未提交的所有修改，分支不变
git stash save "临时保存调试注释，未完成"
# 查看所有stash存档
git stash list
# 恢复保存的修改
git stash pop
```

```bash
#打印仓库提交日志
git log 

q退出
'space'下一页
```

```bash
#切换到main分支，将wb_best分支合并到main分支
git merge wb_test

```

```
SSH地址配置

ssh key 

github->settings->SSH and GPG keys->New SSH keys->crtl +v
```

首次http推送
![image-20260723004456645](https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260723_00-45_ab68e180d103b309215c38dc19f12bde.png)

<u></u> 

```
hui tui
git checkout <commit 哈希>

git checkout ace2fc77adf6caacd584c1929ba0bc637e1f6f4f

```

