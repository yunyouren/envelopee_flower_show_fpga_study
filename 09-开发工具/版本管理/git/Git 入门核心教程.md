

> 本笔记内容主要参考自 [廖雪峰的Git教程](https://liaoxuefeng.com/books/git/time-travel/index.html)
> [保姆级Git入门指南！看完就会！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1TwtVzqEE2/?vd_source=8b7cf57677b18dd95bbdb43cfd758758)（这个非常好用）
[Git工作流程介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1T3ghzkEGR)
Git 是一个非常强大的**分布式版本控制系统**，是现代软件开发的基石。本指南将涵盖其最核心的概念和命令，帮助您快速上手。

---

还是直接用插件好用> [保姆级Git入门指南！看完就会！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1TwtVzqEE2/?vd_source=8b7cf57677b18dd95bbdb43cfd758758)（这个非常好用）
[在 VS Code 中使用 Git 源代码控制 --- Using Git source control in VS Code](https://code.visualstudio.com/docs/sourcecontrol/overview)
[【VSCode ☆ Git 】最佳代码管理 ➔ 高效且优雅_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1w14y1C7oi/?vd_source=8b7cf57677b18dd95bbdb43cfd758758&spm_id_from=333.788.videopod.sections)

一定要写commit，不然会出问题

## 一、 初始设置

### 1. Git 是什么？

简单来说，Git 就像一个可以给您的代码项目“存档”和“读档”的超级工具。它能：

*   **追踪文件的每一次修改**：您可以随时回溯到任何一个历史版本。
*   **支持多人协作**：方便您和团队成员在同一个项目上工作，而不会互相干扰。
*   **创建“平行宇宙”（分支）**：您可以安全地在新功能分支上进行实验，成功后再合并回主线。

### 2. 首次配置（全局只需一次）

在使用Git前，您需要告诉它您的身份。这会记录在您的每一次“存档”（提交）中。

```bash
# 设置用户名
git config --global user.name "Your Name"

# 设置用户邮箱
git config --global user.email "youremail@example.com"
```

---

## 二、 本地仓库操作

### 1. 创建仓库

有两种方式开始使用Git：

*   **初始化新仓库**：如果您有一个尚未进行版本控制的项目文件夹，可以进入该文件夹并运行：
    ```bash
    cd path/to/your/project
    git init
    ```
    > 这会在当前目录下创建一个名为 `.git` 的隐藏子目录，您的所有版本历史都将存放在这里。

*   **克隆现有仓库**：如果您想从一个远程服务器（如 GitHub, GitLab）上获取一个现有项目，使用 `git clone`：
    ```bash
    git clone https://github.com/example/project.git
    ```

### 2. 核心工作流程：修改 -> 暂存 -> 提交

这是您日常使用最频繁的流程，请务必理解**工作区**、**暂存区**和**版本库**的概念。一个完整的提交流程如下：

1.  **`git status`：查看状态**
    随时使用此命令查看您项目当前的状态。Git会告诉您哪些文件被修改了，但还没有被暂存。
    ```bash
    git status
    ```

2.  **`git diff`：检查修改**
    如果想知道具体修改了什么，可以使用 `git diff` 命令。它会显示工作区文件与暂存区（或上一次提交）之间的差异。
    ```bash
    git diff rtl/DSP_PWM_autotest.v
    ```

3.  **`git add`：暂存更改**
    确认修改无误后，将您的更改放入“暂存区”，为提交做准备。
    ```bash
    # 暂存一个特定文件
    git add rtl/DSP_PWM_autotest.v
    
    # 暂存所有已修改和新增的文件
    git add .
    ```

4.  **`git commit`：提交更改**
    将暂存区的所有内容一次性提交到本地仓库中，形成一个新的版本。
    ```bash
    git commit -m "一个清晰的提交说明，例如：修复了时钟悬空导致的频率问题"
    ```

### 3. 版本回退（时光穿梭）

1.  **`git log`：查看历史**
    查看项目的所有提交历史记录，以便您知道想要回退到哪个版本。
    ```bash
    # 查看详细历史
    git log

    # 查看简化的单行历史，包含commit id
    git log --oneline --graph
    ```

2.  **`git reset`：回退版本**
    如果您想撤销某些提交，或者回到过去某个状态，可以使用 `git reset`。
    ```bash
    # 回退到上一个版本
    git reset --hard HEAD^

    # 回退到指定commit_id的版本
    git reset --hard <commit_id>
    ```
    > **警告**：`--hard` 参数会彻底丢弃工作区和暂存区的修改，请在操作前确认所有重要更改都已保存或提交。

---

## 三、 分支与协作

### 1. 分支管理

分支是Git的“杀手级”功能。它允许您从主线（通常是 `master` 或 `main`）分离出去，进行开发和实验，完成后再安全地合并回来。

```bash
# 查看所有分支
git branch

# 创建一个新分支
git branch feature/new-pwm-logic

# 切换到新分支上工作
git checkout feature/new-pwm-logic

# --- 在新分支上进行代码修改、add和commit ---

# 当新功能完成后，切换回主分支
git checkout main

# 将新分支的修改合并到主分支
git merge feature/new-pwm-logic
```

### 2. 与远程仓库协作

当您想与团队分享您的代码，或者将本地代码备份到远程服务器时，就需要用到以下命令。

1.  **`git remote`：关联远程仓库**
    如果您的仓库是本地 `init` 的，需要先告诉它远程仓库的地址。`origin` 是远程仓库的默认别名。
    ```bash
    git remote add origin https://github.com/your-username/your-repo.git
    ```

2.  **`git push`：推送更改**
    将您本地的提交推送到远程仓库。
    ```bash
    # 将本地的main分支推送到origin远程的main分支
    git push -u origin main
    ```
    > `-u` 选项只需在第一次推送时使用，它会建立本地分支与远程分支的关联。

3.  **`git pull`：拉取更改**
    从远程仓库获取最新的更改，并与您的本地分支合并。这是与他人协作时保持代码同步的关键。
    ```bash
    git pull origin main
    ```

---

这个入门指南涵盖了Git 80%的日常使用场景。希望它能帮助您顺利上手！





# [解决Git Push至GitHub还是很慢或报错的问题](https://www.cnblogs.com/alphainf/p/17150558.html "发布于 2023-02-24 11:00")

## 问题描述

从本地提交代码到 GitHub 远程仓库，由于 DNS 污染的问题，国内提交速度很慢，有时候还报错。笔者自己花钱买了一个梯子，但开启梯子的代理后仍然没有解决问题，不过 Google 等倒是可以访问了。

### 原因分析

虽然开启了代理，但可能 git push 并没有走代理，因为需要在 git 里面进行配置。

### 解决方法

配置 git push 直接走网络代理

git config --global http.proxy socks5://127.0.0.1:1080
git config --global https.proxy socks5://127.0.0.1:1080  

其中 1080 是 SOCKS 代理的端口，一般默认 1080，可以在代理工具的设置中查看。

下面以Clash for windows为例，进行代理IP和PORT的配置和查询

PORT可以在这里查询

![](https://img2023.cnblogs.com/blog/1260344/202302/1260344-20230224105912758-1924618478.png)

IP可以在这里配置和查询

![](https://img2023.cnblogs.com/blog/1260344/202302/1260344-20230224105945512-1905894000.png)

然后就可以流畅push了！