
# Git 常用命令操作手册

> 适用环境：Ubuntu / Linux / ROS / Python / C++ 自动驾驶项目  
> 目标：掌握 Git 本地版本管理、远程仓库同步、分支管理、冲突处理和错误恢复。

---

# 一、Git 基础概念

## 1. Git 是什么？

Git 是一个分布式版本控制工具，可以记录代码每一次修改，方便：

- 保存代码历史版本
- 对比代码修改内容
- 恢复之前的代码
- 多人协同开发
- 使用分支开发不同功能
- 将本地代码同步到 GitHub、Gitee、GitLab

## 2. Git 四个核心区域

```text
工作区 Working Directory
       |
       | git add
       v
暂存区 Staging Area
       |
       | git commit
       v
本地仓库 Local Repository
       |
       | git push
       v
远程仓库 Remote Repository
```

| 区域 | 作用 |
|---|---|
| 工作区 | 当前正在编辑的代码目录 |
| 暂存区 | 准备提交的修改 |
| 本地仓库 | 已经提交保存的历史版本 |
| 远程仓库 | GitHub、Gitee、GitLab 上的仓库 |

## 3. 常用命令对应关系

| 命令 | 作用 |
|---|---|
| `git status` | 查看当前状态 |
| `git add` | 添加到暂存区 |
| `git commit` | 提交到本地仓库 |
| `git push` | 上传到远程仓库 |
| `git pull` | 拉取并整合远程修改 |
| `git fetch` | 获取远程更新，但不自动合并 |
| `git clone` | 克隆远程仓库 |
| `git branch` | 查看或管理分支 |
| `git switch` | 切换分支 |
| `git merge` | 合并分支 |
| `git log` | 查看提交历史 |
| `git diff` | 对比代码差异 |

---

# 二、Git 安装与初始配置

## 1. 安装 Git

```bash
sudo apt update
sudo apt install git -y
```

查看版本：

```bash
git --version
```

## 2. 配置用户名和邮箱

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

查看配置：

```bash
git config --global --list
```

查看单项配置：

```bash
git config --global user.name
git config --global user.email
```

## 3. 设置默认分支名称

```bash
git config --global init.defaultBranch main
```

## 4. 设置命令行输出颜色

```bash
git config --global color.ui auto
```

## 5. 设置常用 Git 别名

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm commit
git config --global alias.lg "log --oneline --graph --decorate --all"
```

例如：

```bash
git st
git lg
```

---

# 三、本地仓库操作

## 1. 初始化仓库

进入项目目录：

```bash
cd /home/ldd/mult_truck-main
```

初始化 Git：

```bash
git init
```

初始化后会生成隐藏目录：

```text
.git/
```

这个目录保存 Git 的版本管理信息。

注意：`git init` 不会自动将所有文件提交到 Git。

## 2. 查看仓库状态

```bash
git status
```

简洁模式：

```bash
git status -sb
```

## 3. 添加文件到暂存区

添加单个文件：

```bash
git add 文件名
```

例如：

```bash
git add src/decision/scripts/truck_decision.py
```

添加整个目录：

```bash
git add src/decision/
```

添加所有修改：

```bash
git add .
```

注意：`git add .` 会添加当前目录下符合 Git 跟踪规则的新增和修改文件，不会添加被 `.gitignore` 忽略的文件。

## 4. 提交到本地仓库

```bash
git commit -m "提交说明"
```

例如：

```bash
git commit -m "完成毫米波雷达障碍物检测"
```

查看提交：

```bash
git log
```

简洁查看：

```bash
git log --oneline
```

图形化查看：

```bash
git log --oneline --graph --decorate --all
```

## 5. 查看修改内容

查看尚未暂存的修改：

```bash
git diff
```

查看已经暂存的修改：

```bash
git diff --staged
```

比较两个提交：

```bash
git diff commit_id1 commit_id2
```

查看某次提交具体修改：

```bash
git show commit_id
```

## 6. 查看文件是否被 Git 跟踪

```bash
git ls-files
```

查看某个文件是否被忽略：

```bash
git check-ignore -v 文件路径
```

---

# 四、远程仓库操作

## 1. 创建远程仓库

常见平台：

- GitHub
- Gitee
- GitLab

在平台创建仓库后，复制仓库地址。

例如：

```text
https://gitee.com/用户名/项目名.git
```

或者 SSH 地址：

```text
git@gitee.com:用户名/项目名.git
```

## 2. 添加远程仓库

```bash
git remote add origin 远程仓库地址
```

例如：

```bash
git remote add origin git@gitee.com:用户名/mult_truck-main.git
```

其中：

- `origin`：远程仓库的默认名称，可以理解为远程仓库的别名。
- 后面是远程仓库地址。

查看远程仓库：

```bash
git remote -v
```

## 3. 修改远程仓库地址

```bash
git remote set-url origin 新仓库地址
```

删除远程仓库关联：

```bash
git remote remove origin
```

注意：删除远程关联不会删除远程服务器上的仓库。

## 4. 第一次推送代码

```bash
git push -u origin main
```

其中：

- `push`：上传本地提交。
- `origin`：远程仓库名称。
- `main`：本地分支名称。
- `-u`：建立本地分支和远程分支的跟踪关系。

建立跟踪关系后，后续可以直接：

```bash
git push
```

## 5. 克隆远程仓库

```bash
git clone 远程仓库地址
```

例如：

```bash
git clone git@gitee.com:用户名/mult_truck-main.git
```

克隆后进入项目：

```bash
cd mult_truck-main
```

## 6. 拉取远程更新

```bash
git pull
```

指定远程仓库和分支：

```bash
git pull origin main
```

`git pull` 本质上相当于：

```bash
git fetch
git merge
```

## 7. 获取远程更新但不合并

```bash
git fetch
```

或者：

```bash
git fetch origin
```

查看远程分支：

```bash
git branch -r
```

比较本地和远程分支：

```bash
git log --oneline main..origin/main
```

查看远程更新后，再决定是否合并：

```bash
git merge origin/main
```

## 8. 推送代码

```bash
git push
```

指定远程仓库和分支：

```bash
git push origin main
```

推送新分支：

```bash
git push -u origin feature/radar
```

---

# 五、Git 分支管理

## 1. 什么是分支？

分支可以理解为一条独立的代码开发线路。

例如：

```text
                    feature/radar
                   /
main -------------+----------------
                   \
                    feature/mqtt
```

不同分支可以开发不同功能，不会直接影响主分支。

## 2. 查看分支

查看本地分支：

```bash
git branch
```

查看远程分支：

```bash
git branch -r
```

查看所有分支：

```bash
git branch -a
```

## 3. 创建分支

```bash
git branch feature/radar
```

创建并切换：

```bash
git switch -c feature/radar
```

旧版常用写法：

```bash
git checkout -b feature/radar
```

## 4. 切换分支

```bash
git switch main
```

切换到开发分支：

```bash
git switch dev
```

旧版写法：

```bash
git checkout main
```

## 5. 从远程分支创建本地分支

先获取远程分支信息：

```bash
git fetch origin
```

查看远程分支：

```bash
git branch -r
```

创建本地分支并跟踪远程分支：

```bash
git switch -c feature/mqtt --track origin/feature/mqtt
```

## 6. 合并分支

例如，将 `feature/radar` 合并到 `main`：

```bash
git switch main
git merge feature/radar
```

推送合并后的主分支：

```bash
git push origin main
```

注意：合并操作在哪个分支执行，目标就是当前所在分支。

## 7. 删除本地分支

删除已经合并的分支：

```bash
git branch -d feature/radar
```

强制删除：

```bash
git branch -D feature/radar
```

注意：

- `-d`：安全删除，未合并时通常会拒绝。
- `-D`：强制删除，可能丢失该分支尚未合并的提交。
- 不能删除当前所在分支，需要先切换到其他分支。

## 8. 删除远程分支

```bash
git push origin --delete feature/radar
```

清理本地已经失效的远程分支记录：

```bash
git fetch --prune
```

## 9. 查看分支独有提交

查看 feature/radar 中有、main 中没有的提交：

```bash
git log main..feature/radar --oneline
```

查看 main 中有、feature/radar 中没有的提交：

```bash
git log feature/radar..main --oneline
```

---

# 六、分支管理实战流程

推荐自动驾驶项目采用以下分支结构：

```text
main
  |
  | 稳定版本
  |
dev
  |
  | 日常集成开发
  |
  |------ feature/radar
  |
  |------ feature/navigation
  |
  |------ feature/mqtt
  |
  |------ fix/planning-bug
```

分支作用：

| 分支 | 作用 |
|---|---|
| main | 稳定版本，尽量不直接开发 |
| dev | 日常集成开发分支 |
| feature/radar | 雷达功能开发 |
| feature/navigation | 导航功能开发 |
| feature/mqtt | MQTT 功能开发 |
| fix/planning-bug | 规划模块问题修复 |

## 1. 创建 dev 分支

```bash
git switch main
git switch -c dev
git push -u origin dev
```

## 2. 开发雷达功能

```bash
git switch dev
git switch -c feature/radar
```

开发代码后：

```bash
git status
git add .
git commit -m "完成毫米波雷达障碍物检测"
git push -u origin feature/radar
```

## 3. 将雷达功能合并到 dev

```bash
git switch dev
git pull origin dev
git merge feature/radar
```

推送：

```bash
git push origin dev
```

## 4. 删除已经完成的功能分支

```bash
git branch -d feature/radar
git push origin --delete feature/radar
```

注意：只有确认功能已经合并、没有需要保留的独有提交后，才删除分支。

---

# 七、Git 冲突处理

## 1. 什么是冲突？

例如：

- 你在本地修改了 `truck_decision.py`。
- 同事在远程仓库也修改了同一个文件。
- 双方修改的内容无法自动合并。

此时 Git 可能提示：

```text
CONFLICT (content): Merge conflict
```

## 2. 查看冲突文件

```bash
git status
```

## 3. 打开冲突文件

冲突内容可能类似：

```python
self.speed = 1.0
```

含义：

- `<<<<<<< HEAD`：当前分支的内容。
- `=======`：冲突分隔线。
- `>>>>>>> origin/main`：被合并进来的分支内容。

手动修改成最终需要的代码，例如：

```python
self.speed = 1.5
```

删除冲突标记。

## 4. 标记冲突已解决

```bash
git add 冲突文件
```

## 5. 完成合并

```bash
git commit
```

## 6. 放弃本次合并

如果不想继续合并：

```bash
git merge --abort
```

注意：执行前确认自己的未提交修改已经妥善保存。

---

# 八、撤销与恢复操作

这是 Git 中非常重要的一部分，使用前一定要明确自己要撤销什么。

## 1. 撤销工作区修改

撤销某个文件尚未暂存的修改：

```bash
git restore 文件名
```

例如：

```bash
git restore src/decision/scripts/truck_decision.py
```

撤销当前目录下所有尚未暂存的修改：

```bash
git restore .
```

注意：这会丢弃对应的未提交修改。

## 2. 从暂存区撤回文件

已经执行：

```bash
git add .
```

但是还没有 commit，想把文件从暂存区撤回来：

```bash
git restore --staged 文件名
```

全部撤回：

```bash
git restore --staged .
```

注意：这不会删除工作区中的修改。

## 3. 修改最后一次提交说明

```bash
git commit --amend -m "新的提交说明"
```

如果已经推送过这次提交，修改历史后需要谨慎处理远程同步。

## 4. 撤销最后一次提交，但保留修改

保留在暂存区：

```bash
git reset --soft HEAD~1
```

保留在工作区：

```bash
git reset --mixed HEAD~1
```

默认就是 mixed：

```bash
git reset HEAD~1
```

## 5. 强制撤销最后一次提交

```bash
git reset --hard HEAD~1
```

危险操作！

会同时重置提交历史、暂存区和工作区，可能导致未保存的代码修改丢失。

## 6. 使用 revert 撤销提交

```bash
git revert HEAD
```

作用：

- 不直接删除原提交。
- 新建一个反向提交。
- 适合已经推送到远程、其他人可能已经使用的提交。

指定提交：

```bash
git revert commit_id
```

## 7. 查看历史操作记录

```bash
git reflog
```

查看最近操作：

```bash
git reflog --oneline
```

可以通过 reflog 找到之前的提交位置，再决定是否恢复。

例如：

```bash
git reset --hard commit_id
```

执行前先确认需要恢复的提交，避免覆盖当前工作。

---

# 九、Git 暂存操作 stash

## 1. 什么是 stash？

当你正在开发一个功能，突然需要切换到其他分支修复问题，但当前代码还不想提交，就可以使用 stash 暂时保存修改。

## 2. 保存当前修改

```bash
git stash
```

带说明保存：

```bash
git stash push -m "暂存雷达功能开发"
```

包含未跟踪的新文件：

```bash
git stash push -u -m "暂存当前全部修改"
```

## 3. 查看暂存记录

```bash
git stash list
```

## 4. 恢复最近一次暂存

```bash
git stash pop
```

恢复后删除对应 stash 记录。

## 5. 恢复但不删除暂存记录

```bash
git stash apply
```

指定某条记录：

```bash
git stash apply stash@{0}
```

## 6. 删除暂存记录

```bash
git stash drop stash@{0}
```

清空全部暂存：

```bash
git stash clear
```

注意：清空后无法通过普通 stash 操作恢复。

---

# 十、.gitignore 文件配置

## 1. .gitignore 是什么？

`.gitignore` 用来告诉 Git 哪些文件或目录不需要纳入版本管理。

自动驾驶 ROS 项目中，通常不需要提交编译产物、日志和缓存文件。

## 2. 创建 .gitignore

```bash
touch .gitignore
```

## 3. ROS 项目推荐配置

```gitignore
# ROS 编译目录
build/
devel/
install/
logs/

# Python 缓存
__pycache__/
*.pyc
*.pyo

# C++ 编译产物
*.o
*.a
*.so
*.out

# IDE 配置
.idea/
.vscode/

# 系统文件
.DS_Store
Thumbs.db

# 日志
*.log

# 临时文件
*.tmp
*.bak
*.swp

# 虚拟环境
venv/
.venv/

# 数据集和大型训练文件（按项目实际情况决定）
# dataset/
# weights/
# *.pt
# *.pth
# *.onnx
```

注意：

- `.gitignore` 只对尚未被 Git 跟踪的文件生效。
- 已经提交过的文件不会因为添加忽略规则而自动停止跟踪。

## 4. 让 Git 停止跟踪已经提交的目录

例如停止跟踪 `build/`：

```bash
git rm -r --cached build/
```

停止跟踪多个目录：

```bash
git rm -r --cached build/ devel/ logs/
```

然后：

```bash
git add .gitignore
git commit -m "更新gitignore规则"
```

注意：`--cached` 只从 Git 索引中移除，不会删除本地文件。

---

# 十一、SSH 密钥配置

## 1. 为什么使用 SSH？

使用 SSH 可以避免每次推送代码都输入账号密码或访问令牌。

## 2. 生成 SSH 密钥

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
```

一路按 Enter 即可使用默认路径。

默认生成：

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

## 3. 查看公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

复制输出内容，添加到 GitHub 或 Gitee 的 SSH 公钥设置中。

注意：只上传 `.pub` 公钥，不要泄露没有 `.pub` 后缀的私钥。

## 4. 测试 SSH 连接

Gitee：

```bash
ssh -T git@gitee.com
```

GitHub：

```bash
ssh -T git@github.com
```

## 5. 将远程地址改为 SSH

```bash
git remote set-url origin git@gitee.com:用户名/项目名.git
```

查看：

```bash
git remote -v
```

---

# 十二、标签 Tag 操作

Tag 通常用于标记重要版本。

例如：

```text
v1.0.0
v1.1.0
v2.0.0
```

## 1. 创建标签

```bash
git tag v1.0.0
```

创建带说明的标签：

```bash
git tag -a v1.0.0 -m "第一个稳定版本"
```

## 2. 查看标签

```bash
git tag
```

## 3. 推送标签

```bash
git push origin v1.0.0
```

推送全部标签：

```bash
git push origin --tags
```

## 4. 删除标签

删除本地标签：

```bash
git tag -d v1.0.0
```

删除远程标签：

```bash
git push origin --delete v1.0.0
```

---

# 十三、其他实用命令

## 1. 查看某个文件的修改历史

```bash
git log -- 文件名
```

## 2. 查看某行代码是谁修改的

```bash
git blame 文件名
```

例如：

```bash
git blame src/decision/scripts/truck_decision.py
```

## 3. 查看提交详情

```bash
git show
```

## 4. 比较两个分支的差异

```bash
git diff main dev
```

只查看文件名称：

```bash
git diff --name-only main dev
```

## 5. 从其他分支拿一个提交

```bash
git cherry-pick commit_id
```

适合只需要其他分支某一个修复提交的情况。

## 6. 清理未跟踪文件

先预览：

```bash
git clean -nfd
```

确认后删除：

```bash
git clean -fd
```

危险：会删除未被 Git 跟踪的文件和目录，执行前一定检查预览结果。

## 7. 查看 Git 仓库根目录

```bash
git rev-parse --show-toplevel
```

## 8. 查看当前分支

```bash
git branch --show-current
```

---

# 十四、日常开发最常用工作流

## 1. 每天开始开发

```bash
cd /home/ldd/mult_truck-main

# 查看当前分支
git branch --show-current

# 查看工作区状态
git status

# 获取远程最新代码
git pull
```

## 2. 开发新功能

```bash
# 切换到开发分支
git switch dev

# 创建功能分支
git switch -c feature/new-function
```

## 3. 开发过程中提交代码

```bash
# 查看修改
git status

# 添加修改
git add .

# 查看暂存内容
git diff --staged

# 提交
git commit -m "完成新功能开发"
```

## 4. 推送到远程仓库

```bash
git push -u origin feature/new-function
```

第一次推送新分支使用 `-u`，以后通常可以直接：

```bash
git push
```

## 5. 功能完成后合并

```bash
# 切换到 dev
git switch dev

# 更新远程 dev
git pull origin dev

# 合并功能分支
git merge feature/new-function

# 推送
git push origin dev
```

## 6. 删除功能分支

```bash
git branch -d feature/new-function

git push origin --delete feature/new-function
```

---

# 十五、Git 常见问题及处理

## 1. 提示 fatal: not a git repository

原因：当前目录不是 Git 仓库，或者没有进入项目目录。

检查：

```bash
pwd
ls -la
```

进入正确目录：

```bash
cd /home/ldd/mult_truck-main
```

如果这是一个全新项目，可以初始化：

```bash
git init
```

## 2. 提示 nothing to commit

原因：当前没有可提交的修改。

检查：

```bash
git status
```

## 3. 提示 rejected: non-fast-forward

原因：远程仓库存在本地没有的提交。

通常先执行：

```bash
git pull --rebase origin main
```

然后：

```bash
git push origin main
```

如果出现冲突，先解决冲突，再继续：

```bash
git add 冲突文件
git rebase --continue
```

放弃本次 rebase：

```bash
git rebase --abort
```

注意：如果远程仓库和本地仓库有不同的初始提交，需要先确认双方历史，再决定合并方式，不要直接强制推送。

## 4. 提示 branch is not fully merged

原因：要删除的分支还有未合并提交。

先查看：

```bash
git log main..feature/radar --oneline
```

确认不需要这些提交后，才考虑：

```bash
git branch -D feature/radar
```

## 5. 切换分支失败

先检查：

```bash
git status
```

如果有未提交修改，可以选择：

提交保存：

```bash
git add .
git commit -m "保存当前修改"
```

或者暂存：

```bash
git stash push -u -m "切换分支前暂存"
```

再切换：

```bash
git switch main
```

---

# 十六、Git 命令速查表

| 需求 | 命令 |
|---|---|
| 初始化仓库 | `git init` |
| 克隆仓库 | `git clone URL` |
| 查看状态 | `git status` |
| 添加文件 | `git add 文件名` |
| 添加全部修改 | `git add .` |
| 提交修改 | `git commit -m "说明"` |
| 查看提交历史 | `git log --oneline` |
| 查看代码差异 | `git diff` |
| 查看暂存差异 | `git diff --staged` |
| 推送代码 | `git push` |
| 拉取代码 | `git pull` |
| 获取远程更新 | `git fetch` |
| 查看远程地址 | `git remote -v` |
| 添加远程仓库 | `git remote add origin URL` |
| 查看本地分支 | `git branch` |
| 查看所有分支 | `git branch -a` |
| 创建分支 | `git branch 分支名` |
| 创建并切换 | `git switch -c 分支名` |
| 切换分支 | `git switch 分支名` |
| 合并分支 | `git merge 分支名` |
| 删除本地分支 | `git branch -d 分支名` |
| 强制删除分支 | `git branch -D 分支名` |
| 删除远程分支 | `git push origin --delete 分支名` |
| 暂存修改 | `git stash` |
| 恢复暂存 | `git stash pop` |
| 撤销工作区修改 | `git restore 文件名` |
| 撤销暂存 | `git restore --staged 文件名` |
| 撤销最近提交但保留修改 | `git reset --soft HEAD~1` |
| 撤销提交并保留工作区修改 | `git reset --mixed HEAD~1` |
| 强制回退提交 | `git reset --hard HEAD~1` |
| 安全撤销已提交修改 | `git revert HEAD` |
| 查看历史操作 | `git reflog` |
| 创建版本标签 | `git tag v1.0.0` |
| 查看当前分支 | `git branch --show-current` |

---

# 十七、最重要的几个注意事项

1. `git add` 不等于提交，只是将修改放入暂存区。
2. `git commit` 只提交到本地仓库，不会自动上传远程。
3. `git push` 才是将本地提交上传到远程仓库。
4. `git pull` 会获取远程代码并整合到当前分支，执行前先确认当前分支。
5. 切换分支前先执行 `git status`，避免未提交代码带来问题。
6. 删除分支不等于删除已合并进其他分支的代码。
7. `git reset --hard`、`git clean -fd`、`git branch -D` 都要谨慎使用。
8. 不要随意使用 `git push --force`，尤其是多人协作项目。
9. ROS 项目通常不需要提交 `build/`、`devel/`、`install/` 等编译产物。
10. 重要功能完成后及时 commit，养成小步提交的习惯。

## 推荐记住的日常命令组合

```bash
# 查看状态
git status

# 拉取最新代码
git pull

# 查看分支
git branch

# 创建并切换功能分支
git switch -c feature/radar

# 添加修改
git add .

# 提交修改
git commit -m "完成雷达功能开发"

# 推送代码
git push -u origin feature/radar

# 切换开发分支
git switch dev

# 合并功能分支
git merge feature/radar

# 推送开发分支
git push origin dev
```

**核心记忆：**

```text
git status   查看状态
git add      添加修改
git commit   保存版本
git push     上传代码
git pull     同步远程
git switch   切换分支
git merge    合并分支
git log      查看历史
git restore  撤销修改
git stash    暂存修改
```
