# 一、 介绍

Git是目前世界上最先进的分布式版本控制系统。

# 二、 安装

## 1. Windows上安装

**下载地址**：https://git-scm.com/download/win

![1644288011107](https://images.cnblogs.com/cnblogs_com/upstudy/2101984/o_220208035924_1644288011107.png)

下载好后，双击进行下一步下一步即可。

**验证git安装是否成功**

1). 命令行输入：`git --version`，提示如下标识安装成功

![1644290405186](https://images.cnblogs.com/cnblogs_com/upstudy/2101984/o_220208040007_1644290405186.png)

2). 回到电脑桌面，鼠标右击如果看到有两个git单词则安装成功

![1644289587766](https://images.cnblogs.com/cnblogs_com/upstudy/2101984/o_220208040040_1644289587766.png)

## 2. CentOS 9 Stream上安装

**更新系统**
首先，确保您的系统是最新的：

```
sudo dnf update
```

**安装 Git**
使用以下命令安装 Git：

```
sudo dnf install git
```

**验证安装**
安装完成后，您可以使用以下命令检查 Git 是否已成功安装，并查看版本：

```
git --version
```

# 三、 配置

> Windows中可以右键打开git bash进行git指令的执行

## 1. 个人信息配置

### 1.1 查看所有配置参数

```
git config --list
```

### 1.2 设置用户名

```
git config --global user.name "<用户名>"
```

### 1.3 设置用户邮箱

```
git config --global user.email "<电子邮件>"
```



## 2. SSH登录配置

配置ssh登录，不需要账号密码（使用gitee的账户名）

### 2.1 创建SSH连接密钥

```
ssh-keygen -t rsa -C "xxx@xxx.com"
```

> **命令组成部分**
>
> 1. **`ssh-keygen`**
>    这是一个用于生成、管理和转换 SSH 密钥的工具。SSH 密钥用于在服务器之间进行安全的无密码登录，是 SSH（Secure Shell）协议中的一种身份验证方式。
> 2. **`-C "xxx@xxx.com"`**
>    `-C` 参数用于为生成的密钥添加一个注释（Comment）。注释通常用于标识密钥的所有者或用途，方便后续管理和识别。
> 3. **`-t rsa`**（可选）
>    指定生成的密钥类型为 RSA（Rivest–Shamir–Adleman）。RSA 是一种常用的非对称加密算法，广泛用于 SSH 密钥生成。它提供了较高的安全性，适合大多数场景。
>
> **执行过程**
>
> 当你运行这个命令时，`ssh-keygen` 会执行以下操作：
>
> 1. **生成密钥对**
>    - 生成一对密钥，包括一个**公钥**（public key）和一个**私钥**（private key）。公钥可以公开分发，而私钥必须严格保密。
>    - 默认情况下，生成的密钥对会保存在用户的主目录下的 `.ssh` 文件夹中。公钥文件名为 `id_rsa.pub`，私钥文件名为 `id_rsa`。
> 2. **提示输入保存路径（可选）**
>    - 如果你不指定保存路径，`ssh-keygen` 会默认将密钥保存在 `~/.ssh/id_rsa` 和 `~/.ssh/id_rsa.pub`。
>    - 你可以通过提示输入自定义的保存路径，例如 `~/.ssh/my_custom_key`。
> 3. **提示输入密码（可选）**
>    - 为了增加安全性，`ssh-keygen` 会提示你输入一个密码（passphrase）。这个密码用于加密私钥文件，防止私钥被非法使用。
>    - 如果你不想设置密码，可以直接按回车跳过。
>
> **使用场景**
>
> 生成的密钥对通常用于以下场景：
>
> 1. **SSH 无密码登录**
>    - 将公钥添加到远程服务器的 `~/.ssh/authorized_keys` 文件中。之后，当你使用 SSH 连接到该服务器时，SSH 客户端会使用你的私钥进行身份验证，而无需输入密码。
> 2. **Git 仓库访问**
>    - 对于使用 SSH 协议访问 Git 仓库（如 GitHub、Gitee 等），你需要将公钥添加到 Git 服务的账户设置中。这样，你就可以通过 SSH 方式安全地克隆、推送和拉取代码，而无需每次都输入用户名和密码。



### 2.2 查看公钥

```
cat ~/.ssh/id_rsa.pub
```



### 2.3 给git服务器配置公钥

登录gitee/github/gitlab，在平台的设置界面找到SSH设置选项，将上一步查看得到的公钥添加进去。
#### 2.3.1 gitee

#### 2.3.2 github

#### 2.3.3 gitlab
- 打开gitlab，找到Profile Settings ---> SSH Keys ---> Add SSH Key。
- 把上一步中复制的内容粘贴到Key所对应的文本框，在Title对应的文本框中给这个sshkey设置一个名字，点击Add key按钮

![gitlab_config_cvknsdauj](E:\各种资料\Java开发笔记\我的笔记\images\gitlab_config_cvknsdauj.png)



### 2.4 测试利用SSH连接git服务器

#### 2.4.1 连接gitee

```
ssh -T git@gitee.com
```

连接成功会出现类似如下信息：

```
Warning: Permanently added 'gitee.com' (ED25519) to the list of known hosts.
Hi ******* You've successfully authenticated, but GITEE.COM does not provide shell access.
```

#### 2.4.2 连接github

```
ssh -T git@github.com
```

连接成功会出现类似如下信息：

```
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
Hi Hojay-Chen! You've successfully authenticated, but GitHub does not provide shell access.
```



> 注意：若因为SSH的路径存在中文导致运行失败，需要迁移SSH的安装路径，如下是让Git使用另一个路径的SSH的步骤：
>
> **1. 确定默认.ssh位置**
>
> 在更改默认.ssh位置之前，首先需要确定当前的默认位置。为了做到这一点，可以使用Git bash命令行界面，并输入以下命令：
>
> ```bash
> echo $HOME
> ```
>
> 该命令会输出当前用户的主目录路径，例如：/c/Users/Your-Username。在该目录下，可以找到.ssh文件夹。
>
> **2. 创建新的.ssh文件夹**
>
> 在更改默认.ssh位置之前，需要在新的位置创建一个新的.ssh文件夹。打开Git bash命令行界面，并输入以下命令：
>
> ```bash
> mkdir D:/java-resourse/.ssh
> ```
>
> 其中，D:/java-resourse/是用来存储.ssh文件夹的完整路径。请确保具有足够的权限在该位置创建文件夹。
>
> **3. 复制已有的SSH密钥**
>
> 在新的.ssh文件夹中创建后，需要将已有的SSH密钥复制到新位置。在Git bash命令行界面中，输入以下命令：
>
> ```bash
> cp -R ~/.ssh/* D:/java-resourse/.ssh
> ```
>
> 这将递归复制当前用户主目录下的所有文件和文件夹到新的.ssh位置。
>
> **4. 更新SSH配置**
>
> 完成密钥的复制后，需要更新Git的SSH配置，以指定新的.ssh位置。在Git bash命令行界面中，输入以下命令：
>
> ```bash
> echo "export HOME=D:/java-resourse" > ~/.bashrc
> source ~/.bashrc
> ```
>
> 其中，D:/java-resourse/是之前创建新.ssh文件夹时使用的完整路径。
>
> 这将更新.bashrc文件，以设置新的主目录路径。然后，通过执行source命令，可以对当前会话应用更改，而不需要重新启动Git bash。
>
> **5. 配置系统环境变量**（可选）
>
> 如果完成如上配置后，依然无法正常连接，可能是因为Windows 系统的环境变量没有正确更新，导致 SSH 客户端仍然使用旧的路径，此时执行本步骤。
>
> 打开“控制面板” > “系统和安全” > “系统” > “高级系统设置” > “环境变量”。在“用户变量”中找到 `HOME` 变量，确认其值是否正确。如果不存在，可以手动添加一个 `HOME` 变量，值为 `D:/java-resourse`，其中 `D:/java-resourse`是刚刚用来存储.ssh的完整路径。
>
> **6. 验证更改**
>
> 可以重新启动Git bash，并使用之前的命令检查默认.ssh位置：
>
> ```bash
> echo $HOME
> ```
>
> 输出结果应该是之前设置的新位置的完整路径。此时，可以使用Git bash生成新的SSH密钥对，并将其存储在新的.ssh位置。这样，Git操作将自动使用新的.ssh文件夹。

#### 2.4.3 连接gitlab
```
ssh -T git@github.com
```

连接成功会出现类似如下信息：

```
Warning: Permanently added 'github.com' (ED25519) to the list of known hosts.
Hi Hojay-Chen! You've successfully authenticated, but GitHub does not provide shell access.
```



# 四、IDEA配置Git环境

## 1. 创建仓库

### 1.1 Gitee创建仓库

打开Gitee网站，点击个人主页，点击网站右上角如下图所示的加号，即可创建仓库。

![gitee_fasdnm](.\images\gitee_fasdnm.png)

## 2. 克隆仓库到IDEA

创建好仓库后，打开IDEA，点击如下图所示的功能：

<img src=".\images\git_IDEA_ankjnsac.png" alt="git_IDEA_ankjnsac" style="zoom:80%;" />

出现如下窗口，选择git，并在下方的URL中输入仓库的网址，即可将仓库克隆到IDEA作为一个maven项目：

<img src=".\images\git_IDEA_cniasni.png" alt="git_IDEA_cniasni" style="zoom: 80%;" />

## 3. 配置.gitignore文件

`.gitignore` 文件是 Git 仓库中的一个重要配置文件，用于指定哪些文件或目录应该被 Git 忽略，不纳入版本控制。通过 `.gitignore` 文件，可以避免将一些不必要的文件或目录提交到 Git 仓库中，从而保持仓库的整洁和高效。

在`.gitignore`中写入不需要版本控制的文件和文件夹的目录，然后可以在IDEA的commit模块（快捷键：alt+0）中将那些需要进行版本控制的文件选中，右键弹出选框并选择`Add to VCS`从而将这些文件纳入版本控制，如下图所示：

<img src=".\images\git_IDEA_vmkdna.png" alt="git_IDEA_vmkdna" style="zoom:80%;" />

## 4. 在IDEA配置git服务器

### 4.1 安装gitee插件

在“file -> setting -> Plugins”中下载并安装gitee插件。

### 4.2 在IDEA添加Gitee账号

安装好gitee插件后，打开“File -> Settings -> Version Control -> Gitee”添加自己的Gitee账号，注意IDEA只支持用邮箱登录，如果Gitee没有绑定邮箱需要先绑定邮箱，如下图所示：

<img src=".\images\git_IDEA_nvjsdn.png" alt="git_IDEA_nvjsdn" style="zoom:80%;" />

## 5. 在IDEA上传项目到Gitee

回到IDEA的Commit模块，将所有要提交的文件选中，然后点击Commit即可上传至本地仓库，点击Commit and Push即可上传至Gitee远程仓库。

<img src=".\images\git_IDEA_vmkdna.png" alt="git_IDEA_vmkdna" style="zoom:80%;" />

# 五、快速入门

## 1. 初始化git仓库

在项目的根目录下执行初始化git仓库的命令：

```
git init
```



## 2. 配置远程仓库

### 2.1 查看远程仓库配置

可以先查看当前本地仓库已经连接的远程仓库，通过如下指令实现，如果打印为空，则表示暂未配置任何远程仓库：

```
git remote -v
```

### 2.2 添加远程仓库配置

添加远程仓库链接的指令如下，其中“origin”为该远程仓库在本地仓库的别名，“https://github.com/...”为远程仓库的链接，可以使用https也可以使用ssh，若前面有进行ssh免密登录配置的话，使用ssh可以在推送的时候不需要再进行账号登录：

```
git remote add origin https://github.com/...
```

### 2.3 修改远程仓库配置

修改远程仓库链接的指令如下：

```
git remote set-url origin https://github.com/...
```

### 2.4 删除远程仓库配置

删除远程仓库链接的指令如下：

```
git remote remove origin
```



## 3. 推送

### 3.1 添加文件

要提交，首先得添加要提交的文件，使用 `git add` 命令添加文件进入 git 管理：

```
git add .  # 提交所有修改的和新建的数据暂存区

git add -u <==> git add –update  # 提交所有被删除和修改的文件到数据暂存区

git add -A <==>git add –all  # 提交所有被删除、被替换、被修改和新增的文件到数据暂存区
```

通常我们都使用“git add .”

### 3.2 查看状态

接下来，我们可以查看文件状态，看看文件是否被添加进了 git 管理：

```
git status
```

### 3.3 提交文件更改

刚才添加的文件只是到了 git 的暂存区，我们还需要提交添加的文件：

```bash
git commit -m "input yours message"
```

### 3.4 推送

使用 `push` 命令 将本地仓库推送到远程仓库（Github）中：

```
git push origin main
```

`git push origin main` 的作用是将本地 `main` 分支的更改推送到别名为`origin`的远程仓库的 `main` 分支，这里`origin`和`main`可以根据自己情况修改。具体步骤如下：

1. **检查本地分支状态**：

   Git 会检查本地 `main` 分支的状态，确保你已经提交了所有更改。如果还有未提交的更改，你需要先运行 `git add` 和 `git commit`。

2. **连接远程仓库**：

   Git 会尝试连接到远程仓库（`origin`）。连接方式取决于远程仓库的 URL，可以是 HTTPS 或 SSH。

3. **推送更改**：

   如果连接成功，Git 会将本地 `main` 分支的更改推送到远程仓库的 `main` 分支。如果远程分支上存在冲突或不一致，Git 会提示你解决这些问题。

## 4. 下载

使用如下指令，输入仓库的url即可下载。

```
git clone <url>
```



**常见问题**

1. **远程仓库包含未同步的更改**：

   如果远程仓库中有你本地没有的更改，Git 会拒绝推送。你需要先拉取远程仓库的最新更改：

   ```
   git pull origin main
   ```

   如果拉取过程中出现冲突，你需要手动解决冲突，然后再次提交。

1. **网络问题**：

   如果网络连接有问题，Git 会提示无法连接到远程仓库。你可以检查网络连接，或者配置代理。

2. **权限问题**：

   如果你没有权限推送更改到远程仓库，Git 会提示权限不足。你需要确保你有正确的权限，或者联系仓库的管理员。



# 六、使用

## 1. 分支管理

### 1.1 查看本地分支

查看当前仓库的所有本地分支：

```bash
git branch
```

输出示例：

```
* main
  develop
  feature/new-feature
  fix/bugfix
```

- `*` 表示当前所在的分支。



### 1.2 创建新分支

创建一个新的本地分支：

```bash
git branch <分支名称>
```

例如，创建一个名为 `feature/new-feature` 的分支：

```bash
git branch feature/new-feature
```

或者，创建并切换到新分支：

```bash
git checkout -b <分支名称>
```

例如：

```bash
git checkout -b feature/new-feature
```



### 1.3 切换分支

切换到已存在的分支：

```bash
git checkout <分支名称>
```

例如，切换到 `develop` 分支：

```bash
git checkout develop
```



### 1.4 删除本地分支

删除本地分支：

```bash
git branch -d <分支名称>
```

例如，删除 `feature/new-feature` 分支：

```bash
git branch -d feature/new-feature
```

- 如果分支尚未合并到当前分支，Git 会阻止删除。你可以使用 `-D` 强制删除：

  ```bash
  git branch -D feature/new-feature
  ```

  

### 1.5 重命名本地分支

重命名当前分支：

```bash
git branch -m <新分支名称>
```

例如，将当前分支重命名为 `feature/new-feature`：

```bash
git branch -m feature/new-feature
```

重命名其他分支：

```bash
git branch -m <旧分支名称> <新分支名称>
```

例如，将 `feature/old-feature` 重命名为 `feature/new-feature`：

```bash
git branch -m feature/old-feature feature/new-feature
```



### 1.6 查看分支状态

查看当前分支的状态：

```bash
git status
```

输出示例：

```
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```



### 1.7 查看分支提交历史

查看当前分支的提交历史：

```bash
git log
```

或者，查看特定分支的提交历史：

```bash
git log <分支名称>
```



### 1.8 合并分支

将一个分支的更改合并到当前分支：

```bash
git merge <分支名称>
```

例如，将 `feature/new-feature` 分支的更改合并到当前分支：

```bash
git merge feature/new-feature
```



### 1.9 拉取远程分支

从远程仓库拉取最新的分支信息：

```bash
git fetch
```

查看远程分支：

```bash
git branch -r
```

输出示例：

```
  origin/main
  origin/develop
  origin/feature/new-feature
```

拉取远程分支并创建本地分支：

```bash
git checkout -b <本地分支名称> <远程分支名称>
```

例如，拉取远程的 `feature/new-feature` 分支并创建本地分支：

```bash
git checkout -b feature/new-feature origin/feature/new-feature
```



### 1.10 推送本地分支

将本地分支推送到远程仓库：

```bash
git push -u origin <分支名称>
```

例如，将 `feature/new-feature` 分支推送到远程仓库：

```bash
git push -u origin feature/new-feature
```



### 1.11 查看分支差异

查看当前分支与另一个分支的差异：

```bash
git diff <分支名称>
```

例如，查看当前分支与 `main` 分支的差异：

```bash
git diff main
```



### 1.12 查看分支的上游分支

查看当前分支的上游分支：

```bash
git branch -vv
```

输出示例：

```
* main 1234567 [origin/main] Add new feature
  develop 7890abc [origin/develop] Fix bug
```



### 1.13 设置分支的上游分支

设置当前分支的上游分支：

```bash
git branch -u origin/<远程分支名称>
```

例如，设置当前分支的上游分支为 `origin/main`：

```bash
git branch -u origin/main
```



# 七、原理

## 1. Git 的三大工作区域

### 1.1 工作目录（Working Directory）

#### 1.1.1 概述

本地项目文件夹（你直接编辑文件的地方）。

#### 1.1.2 作用

开发者的“沙盒”，所有修改从这里开始。

#### 1.1.3 特性

- 文件状态分为：
  - `Untracked`
  - `Modified`（已修改，Git 已跟踪但未暂存）。
  - `Unmodified`（未修改，与最新提交一致）。
- **高风险区**：此处的改动可能被覆盖（如 `git checkout -- file`）。

#### 1.1.4 控制

- （未跟踪，新增文件）。

### 1.2 暂存区（Staging Area / Index）

#### 1.2.1 概述

位于 `.git/index` 的二进制文件，记录下一次提交的快照。

#### 1.2.2 作用

**精确控制提交内容**（如只提交部分文件或部分改动）。

#### 1.2.3 特性

- 通过 `git add` 将工作目录的变更添加到暂存区。
- 支持选择性提交（例如修复多个 Bug 但分次提交）。
- **安全区**：暂存的内容不会被意外丢弃。

#### 1.2.4 控制



### 1.3 本地仓库（Local Repository）

#### 1.3.1 概述

存储在 `.git/objects` 中的版本数据库。

#### 1.3.2 作用

永久保存提交历史，形成可追溯的快照链。

#### 1.3.3 特性

- 通过 `git commit` 将暂存区内容生成 Commit 对象。
- 提交后生成四种对象：
  - `Blob`（文件内容）、`Tree`（目录结构）。
  - `Commit`（提交元数据）、`Tag`（可选标签）。



## 2. Git 的标准工作流程

### 2.1 工作流程介绍

#### 2.1.1 修改代码（工作目录）

```
echo "New content" > file.txt  # 修改或创建文件
```

- 文件状态变为 `Modified` 或 `Untracked`。

#### 2.1.2 暂存变更（工作目录 → 暂存区）

```
git add file.txt    # 暂存单个文件
git add .           # 暂存所有修改
```

- 文件状态变为 `Staged`。

#### 2.1.3 提交到本地仓库（暂存区 → 本地仓库）

```
git commit -m "Update file.txt"
```

- 生成 Commit 对象，分支指针（如 `main`）指向新提交。

#### 2.1.4 推送到远程仓库（可选，本地仓库 → 远程仓库）

```
git push origin main
```

- 将本地提交同步到远程（如 GitHub/Gitee）。

### 2.2 区域间的状态转换

| **操作**         | **影响区域**          | **命令示例**               |
| :--------------- | :-------------------- | :------------------------- |
| 修改文件         | 工作目录 → `Modified` | `vim file.txt`             |
| 暂存文件         | 工作目录 → 暂存区     | `git add file.txt`         |
| 提交             | 暂存区 → 本地仓库     | `git commit -m "msg"`      |
| 推送             | 本地仓库 → 远程仓库   | `git push origin main`     |
| 撤销工作目录修改 | 丢弃未暂存的改动      | `git checkout -- file.txt` |
| 撤销暂存区修改   | 移回工作目录          | `git reset HEAD file.txt`  |



## 1. git信息存储

### 1.1 Blob 对象（Binary Large Object）

**作用**

存储文件的内容（文本或二进制数据），但不包含文件名、路径等元信息。

**特点**

- **内容寻址**：以文件内容的 SHA-1 哈希值命名（如 `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`）。
- **去重**：相同内容的文件只会存储一个 Blob，即使文件名不同。
- **无结构**：仅存储原始数据，无任何附加信息。

**生成方式**

```
echo "Hello Git" | git hash-object --stdin  
# 输出 SHA-1 哈希值（如 8ab686eafeb1f44702738c8b0f24f2567c36da6d）
```

**存储路径**

`.git/objects/8a/b686eafeb1f44702738c8b0f24f2567c36da6d`
（前两位为子目录名，剩余部分为文件名）

**查看内容**

```
git cat-file -p 8ab686e  # 输出 "Hello Git"
```

**查看类别**

```
git cat-file -t 8ab686e  # 输出 "Hello Git"
```

### 1.2 Tree 对象

**作用**

记录目录结构，将文件名、权限与 Blob 或子目录（其他 Tree）关联起来，类似文件系统的目录。

**结构示例**

一个 Tree 对象可能包含以下条目：

```
100644 blob e69de29    README.md  
040000 tree 1a0b36a    src  
100755 blob 3c4e9cd    script.sh
```

字段解释：

- `100644`：文件权限（普通文件、可执行文件等）。
- `blob/tree`：指向的对象类型。
- `SHA-1`：对应 Blob 或 Tree 的哈希值。
- 文件名（如 `README.md`）。

**生成方式**

1. 通过 `git add` 将文件加入暂存区（Index），此时 Git 会生成 Blob 并更新 Tree。
2. 提交时，Git 根据暂存区生成根 Tree 对象。

**查看命令**

```
git ls-tree HEAD          # 查看当前提交的根 Tree  
git ls-tree <commit-hash> # 查看指定提交的 Tree
```

### 1.3 **Commit 对象**

**作用**

记录一次提交的元信息，并指向一个根 Tree（项目快照）和父提交（形成历史链）。

**结构示例**

```
tree 1a0b36a...  
parent d3b8c7f...  
author Alice <alice@example.com> 1620000000 +0800  
committer Bob <bob@example.com> 1620000000 +0800  

Initial commit message...
```

字段解释：

- `tree`：当前项目的根 Tree 对象哈希。
- `parent`：父提交的哈希（首次提交无此字段，合并提交可能有多个父提交）。
- `author` 和 `committer`：作者与提交者信息（可能不同）。
- 提交信息（Commit Message）。

**生成方式**

通过 `git commit` 生成，步骤如下：

1. 根据暂存区生成根 Tree 对象。
2. 创建 Commit 对象，关联 Tree 和父提交。
3. 更新当前分支引用（如 `refs/heads/main`）指向新 Commit。

**查看命令**

```
git cat-file -p <commit-hash>  # 查看 Commit 对象内容  
git show --pretty=raw <commit> # 显示完整提交信息
```

### 1.4 **Tag 对象（可选）**

**作用**

为特定提交提供一个永久、可读的引用（如版本号 `v1.0.0`），通常用于发布标记。

**类型**

1. **轻量标签（Lightweight Tag）**

   - 直接指向某个 Commit 的引用（存储在 `.git/refs/tags/`），不生成独立对象。

   ```
   git tag v1.0-lightweight <commit-hash>
   ```

2. **附注标签（Annotated Tag）**

   - 生成独立的 Tag 对象，包含标签作者、日期、注释等信息。

   ```
   git tag -a v1.0 -m "Release version 1.0"
   ```

**Tag 对象结构**

```
object 1a0b36a...  # 指向的 Commit 哈希  
type commit  
tag v1.0  
tagger Alice <alice@example.com> 1620000000 +0800  

Release version 1.0
```

**查看命令**

```
git cat-file -p v1.0      # 查看 Tag 对象内容  
git show v1.0            # 显示标签及关联的提交
```

















































