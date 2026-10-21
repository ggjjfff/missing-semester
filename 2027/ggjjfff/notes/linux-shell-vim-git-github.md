# Lesson 1 Notes: Linux, Shell & Vim, Git & GitHub

> 课程：27 赛季控制视觉招新培训第一课
> 主题：从 Linux 到第一次 Pull Request
> 作者：`ggjjfff`

## 1. Linux 基础

### 1.1 内核与发行版

Linux 严格来说只是**内核**，我们日常安装使用的是**发行版**：

> 发行版 = Linux 内核 + 系统工具与软件包 + 默认配置

常见的发行版有 Ubuntu、Debian、Fedora、Arch Linux。团队统一使用 **Ubuntu**，原因是 ROS 兼容性好、开发机与车载主机环境一致。

### 1.2 机器人开发中的 Linux

典型的开发方式不是在机器人上写代码，而是通过网络远程登录：

```
开发电脑  ──SSH──>  机器人主机（Ubuntu Server）
                     ├── ROS 节点 / 控制程序
                     └── C++ 工具链
```

车载主机通常没有图形桌面，程序开机自启。这解释了为什么必须掌握 SSH 远程操作和终端内的文件编辑。

### 1.3 目录与路径

Linux 的目录树从根 `/` 开始，常用目录如下：

| 目录 | 用途 |
| --- | --- |
| `/home/user` | 用户文件，`~` 指向这里 |
| `/etc` | 系统配置 |
| `/usr` | 程序与库 |
| `/var` | 日志等可变数据 |
| `/dev` | 设备文件 |

三种路径写法：

- **绝对路径**：从 `/` 开始，如 `/home/user/example`
- **相对路径**：从当前位置开始，如 `./hello.txt`
- **家目录路径**：从 `~` 开始，如 `~/example`

三个特殊符号：`~` 家目录、`.` 当前目录、`..` 上一级目录。

> 注意：Linux **区分大小写**，`README.md` 与 `readme.md` 是两个不同的文件。

## 2. Shell 与终端

### 2.1 终端、Shell 与程序

| 层 | 角色 | 例子 |
| --- | --- | --- |
| Terminal | 输入输出界面 | 终端窗口 |
| Shell | 解释命令、展开路径 | Bash / Zsh |
| 程序 | 真正执行任务 | `ls`、`git` |

`cd` 属于 Shell 内建命令，不启动新进程，所以执行后没有输出是正常的。

### 2.2 命令的通用结构

```bash
# 命令  选项   参数
ls      -la    ~/projects
```

- `-l` 显示详细信息，`-a` 包含隐藏文件

### 2.3 常用命令

```bash
pwd                 # 显示当前目录
ls -la              # 列出全部内容
cd ~/linux_demo     # 进入目录
mkdir -p demo/src   # 递归创建目录
touch hello.txt     # 创建空文件
cp hello.txt a.txt  # 复制
mv a.txt b.txt      # 移动或重命名
rm b.txt            # 删除（不进回收站）
cat hello.txt       # 查看内容，less 分页查看
```

### 2.4 四种组合技巧

1. **变量**：`name="RM"` 然后 `echo "$name"`。等号两边**不能有空格**。
2. **重定向**：`echo "hello" > hello.txt` 覆盖写入，`>>` 追加写入。
3. **管道**：`ls -la | grep txt`，把前一个命令的输出交给下一个程序。
4. **条件连接**：`mkdir -p demo && cd demo`，前一条成功才执行后一条。

### 2.5 求助方式

| 方式 | 作用 |
| --- | --- |
| `ls --help` | 快速查看选项 |
| `man ls` | 打开手册，按 `q` 退出 |
| `Tab` | 补全命令或路径 |
| `↑` / `↓` | 调出历史命令 |

## 3. Vim

Vim 是**有模式的编辑器**，同一按键在不同模式下含义不同。

### 3.1 三种模式

| 模式 | 用途 | 进入方式 |
| --- | --- | --- |
| 普通模式 | 移动、删除、复制 | 打开时默认；按 `Esc` 返回 |
| 插入模式 | 输入文本 | 普通模式按 `i` |
| 命令行模式 | 保存、退出 | 普通模式按 `:` |

### 3.2 一次完整编辑

```text
vim hello.txt   # 打开文件
i               # 进入插入模式，输入内容
Esc             # 回到普通模式
:wq             # 保存并退出
```

### 3.3 常用按键

| 按键 | 作用 |
| --- | --- |
| `h` `j` `k` `l` | 左 / 下 / 上 / 右 |
| `dd` | 删除当前行 |
| `yy` / `p` | 复制当前行 / 粘贴 |
| `u` | 撤销 |
| `/word` | 搜索 word |

保存与退出：

- `:w` 保存
- `:q` 退出，有未保存修改会被阻止
- `:wq` 保存并退出
- `:q!` 放弃修改强制退出

> 卡住时先按 `Esc`，回到普通模式再决定下一步。

## 4. Git 与 GitHub

### 4.1 两者的分工

- **Git** 是本地版本控制工具：记录提交、比较修改、创建分支。
- **GitHub** 是远程托管与协作平台：托管仓库、通过 PR 评审、合并修改。

### 4.2 四个位置

| 位置 | 存放内容 | 如何进入 |
| --- | --- | --- |
| 工作区 | 正在编辑的文件 | 直接修改 |
| 暂存区 | 本次提交的清单 | `git add` |
| 本地仓库 | 提交历史 | `git commit` |
| 远程仓库 | 远程的提交历史 | `git push` |

`commit` 只写入本地，`push` 之后队友才能看到。

### 4.3 首次配置

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

这只是提交署名，**不等于登录 GitHub**。

### 4.4 SSH 密钥

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub      # 公钥，复制到 GitHub
ssh -T git@github.com          # 测试连接
```

- 公钥 `id_ed25519.pub` 添加到 GitHub 的 SSH 设置
- 私钥 `id_ed25519` **不上传、不截图、不分享**

### 4.5 常用命令

```bash
git clone <仓库地址>              # 下载仓库
git status                        # 查看状态
git switch -c feature/hello       # 创建并切换分支
git add hello.txt                 # 暂存
git commit -m "Add greeting."     # 提交到本地
git push -u origin feature/hello  # 推送并建立跟踪
git pull                          # 拉取远程更新
git log --oneline                 # 查看提交历史
```

### 4.6 Fork 协作流程

```text
团队仓库 --Fork--> 个人 Fork --Clone--> 本地
本地 --修改/commit--> 个人 Fork --PR--> 团队仓库
```

- **Fork** 是另一个仓库，**Branch** 是同一仓库内的开发线
- `origin` 通常指向自己的 Fork
- 评审要求修改时，继续向同一分支提交并推送，PR 会自动更新

## 5. 参考

- [Linux 命令行基础](https://missing.csail.mit.edu/2020/course-shell/)
- 课程 PPT：27 赛季控制视觉招新培训第一课
