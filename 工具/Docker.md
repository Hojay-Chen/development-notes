# Docker

## 一、安装(Ubuntu为例)

### 1.下载指南

​	https://docs.docker.com/engine/install/ubuntu/



### 2.先决条件

#### 防火墙限制

> **警告**
>
> 在安装 Docker 之前，请务必考虑以下事项 安全隐患和防火墙不兼容。

- 如果您使用 ufw 或 firewalld 来管理防火墙设置，请注意 当您使用 Docker 公开容器端口时，这些端口会绕过 防火墙规则。有关更多信息，请参阅 [Docker 和 ufw](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)。

- Docker 仅与 和 兼容。 在安装了 Docker 的系统上不支持使用 创建的防火墙规则。 确保您使用的任何防火墙规则集都是使用 或 创建的。 并将它们添加到链中， 请参阅[数据包筛选和防火墙](https://docs.docker.com/engine/network/packet-filtering-firewalls/)。

  `iptables-nft``iptables-legacy``nft``iptables``ip6tables``DOCKER-USER`

#### 作系统要求

要安装 Docker Engine，您需要以下 Ubuntu 之一的 64 位版本 版本：

- Ubuntu 神谕 24.10
- Ubuntu Noble 24.04 （LTS）
- Ubuntu Jammy 22.04 （LTS）
- Ubuntu Focal 20.04 （LTS）

Docker Engine for Ubuntu 与 x86_64（或 amd64）、armhf、arm64、 S390x 和 PPC64LE （PPC64EL） 架构。

> **注意**
>
> 在 Ubuntu 衍生发行版（如 Linux Mint）上安装不是正式的 支持（尽管它可能有效）。



### 3.卸载旧版本

在安装 Docker Engine 之前，您需要卸载任何冲突的软件包。

您的 Linux 发行版可能提供非官方的 Docker 软件包，这可能会发生冲突 使用 Docker 提供的官方软件包。您必须卸载这些软件包 在安装 Docker Engine 正式版之前。

要卸载的非官方软件包是：

- `docker.io`
- `docker-compose`
- `docker-compose-v2`
- `docker-doc`
- `podman-docker`

此外，Docker Engine 依赖于 和 。Docker 引擎 将这些依赖项捆绑为一个捆绑包：。如果你有 已安装或之前卸载它们以避免 与 Docker Engine 捆绑的版本冲突。`containerd``runc``containerd.io``containerd``runc`

执行以下命令卸载所有冲突的软件包。

```console
$ for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

`apt-get`可能会报告您没有安装这些软件包。

存储在其中的映像、容器、卷和网络不是 卸载 Docker 时自动删除。如果您想从 全新安装，并希望清理任何现有数据，请阅读[卸载 Docker Engine](https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine) 部分。`/var/lib/docker/`



### 4.安装方法

在新主机上首次安装 Docker Engine 之前，您需要 需要设置 Docker 存储库。之后，您可以安装和更新 存储库中的 Docker。`apt`

1. 设置 Docker 的存储库。`apt`

   ```bash
   # Add Docker's official GPG key:
   sudo apt-get update
   sudo apt-get install ca-certificates curl
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   
   # Add the repository to Apt sources:
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
     $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt-get update
   ```

2. 安装 Docker 软件包。

   最近的 特定版本

   ------

   要安装最新版本，请运行：

   

   ```console
   $ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

   ------

3. 通过运行映像来验证安装是否成功：`hello-world`

   

   ```console
   $ sudo docker run hello-world
   ```

   此命令将下载测试映像并在容器中运行它。当 container 运行时，它会打印确认消息并退出。

现在，您已成功安装并启动 Docker Engine。

> **提示**
>
> 尝试在没有 root 的情况下运行时收到错误？
>
> 用户组存在，但不包含任何用户，这就是需要您的原因 用于运行 Docker 命令。继续执行 [Linux postinstall](https://docs.docker.com/engine/install/linux-postinstall)，以允许非特权用户运行 Docker 命令和其他可选配置步骤。`docker``sudo`



### 5.卸载Docker Engine

1. 卸载 Docker Engine、CLI、containerd 和 Docker Compose 软件包：

   ```console
   $ sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
   ```

2. 主机上的映像、容器、卷或自定义配置文件 不会自动删除。要删除所有镜像、容器和卷，请执行以下作：

   ```console
   $ sudo rm -rf /var/lib/docker
   $ sudo rm -rf /var/lib/containerd
   ```

3. 删除源列表和密钥环

   ```console
   $ sudo rm /etc/apt/sources.list.d/docker.list
   $ sudo rm /etc/apt/keyrings/docker.asc
   ```

您必须手动删除任何已编辑的配置文件。

## 二、指令

### 1.容器声明周期管理

#### 1.1 run

**语法**

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

常用参数说明：

- **`-d`**: 后台运行容器并返回容器 ID。
- **`-it`**: 交互式运行容器，分配一个伪终端。
- **`--name`**: 给容器指定一个名称。
- **`-p`**: 端口映射，格式为 `host_port:container_port`。
- **`-v`**: 挂载卷，格式为 `host_dir:container_dir`。
- ~~**`--rm`**: 容器停止后自动删除容器。~~
- **`--env` 或 `-e`**: 设置环境变量。
- ~~**`--network`**: 指定容器的网络模式。~~
- **`--restart`**: 容器的重启策略（如 `no`、`on-failure`、`always`、`unless-stopped`）。
- **`-u`**: 指定用户。

**实例**

**1. 基本使用**

```
docker run ubuntu
```

拉取 ubuntu 镜像并在前台启动一个容器。w

**2. 后台运行容器**

```
docker run -d ubuntu
```

在后台运行 ubuntu 容器并返回容器 ID。

**3. 交互式运行并分配终端**

```
docker run -it ubuntu /bin/bash
```

以交互模式运行 ubuntu 容器，并启动一个 Bash shell。

**4. 指定容器名称**

```
docker run --name my_container ubuntu
```

运行一个 ubuntu 容器，并将其命名为 my_container。

**5. 端口映射**

```
docker run -p 8080:80 nginx
```

将本地主机的 8080 端口映射到容器内的 80 端口，运行 nginx 容器。

**6. 挂载卷**

```
docker run -v /host/data:/container/data ubuntu
```

将主机的 /host/data 目录挂载到容器内的 /container/data 目录。

**7. 设置环境变量**

```
docker run -e MY_ENV_VAR=my_value ubuntu
```

设置环境变量 MY_ENV_VAR 的值为 my_value，运行 ubuntu 容器。

~~**8. 使用网络模式**~~

```
docker run --network host nginx
```

~~使用主机的网络模式运行 nginx 容器。~~

**9. 指定重启策略**

```
docker run --restart always nginx
```

设置容器的重启策略为 always，即使容器停止也会自动重启。

**10. 指定用户**

```
docker run -u user123 ubuntu
```

以 user123 用户运行 ubuntu 容器。

**11. 组合多个选项**

```
docker run -d -p 8080:80 -v /host/data:/data --name webserver nginx
```

后台运行一个命名为 webserver 的 nginx 容器，将主机的 8080 端口映射到容器的 80 端口，并将主机的 /host/data 目录挂载到容器的 /data 目录。



#### 1.2 start

**语法**

```
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

~~**参数**~~

- ~~**`-a`**: 附加到容器的标准输入输出流。~~
- ~~**`-i`**: 附加并保持标准输入打开。~~

实例

启动一个容器:

```
docker start my_container
```

启动名称为 my_container 的容器。



~~启动并附加到容器:~~

```
docker start -a my_container
```

~~启动容器并附加到它的标准输入输出流。~~



同时启动多个容器:

```
docker start container1 container2 container3
```

同时启动 container1、container2 和 container3 容器。



#### 1.3 stop

**语法**

```
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

**参数**

**-t, --time**: 停止容器之前等待的秒数，默认是 10 秒。

**实例**

停止一个容器:

```
docker stop my_container
```

停止名称为 my_container 的容器。



指定等待时间停止容器:

```
docker stop -t 30 my_container
```

等待 30 秒后停止容器。



同时停止多个容器:

```
docker stop container1 container2 container3
```

同时停止 container1、container2 和 container3 容器。



#### 1.4 restart

**语法**

```
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

**参数**

**-t, --time**: 重启容器之前等待的秒数，默认是 10 秒。

**实例**

重启一个容器:

```
docker restart my_container
```

重启名称为 my_container 的容器。



指定等待时间重启容器:

```
docker restart -t 15 my_container
```

等待 15 秒后重启容器。



同时重启多个容器:

```
docker restart container1 container2 container3
```

同时重启 container1、container2 和 container3 容器。



#### 1.5 kill

docker kill 命令用于立即终止一个或多个正在运行的容器。

与 docker stop 命令不同，docker kill 命令会直接发送 **SIGKILL 信号**给容器的主进程，导致容器立即停止，而不会进行优雅的关闭。

**语法**

```
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

OPTIONS说明：

- **`-s, --signal`**: 发送给容器的信号（默认为 SIGKILL）。

**常用信号**

- **SIGKILL**: 强制终止进程（默认信号）。
- **SIGTERM**: 请求进程终止。
- **SIGINT**: 发送中断信号，通常表示用户请求终止。
- **SIGHUP**: 挂起信号，通常表示终端断开。

**实例**

立即终止一个容器:

```
docker kill my_container
```

立即终止名称为 my_container 的容器。



发送自定义信号:

```
docker kill -s SIGTERM my_container
```

向名称为 my_container 的容器发送 SIGTERM 信号，而不是默认的 SIGKILL 信号。



同时终止多个容器:

```
docker kill container1 container2 container3
```

立即终止 container1、container2 和 container3 容器。

`docker kill` 命令用于立即终止一个或多个正在运行的容器，通过发送信号（默认 SIGKILL）给容器的主进程实现。与 `docker stop` 不同，`docker kill` 不会等待容器优雅地停止，而是直接终止进程。该命令在需要快速停止容器时非常有用，但应谨慎使用以避免数据损失或不一致。



#### 1.6 rm

docker rm 命令用于删除一个或多个**已经停止的容器**。

docker rm 命令不会删除正在运行的容器，如果你需要先停止容器，可以使用 docker stop 命令。

**语法**

```
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

- `CONTAINER [CONTAINER...]`: 一个或多个要删除的容器的名称或 ID。

OPTIONS说明：

- `-f`, `--force`: 强制删除正在运行的容器（使用 `SIGKILL` 信号）。
- `-l`, `--link`: 删除指定的连接，而不是容器本身。
- `-v`, `--volumes`: 删除容器挂载的卷。

删除单个容器：

```
docker rm <container_id_or_name>
```

你可以一次性删除多个容器，只需将容器 ID 或名称用空格分隔：

```
docker rm <container_id_or_name1> <container_id_or_name2> ...
```

你可以使用以下命令来删除所有已停止的容器：

```
docker container prune
```

这个命令会提示你确认删除所有已停止的容器。

**实例**

假设你有以下两个容器：

```
CONTAINER ID        IMAGE          COMMAND             CREATED             STATUS              PORTS      NAMES
abcd1234            nginx         "nginx -g 'daemon…" 2 minutes ago       Exited (0) 1 minute ago        my_nginx
efgh5678            redis        "redis-server"      3 minutes ago       Exited (0) 2 minutes ago       my_redis
```

你可以使用以下命令删除它们：

```
docker rm abcd1234 efgh5678
```

或者，你可以使用容器名称删除它们：

```
docker rm my_nginx my_redis
```

删除所有已经停止的容器：

```
docker rm $(docker ps -a -q)
```



#### 1.7 pause

docker pause 和 docker unpause 命令用于暂停和恢复容器中的所有进程。

**docker pause** - 暂停容器中所有的进程。

暂停的容器不会被终止，但其进程将被挂起，直到容器被恢复。这在需要临时暂停容器活动的情况下非常有用。

**语法**

```
docker pause CONTAINER [CONTAINER...]
```

**实例**

暂停一个容器：

```
docker pause my_container
```

暂停名称为 my_container 的容器中的所有进程。

暂停多个容器：

```
docker pause container1 container2
```

同时暂停 container1 和 container2 容器中的所有进程。



#### 1.8 unpause

docker pause 和 docker unpause 命令用于暂停和恢复容器中的所有进程。

**docker unpause** - 恢复容器中所有的进程。

**语法**

```
docker unpause CONTAINER [CONTAINER...]
```

**实例**

恢复一个容器：

```
docker unpause my_container
```

恢复名称为 my_container 的容器中的所有进程。

恢复多个容器：

```
docker unpause container1 container2
```

同时恢复 container1 和 container2 容器中的所有进程。

**使用场景**

- **临时暂停活动**: 当需要临时暂停容器中的所有活动以进行系统维护或资源管理时，可以使用 `docker pause`。
- **资源管理**: 在需要重新分配系统资源时，暂停不必要的容器以释放资源。
- **调试和故障排除**: 在调试或故障排除过程中暂停容器以分析当前状态。



#### 1.9 create

`docker create` 命令用于创建一个新的容器，但不会启动它。

`docker create` 命令会根据指定的镜像和参数创建一个容器实例，但容器只会在创建时进行初始化，并不会执行任何进程。

用法同 [docker run](https://www.runoob.com/docker/docker-run-command.html)。

**语法**

```
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**常用参数**

- **`--name`**: 给容器指定一个名称。
- **`-p, --publish`**: 端口映射，格式为 `host_port:container_port`。
- **`-v, --volume`**: 挂载卷，格式为 `host_dir:container_dir`。
- **`-e, --env`**: 设置环境变量。
- **`--network`**: 指定容器的网络模式。
- **`--restart`**: 容器的重启策略（如 `no`、`on-failure`、`always`、`unless-stopped`）。
- **`-u, --user`**: 指定用户。
- **`--entrypoint`**: 覆盖容器的默认入口点。
- **`--detach`**: 在后台创建容器。

**实例**

创建一个容器：

```
docker create ubuntu
```

根据 ubuntu 镜像创建一个容器，但不会启动它。

创建并指定容器名称：

```
docker create --name my_container ubuntu
```

创建一个名为 my_container 的容器，但不会启动它。

创建并设置环境变量：

```
docker create -e MY_ENV_VAR=my_value ubuntu
```

创建一个容器，并设置环境变量 MY_ENV_VAR 的值为 my_value。

创建并挂载卷：

```
docker create -v /host/data:/container/data ubuntu
```

创建一个容器，并将主机的 /host/data 目录挂载到容器的 /container/data 目录。

创建并端口映射：

```
docker create -p 8080:80 nginx
```

创建一个容器，将本地主机的 8080 端口映射到容器的 80 端口，但不会启动它。

创建并指定重启策略：

```
docker create --restart always nginx
```

创建一个容器，并将重启策略设置为 always。

创建并指定用户：

```
docker create -u user123 ubuntu
```

创建一个容器，并以 user123 用户运行容器。

**查看容器**

在创建容器之后，可以使用 docker ps -a 命令查看所有容器，包括已创建但未启动的容器。

```
docker ps -a
```

**启动已创建的容器**

使用 docker start 命令来启动已创建但未启动的容器：

```
docker start my_container
```



#### 1.10 exec

`docker exec` 命令用于在运行中的容器内执行一个新的命令。这对于调试、运行附加的进程或在容器内部进行管理操作非常有用。

**语法**

```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

**常用参数**

- **`-d, --detach`**: 在后台运行命令。
- **`--detach-keys`**: 覆盖分离容器的键序列。
- **`-e, --env`**: 设置环境变量。
- **`--env-file`**: 从文件中读取环境变量。
- **`-i, --interactive`**: 保持标准输入打开。
- **`--privileged`**: 给这个命令额外的权限。
- **`--user, -u`**: 以指定用户的身份运行命令。
- **`--workdir, -w`**: 指定命令的工作目录。
- **`-t, --tty`**: 分配一个伪终端。

**实例**

在容器内运行命令：

```
docker exec my_container ls /app
```

在运行中的 my_container 容器内执行 ls /app 命令，列出 /app 目录的内容。

以交互模式运行命令：

```
docker exec -it my_container /bin/bash
```

在运行中的 my_container 容器内启动一个交互式的 Bash shell。-i 保持标准输入打开，-t 分配一个伪终端。

在后台运行命令：

```
docker exec -d my_container touch /app/newfile.txt
```

在运行中的 my_container 容器内后台执行 touch /app/newfile.txt 命令，创建一个新文件。

设置环境变量：

```
docker exec -e MY_ENV_VAR=my_value my_container env
```

在运行中的 my_container 容器内执行 env 命令，并设置环境变量 MY_ENV_VAR 的值为 my_value。

以指定用户身份运行命令：

```
docker exec -u user123 my_container whoami
```

在运行中的 my_container 容器内以 user123 用户身份执行 whoami 命令。

指定工作目录：

```
docker exec -w /app my_container pwd
```

在运行中的 my_container 容器内以 /app 目录为工作目录执行 pwd 命令。

**使用场景**

- **调试容器**: 进入容器内部进行调试和排查问题。
- **管理任务**: 在容器内运行附加的管理任务或维护操作。
- **监控和检查**: 在容器内执行监控和检查命令，获取运行状态和日志。



### 2. 容器操作

#### 2.1 ps

**语法**

```
docker ps [OPTIONS]
```

OPTIONS说明：

- **`-a, --all`**: 显示所有容器，包括停止的容器。
- **`-q, --quiet`**: 只显示容器 ID。
- **`-l, --latest`**: 显示最近创建的一个容器，包括所有状态。
- **`-n`**: 显示最近创建的 n 个容器，包括所有状态。
- **`--no-trunc`**: 不截断输出。
- **`-s, --size`**: 显示容器的大小。
- **`--filter, -f`**: 根据条件过滤显示的容器。
- **`--format`**: 格式化输出。

**实例**

**1、列出所有在运行的容器信息**

默认情况下，docker ps 只显示正在运行的容器。

```
docker ps
CONTAINER ID   IMAGE          COMMAND                ...  PORTS                    NAMES
09b93464c2f7   nginx:latest   "nginx -g 'daemon off" ...  80/tcp, 443/tcp          myrunoob
96f7f14e99ab   mysql:5.6      "docker-entrypoint.sh" ...  0.0.0.0:3306->3306/tcp   mymysql
```

输出详情介绍：

**CONTAINER ID:** 容器 ID。

**IMAGE:** 使用的镜像。

**COMMAND:** 启动容器时运行的命令。

**CREATED:** 容器的创建时间。

**STATUS:** 容器状态。

状态有7种：

- created（已创建）
- restarting（重启中）
- running（运行中）
- removing（迁移中）
- paused（暂停）
- exited（停止）
- dead（死亡）

**PORTS:** 容器的端口信息和使用的连接类型（tcp\udp）。

**NAMES:** 自动分配的容器名称。

**2、列出所有容器，包括停止的容器**

```
docker ps -a
```

显示所有容器，包括停止的容器。

**3、只显示容器 ID**

```
docker ps -q
```

只显示容器 ID。

**4、显示最近创建的一个容器**

```
docker ps -l
```

显示最近创建的一个容器，包括所有状态。

**5、显示最近创建的 n 个容器**

```
docker ps -n 3
```

显示最近创建的 3 个容器，包括所有状态。

**6、显示容器的大小**

```
docker ps -s
```

显示容器的大小。

**7、根据条件过滤显示的容器**

```
docker ps -f "status=exited"
```

显示状态为 exited 的容器。

```
docker ps -f "name=my_container"
```

显示名称包含 my_container 的容器。

**8、格式化输出**

```
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"
```

以表格形式显示容器的 ID、名称和状态。

**常见过滤器**

- **`status`**: 容器状态（如 `running`、`paused`、`exited`）。
- **`name`**: 容器名称。
- **`id`**: 容器 ID。
- **`label`**: 容器标签。
- **`ancestor`**: 容器镜像。

**使用场景**

- **监控容器状态**: 实时监控运行中的容器状态和资源使用情况。
- **调试和管理**: 查看所有容器，包括停止的容器，以便进行调试和管理操作。
- **自动化脚本**: 使用过滤器和格式化选项，便于在自动化脚本中获取特定容器信息。



#### 2.2 update

**语法**

```
docker update [OPTIONS] CONTAINER [CONTAINER...]
```

**常用参数**

- `CONTAINER`：要更新资源限制的容器名称或容器 ID。你可以指定一个或多个容器。
- `OPTIONS`：用于指定需要更新的资源限制。

**常用选项 OPTIONS：**

~~**1、`--memory, -m`**：设置容器的内存限制。~~

- ~~格式：`<size>[<unit>]`~~
- ~~例如：`500m`、`2g` 等。~~

```
docker update -m 2g my_container
```

~~**2、`--memory-swap`**：设置容器的内存和交换空间（swap）的总限制。如果设置为 `-1`，表示不限制交换空间。~~

- ~~格式：`<size>[<unit>]`，如 `2g`，或 `-1` 表示无限制。~~

```
docker update --memory-swap 3g my_container
```

~~3、**`--cpu-shares`**：设置容器的 CPU 优先级，相对值。默认为 `1024`，较大的值表示较高的优先级。~~

- ~~该选项不会直接限制容器的 CPU 使用量，而是控制 CPU 资源分配的优先级。~~

```
docker update --cpu-shares 2048 my_container
```

~~**4、`--cpus`**：设置容器使用的 CPU 核心数。这个选项可以限制容器最多使用的 CPU 核心数。~~

- ~~格式：`<number>`，例如：`1.5` 表示最多使用 1.5 个 CPU 核心。~~

```
docker update --cpus 2 my_container
```

~~**5、`--cpu-period`**：设置 CPU 周期时间。用于配合 `--cpu-quota` 限制容器的 CPU 使用时间。单位是微秒（默认值：`100000` 微秒 = 100ms）。~~

```
docker update --cpu-period 50000 my_container
```

~~**6、`--cpu-quota`**：设置容器在每个周期内可以使用的最大 CPU 时间。单位是微秒。需要与 `--cpu-period` 配合使用。~~

```
docker update --cpu-quota 25000 my_container
```

~~**7、`--blkio-weight`**：设置块 I/O 权重（范围：`10` 到 `1000`），表示容器对磁盘 I/O 操作的优先级。默认值为 `500`。~~

```
docker update --blkio-weight 800 my_container
```

~~**8、`--pids-limit`**：设置容器可以使用的最大进程数。~~

- ~~格式：`<number>`，例如：`100`。~~

```
docker update --pids-limit 200 my_container
```

**9、`--restart`**：设置容器的重启策略（`no`、`on-failure`、`always`、`unless-stopped`）。

```
docker update --restart always my_container
```

**实例**

~~**1. 更新容器的内存限制：**~~

```
docker update -m 2g my_container
```

~~这条命令将 my_container 的内存限制更新为 2GB。~~

~~**2. 设置 CPU 核心数限制：**~~

```
docker update --cpus 1.5 my_container
```

~~这条命令将 my_container 限制为最多使用 1.5 个 CPU 核心。~~

~~**3. 更新容器的 CPU 权重：**~~

```
docker update --cpu-shares 1024 my_container
```

~~这条命令将容器的 CPU 权重设置为 1024，默认值就是 1024。~~

~~**4. 更新容器的块 I/O 权重：**~~

```
docker update --blkio-weight 700 my_container
```

~~这条命令将容器的磁盘 I/O 权重设置为 700，权重范围是 10 到 1000。~~

~~**使用限制**~~

- ~~`docker update` 命令会立即生效，但并不会影响容器内运行的应用程序，容器继续保持运行状态。~~
- ~~仅支持调整容器的资源限制，对于其他容器配置（如环境变量、端口映射等）无法修改。~~



### 3. 镜像操作

#### 3.1 pull

**语法**

```
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

- **`NAME`**: 镜像名称，通常包含注册表地址（如 `docker.io/library/ubuntu`）。
- **`TAG`**（可选）: 镜像标签，默认为 `latest`。
- **`DIGEST`**（可选）: 镜像的 SHA256 摘要。

常用选项：

- **`--all-tags, -a`**: 下载指定镜像的所有标签。
- **`--disable-content-trust`**: 跳过镜像签名验证。

**1、拉取默认标签（latest）的镜像**

```
docker pull ubuntu
```

这会从 Docker Hub 拉取名为 ubuntu 的镜像，标签默认为 latest。

**2、拉取特定标签的镜像**

```
docker pull ubuntu:20.04
```

这会从 Docker Hub 拉取名为 ubuntu 的镜像，标签为 20.04。

**3、拉取特定摘要的镜像**

```
docker pull ubuntu@sha256:12345abcdef...
```

这会拉取具有特定 SHA256 摘要的 ubuntu 镜像。

**4、拉取所有标签的镜像**

```
docker pull --all-tags ubuntu
```

这会拉取 ubuntu 镜像的所有可用标签。

**5、从自定义注册表拉取镜像**

```
docker pull myregistry.com/myrepo/myimage:mytag
```

这会从 myregistry.com 注册表中拉取 myrepo 仓库中的 myimage 镜像，标签为 mytag。

**实例**

1、拉取 Ubuntu 镜像：

```
docker pull ubuntu
```

输出示例：

```
Using default tag: latest
latest: Pulling from library/ubuntu
Digest: sha256:12345abcdef...
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

2、拉取指定标签的 Ubuntu 镜像

```
docker pull ubuntu:20.04
```

输出示例：

```
20.04: Pulling from library/ubuntu
Digest: sha256:67890abcdef...
Status: Downloaded newer image for ubuntu:20.04
docker.io/library/ubuntu:20.04
```

**注意事项**

- 默认标签为 `latest`，但最好显式指定标签以避免拉取意外的版本。
- 确保有足够的磁盘空间来存储拉取的镜像。
- 在生产环境中，建议使用镜像的摘要（digest）以确保镜像的唯一性和一致性。

`docker pull` 命令是获取 Docker 镜像的基本工具，通过指定镜像名称、标签或摘要，可以从 Docker 注册表中下载所需的镜像。



## 三、

