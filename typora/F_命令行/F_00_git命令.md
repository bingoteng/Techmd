# Git实操踩坑笔记（Techmd笔记仓库，按首次使用操作流程排序）

> 适用场景：从零配置Git、搭建个人笔记仓库，收录完整操作命令、对应报错原因、解决办法，可直接保存为 `Git实操踩坑笔记.md`

## 一、初始Git全局配置（最先操作，整台电脑通用）

### 1. 查看当前完整配置

```bash
git config --list
```

已有的关键配置项说明：

| 配置项                         | 作用                                               |
| --------------------------- | ------------------------------------------------ |
| `user.name`、`user.email`    | 提交代码的身份标识，每条提交都会附带该信息                            |
| `core.autocrlf=true`        | Windows自动做换行转换：拉取文件转为Windows的CRLF换行，提交仓库统一存储LF换行 |
| `core.quotepath=false`      | 终端正常展示中文文件名，不会转义成`\xxx`编码                        |
| `core.longpaths=true`       | 解除Windows 260字符路径长度限制，适配深层级笔记目录                  |
| `init.defaultbranch=main`   | 新建仓库默认主分支为main，不用老旧的master                       |
| `credential.helper=manager` | Windows凭据管理器缓存账号密码，HTTPS推送无需重复输入账号密码             |

### 2. 修改提交用户名（配置身份）

```bash
# 修改全局用户名，后续所有仓库生效
git config --global user.name "新用户名"
git config --gloval user.email "<同远程仓库一致>"
# 仅修改当前单个仓库的用户名
git config user.name "新用户名"
git config user.email "<同远程仓库一致>"
# 查询当前全局设置
git config --global --list
```

## 二、本地创建仓库、编写文件（第二步）

### 1. 初始化本地Git仓库

进入项目根目录后执行：

```bash
git init
#执行后目录生成隐藏的`.git`文件夹，它是仓库核心数据库，保存所有版本记录、远程配置、分支信息，**严禁随意删除**。
```

### 2. 整理本地目录结构（移动文件夹）

需求：把零散A、B、C…Z开头的笔记文件夹，统一移入`typora`总目录

- 推荐用法（`git mv`，保留文件移动历史）

```
mkdir typora
# 批量移动多个目录
git mv A* B* C* D* E* X* Y* Z* typora/
git commit -m "整理目录：所有笔记归入typora文件夹"
```

### 3. 文件暂存、本地提交

```bash
git add .
git commit -m "首次提交笔记文件"
```

## 三、绑定远程GitHub仓库、尝试HTTPS推送（第三步）

### 1. 关联远程仓库

```bash
# 添加远程仓库，origin是远程仓库别名
git remote add origin https://github.com/bingoteng/Techmd.git
# 查看已绑定的远程仓库
git remote -v
# 更换远程仓库地址
git remote set-url origin 新仓库地址
# 删除旧远程绑定
git remote remove origin
//标准合并（保留远程README，适合多人协作）
git pull origin main --allow-unrelated-histories
git push -u origin main
```

`--allow-unrelated-histories`允许合并两套互不关联的提交历史；若出现冲突，编辑冲突文件后重新`git add .`、`git commit`，再推送。 `-u`作用：绑定本地main和远程origin/main分支，后续直接输入`git push`即可推送。 2. 个人私有仓库强制覆盖（舍弃远程初始README）

## 四、切换SSH协议连接仓库、克隆测试（第四步）

### 1. 生成SSH密钥

```
ssh-keygen -t ed25519 -C "你的Git绑定邮箱"
```

连续三次回车，不设置密钥密码；

- 公钥路径：`C:\用户目录\.ssh\id_ed25519.pub`，需要复制全部内容上传GitHub；
- 私钥路径：`id_ed25519`，本地妥善保存，禁止外泄。 GitHub网页操作：头像→Settings→SSH and GPG keys→New SSH key，粘贴公钥保存。

### 2. 测试SSH连通性

```
ssh -T git@github.com
```

首次连接会询问是否确认主机指纹，输入`yes`回车即可；输出欢迎字样代表密钥配置成功。

### 3. SSH克隆仓库

```
git clone git@github.com:bingoteng/Techmd.git
```

### SSH典型报错：`Permission denied (publickey)`

原因：公钥未上传GitHub、复制密钥时遗漏/多复制字符、误上传私钥；解决：重新复制`.pub`公钥添加到GitHub，再测试连通。

## 五、关键事故处理：误删`.git`文件夹

### 现象

手动删除目录内`.git`后，所有`git`命令失效，本地版本历史、远程绑定全部丢失；但`typora`、`README.md`这类原始文件不会被删除。

### 两种修复方案

1. 优先方案：回收站还原`.git`文件夹，仓库所有历史、远程配置直接恢复；
2. 重新初始化仓库：

```
git init
git remote add origin git@github.com:bingoteng/Techmd.git
git add .
git commit -m "重新初始化仓库"
git push -u origin main -f
```

## 六、补充：公开仓库权限常识

1. 其他用户可以克隆你的公开仓库、在本地任意修改文件，但**没有权限直接推送改动到你的远程仓库**；
2. 他人想要提交修改，只能Fork复刻你的仓库，写完代码发起Pull Request，由你审核决定是否合并；
3. 只有仓库所有者、你主动添加的协作者，才拥有直接推送代码的权限，不要随意给陌生人开放协作者权限。

## 七、常用辅助命令汇总

```bash
# 查看提交历史
git log
# 取消Git全局代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

# Git一次性完整笔记工作流（前置全局配置已完成，有序列表版）

> 前提：电脑已完成user.name/user.email、换行、中文路径、长路径等全局配置，无需重复配置

1. 进入存放笔记的本地根目录
   
   ```
   cd D:/Typora_notes/Notes
   ```

2. 初始化本地Git仓库（仅第一次新建文件夹执行）
   
   ```
   git init
   ```

3. 创建统一收纳文件夹，收拢所有零散笔记目录
   
   ```
   mkdir typora
   git mv A* B* C* D* E* X* Y* Z* typora/
   ```

4. 关联GitHub远程仓库（仅首次绑定执行）
   
   ```
   git remote add origin git@github.com:bingoteng/Techmd.git
   # 校验远程是否绑定成功
   git remote -v
   ```

5. 抓取目录下全部文件加入暂存区
   
   ```
   git add .
   ```

6. 提交本次文件变更，填写备注
   
   ```
   git commit -m "整理嵌入式全套笔记，统一归入typora目录"
   ```

7. 拉取远程仓库内容，合并两边无关提交历史（仅首次推送执行）
   
   ```
   git pull origin main --allow-unrelated-histories
   ```

8. 首次推送并绑定本地与远程分支关联
   
   ```
   git push -u origin main
   ```

## 后续日常更新笔记极简循环（前置全部配置完成后，每次修改笔记只用这4步）

1. 修改本地md笔记文件

2. 缓存变更文件
   
   ```
   git add .
   ```

3. 提交更新记录
   
   ```
   git commit -m "更新XX八股文/项目文档知识点"
   ```

4. 同步到远程GitHub仓库
   
   ```
   git push
   ```

## 配套踩坑应急操作（工作流异常时使用）

1. HTTPS推送报连接重置：切换SSH远程地址
   
   ```
   git remote set-url origin git@github.com:bingoteng/Techmd.git
   ```

2. 推送提示rejected、需要先拉取
   
   ```
   git pull origin main
   git push
   ```

3. 误删.git文件夹，重建仓库上传
   
   ```
   git init
   git remote add origin git@github.com:bingoteng/Techmd.git
   git add .
   git commit -m "重建笔记仓库"
   git push -u origin main -f
   ```

# 