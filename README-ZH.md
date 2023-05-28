# autossh
[README](https://github.com/scleeyong/autossh/edit/master/README.md|[中文文档]([https://github.com/ehang-io/nps/blob/master/README_zh.md](https://github.com/scleeyong/autossh/edit/master/README-ZH.md))

[![Docker](https://badgen.net/badge/jnovack/autossh/blue?icon=docker)](https://hub.docker.com/r/jnovack/autossh)
[![Github](https://badgen.net/badge/jnovack/autossh/purple?icon=github)](https://github.com/jnovack/autossh)

高度可定制的 AutoSSH Docker 容器。

## 概述

**jnovack/autossh** 是一个小巧轻量（约15MB）的镜像，旨在提供一种安全的方式来建立 SSH 隧道，而不需要在镜像本身中包含密钥或与主机链接。


有成千上万个autossh Docker容器可供选择，为什么要使用这个呢？我希望您会发现它更易于使用。它体积更小、可定制性更强、具备自动化构建，并且非常易于使用，我希望您能从中学到一些东西。我尽力遵循标准和已经建立的约定，以便让您更容易理解并从该项目中复制和粘贴代码到其他项目中，以扩展您的知识！

## 描述

``autossh`` 是一个程序，用于启动一个 ssh 进程并监控它，如果它死掉或停止传输数据，会自动重新启动。

在开始之前，我想定义一些术语。

- *local* - 指的是当前的 Docker 容器。

- *target* - 隧道的终点和最终目的地。

- *remote* - 中间人，或者说是你通过它进行隧道连接以达到 *target*  的代理服务器。

- *source* - 最初的起点，它无法直接访问 *target*  终点，但可以访问 *remote* 终点。

通常情况下，*local* 机器与 *target* 是相同的，但由于我们在使用 Docker，我们必须将 *local* 容器与我们希望 **autossh** 连接的 *target* 终点分开。通常情况下，这就是 **autossh** 的运行位置。

通常情况下，*target* 可以位于家庭局域网段中，没有公共可访问的 IP 地址；而 *remote* 机器具有一个可被 *target* 和 *source* 访问的地址。而 *source* 只能访问 *remote*。

```text
target ---> |firewall| >--- remote ---< |firewall| <--- source
10.1.1.101               203.0.113.10            192.168.1.101
```

在这个示例中，*target* （运行 **autossh**）连接到 *remote*  服务器，并保持一个隧道以保持活动状态，以便 *source* 可以通过 *remote*  进行代理并访问 *target* 上的资源。将其视为"长距离端口转发"。

### 例子

您正在*target*上运行`docker`，即您的家用计算机。（注意：Linux Docker主机会自动创建一个名为`docker0`的接口，IP地址为`172.17.0.1`，以便容器可以路由到主机并连接到其他网络。一个启动的容器可能会有IP地址`172.17.0.2`，这是我们的示例中的情况。）您在互联网上拥有一个可访问的虚拟专用服务器（VPS）。这个*local* docker容器将与*remote* VPS建立连接，并将*remote*的2222端口隧道到*target*的22端口。任何对*remote*的2222端口的连接实际上都是连接到*target*服务器的22端口。这被称为“反向隧道”。

```text
      TARGET_PORT                  REMOTE_PORT    TUNNEL_PORT
 target <--------------- local ------------> remote <--------------- source
 10.1.1.101           172.17.0.2          203.0.113.10        192.168.1.101
```

> 本地设备（172.17.0.2）连接到远程设备（203.0.113.10）的远程端口（:22），以在远程设备（203.0.113.10）的隧道端口（:11111）上创建隧道。
> 源设备（192.168.1.101）连接到远程设备（203.0.113.10）的隧道端口（:11111），以访问目标设备（10.1.1.101）的目标端口（:22）。

默认情况下，SSH服务器应用程序（如OpenSSH、Dropbear等）仅允许从回环接口（`127.0.0.1`）连接到转发的端口。

这意味着您必须经过身份验证并连接到远程主机，然后才能继续连接到隧道。可以将远程主机视为一个“跳板”（暂且这样称呼）以进一步连接到隧道。

在上面的示例中，从 *source* 开始，您首先需要打开一个SSH连接到 *remote*（`203.0.113.10`），然后您可以通过连接到 `127.0.0.1:TUNNEL_PORT` 继续连接到 *target*（`10.1.1.101`）。这是一个两步的过程。

要将这个过程变为一步（通过 *remote* 从 *source* 连接到 *target*），您必须对 *remote* 进行一些安全性更改（不建议这样做）。请参阅下面的 [SSH_BIND_IP](#SSH_BIND_IP) 部分。

#### 免责声明

通过将远程端口2222隧道传输到目标端口22，您可能会将家庭服务器（以及您的家庭网络）暴露在广域互联网上，这通常被认为是"不好的事情(TM)"。请确保适当地使用防火墙、`fail2ban`脚本、非root访问权限、仅使用基于密钥的身份验证以及其他必要的安全措施。

## Setup

开始之前，您需要在 Docker 主机上生成一个 SSH 密钥。这样可以确保容器的密钥与您正常使用的用户密钥分开，以防需要撤销其中之一。

```text
$ ssh-keygen -t rsa -b 4096 -C "autossh" -f autossh_id_rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/jnovack/autossh_id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/jnovack/autossh_id_rsa.
Your public key has been saved in /home/jnovack/autossh_id_rsa.pub.
The key fingerprint is:
00:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff autossh
The key's randomart image is:
+-----[ RSA 4096]-----+
|     _.-'''''-._     |
|   .'  _     _  '.   |
|  /   (_)   (_)   \  |
| |  ,           ,  | |
| |  \`.       .`/  | |
|  \  '.`'""'"`.'  /  |
|   '.  `'---'`  .'   |
|     '-._____.-'     |
+---------------------+
```

## 命令行选项

一个没有自定义选项的Docker容器将会有何用？我提供了一系列广泛的环境变量，可以进行设置。一个没有自定义选项的Docker容器将会有何用？我提供了一系列广泛的环境变量，可以进行设置。

### 环境变量

所有环境变量都以`SSH_`为前缀，这并不意味着你必须进行SSH隧道连接，而是为了方便分组。唯一需要的SSH连接是从本地设备到远程服务器。然而，如果你有兴趣在网络间通过证书安全地隧道传输其他协议（如mysql、redis、mongodb），你可以考虑使用我的另一个项目[ambassador](https://hub.docker.com/r/jnovack/ambassador/)。

#### SSH_REMOTE_USER

指定在*远程*端点上的用户名。（默认值：`root`）

#### SSH_REMOTE_HOST

指定*远程*端点的地址（优先使用IP）。 （默认值：`localhost`）

#### SSH_REMOTE_PORT

指定连接到*远程*端点的`ssh`端口。（默认值：`22`）

#### SSH_TUNNEL_PORT

指定在*远程*端点上作为隧道入口的端口号。（默认值：随机 > 32768）如果您不希望每次重新启动**jnovack/autossh**时都使用新的端口，您可以选择显式设置此值。

如果设置了`SSH_MODE`选项（见下文），此选项将会反转。

#### SSH_TARGET_HOST

指定*目标*的地址（优先使用IP）。

#### SSH_TARGET_PORT

指定*目标*端点上将用作隧道出口或目标服务的端口号。通常情况下，这是`ssh`（端口号：22），但您也可以通过隧道传输其他服务，如redis（端口号：6379），elasticsearch（端口号：9200）或常见的http（端口号：80）和https（端口号：443）。

如果您有兴趣通过证书在网络间安全地隧道传输其他协议（例如mysql、redis、mongodb），您可能希望考虑使用我的另一个项目[ambassador](https://hub.docker.com/r/jnovack/ambassador/)。

#### SSH_STRICT_HOST_IP_CHECK

如果您不希望检查主机的IP地址（如果提供了`known_hosts`文件），请将其设置为`false`。这可以帮助避免对具有动态IP地址的主机产生问题，但会减少对DNS欺骗攻击的一些额外保护。默认情况下，启用主机IP检查。

#### SSH_KEY_FILE

如果您希望将密钥存储在Docker Secrets中，可以将其设置为`/run/secrets/*secret-name*`。

#### SSH_KNOWN_HOSTS_FILE

如果您希望将`known_hosts`文件存储在Docker Secrets中，可以将其设置为`/run/secrets/*secret-name*`。

#### SSH_MODE

定义了隧道的设置方式：

- `-R` 是默认值，表示远程转发模式。
- `-L` 表示本地转发模式。

#### SSH_BIND_IP

您可以定义隧道在 *remote*（SSH_MODE 为 `-R`）或 *local*（SSH_MODE 为 `-L`）上绑定的 IP 地址。默认情况下，只使用 `127.0.0.1`。

##### SSH_MODE of `-R` (default)

**警告**: _此过程涉及更改服务器的安全设置，并将使您的 *target* 暴露给其他网络和潜在的互联网。不建议在不采取额外预防措施的情况下执行此过程。_

除非您正确配置了 *remote* 服务器配置文件中的 `GatewayPorts` 变量，否则此选项将不会生效。请参阅您的 SSH 服务器文档以进行正确设置。

##### SSH_MODE of `-L`

您可能希望将此设置为 `0.0.0.0`，以便将您的 `SSH_TUNNEL_PORT` 绑定到 *local* 端的所有接口上。

#### SSH_SERVER_ALIVE_INTERVAL

设置一个超时时间间隔（以秒为单位），如果从服务器未接收到任何数据，ssh(1)将通过加密通道发送一条消息以请求服务器响应。

- `0` 将关闭该选项。
- `10` 是此镜像的默认值。

有关更多详细信息，请参阅 [`ssh_config(5)`](https://linux.die.net/man/5/ssh_config)。

#### SSH_SERVER_ALIVE_COUNT_MAX

设置活动消息的阈值，超过该阈值后连接将被终止并重新建立。

- `3` 是此镜像的默认值。
- `SSH_SERVER_ALIVE_INTERVAL=0` 将使此变量无效。

有关更多详细信息，请参阅 [`ssh_config(5)`](https://linux.die.net/man/5/ssh_config)。

#### SSH_OPTIONS 
设置 `ssh` 连接的附加参数。支持多个参数。

示例：
 - SSH_OPTIONS="-o StreamLocalBindUnlink=yes" 用于在存在时重新创建套接字
 - SSH_OPTIONS="-o StreamLocalBindUnlink=yes -o UseRoaming=no" 用于多个参数

有关更多详细信息，请参阅 [`ssh_config(5)`](https://linux.die.net/man/5/ssh_config)。

#### Additional Environment variables

附加的环境变量：
- [`autossh(1)`](https://linux.die.net/man/1/autossh)
- [`ssh_config(5)`](https://linux.die.net/man/5/ssh_config)

### Mounts

挂载（Mounts）是可选的，用于简化使用。但更好的方式是使用可以存储在配置文件中并且易于传输（和备份）的[环境变量](#Environment_Variables)。

#### /id_rsa

将你在**Setup**步骤中生成的密钥文件挂载，或设置`SSH_KEY_FILE`环境变量。

```sh
-v /path/to/id_rsa:/id_rsa
```

#### /known_hosts

如果要启用`StrictHostKeyChecking`，请挂载`known_hosts`文件，或设置`SSH_KNOWN_HOSTS_FILE`环境变量。

```sh
-v /path/to/known_hosts:/known_hosts
```

## 示例

### docker-compose.yml

在第一个示例`ssh-to-docker-host`中，将从Docker容器（名为`autossh-ssh-to-docker-host`）创建一个隧道到运行Docker容器的主机。
使用方法是，通过`ssh`连接到虚拟的互联网地址`203.0.113.10:2222`，您将被转发到`172.17.0.2:22`（运行Docker容器的主机）。

在第二个示例`ssh-to-lan-endpoint`中，将建立一个隧道到Docker主机的私有LAN上的主机。通过`ssh`连接到虚拟的互联网地址`203.0.113.10:22222`，数据将通过Docker容器、Docker主机，最终到达私有LAN上的主机`192.168.123.45:22`。

在第三个示例`ssh-local-forward-on-1234`中，将在容器上创建一个本地转发到`198.168.123.45:22`，映射到端口`1234`。隧道将通过`203.0.113.10:22222`建立。

要使用这些示例，请使用适当的SSH客户端连接到提供的虚拟地址和端口。

```yml
version: '3.7'

services:
  ssh-to-docker-host:
    image: jnovack/autossh
    container_name: autossh-ssh-to-docker-host
    environment:
      - SSH_REMOTE_USER=sshuser
      - SSH_REMOTE_HOST=203.0.113.10
      - SSH_REMOTE_PORT=2222
      - SSH_TARGET_HOST=172.17.0.2
      - SSH_TARGET_PORT=22
    restart: always
    volumes:
      - /etc/autossh/id_rsa:/id_rsa
    dns:
      - 8.8.8.8
      - 1.1.1.1

  ssh-to-lan-endpoint:
    image: jnovack/autossh
    container_name: autossh-ssh-to-lan-endpoint
    environment:
      - SSH_REMOTE_USER=sshuser
      - SSH_REMOTE_HOST=203.0.113.10
      - SSH_REMOTE_PORT=22222
      - SSH_TARGET_HOST=198.168.123.45
      - SSH_TARGET_PORT=22
    restart: always
    volumes:
      - /etc/autossh/id_rsa:/id_rsa
    dns:
      - 8.8.8.8
      - 4.2.2.4
  
  ssh-local-forward-on-1234:
    image: jnovack/autossh
    container_name: autossh-ssh-local-forward
    environment:
      - SSH_REMOTE_USER=sshuser
      - SSH_REMOTE_HOST=203.0.113.10
      - SSH_REMOTE_PORT=22222
      - SSH_BIND_IP=0.0.0.0
      - SSH_TUNNEL_PORT=1234
      - SSH_TARGET_HOST=198.168.123.45
      - SSH_TARGET_PORT=22
      - SSH_MODE=-L
    restart: always
    volumes:
      - /etc/autossh/id_rsa:/id_rsa
    dns:
      - 8.8.8.8
      - 4.2.2.4
    
```

## Multi-Arch Images

该镜像在Docker Hub上自动构建了以下架构：

- `amd64`
- `armv6` (e.g. Raspberry Pi Zero)
- `armv7` (e.g. Raspberry Pi 2 through 4)
- `arm64v8` (e.g. Amazon EC2 A1 Instances)

