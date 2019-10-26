<!-- TOC -->

- [缘起](#缘起)
- [远程开发](#远程开发)
- [安装](#安装)
- [使用 Remote-Containers](#使用-remote-containers)
- [在容器中打开项目](#在容器中打开项目)
- [修改配置](#修改配置)
- [特定配置](#特定配置)
- [管理扩展](#管理扩展)
- [端口转发](#端口转发)
- [终端](#终端)
- [容器设置](#容器设置)
- [在容器环境中使用 Git](#在容器环境中使用-git)
  - [共享 Git 凭据](#共享-git-凭据)
  - [解决换行符问题](#解决换行符问题)
- [配置文件示例](#配置文件示例)
- [参考](#参考)

<!-- /TOC -->

## 缘起

我的主力操作系统是 windows, 但有时不得不需要一些 linux 下的特性,
比如某些工具没有 windows 版本, 无法使用 MakeFile 等.

自从微软推出了 Windows Subsystem for Linux (WSL) 之后,
这种情况已经好了不少了. 具体使用可以参考
[官方 WSL 文档](https://docs.microsoft.com/zh-cn/windows/wsl/about).

但我不太习惯使用它, 日常中更偏爱的是 docker.
毕竟还是镜像方便点, 需要什么组件就 pull 下来,
用完了或者中间搞坏了, 重新开一个就行了, 成本很低.

## 远程开发

[jetbrains](https://www.jetbrains.com/help/clion/remote-development.html)
的 IDE 是支持 remote development(远程开发)的,
幸好, VS Code 也开始支持了.

所谓的远程开发, 就是将远程的服务器, 或者容器, 或 WSL 作为开发环境.
本地的代码实时同步到远程中, 而各种工具比如命令行都是在远程环境中运行的.

下面就来介绍下如何使用 VS Code 进行远程开发.

## 安装

首先, 需要在 VS Code 中安装对应的插件,
[Remote Development extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack).

这其实是插件集合, 包括了 Remote-SSH , Remote-Containers, Remote-WSL 等.

安装完成之后, 重启 VS Code. 左下角会出现一个图标, 类似于 `><`.
点击之后, 会弹出对应的命令选择框.

![qucik icon](https://code.visualstudio.com/assets/docs/remote/common/remote-dev-status-bar.png)

这里主要介绍 `Remote-Containers`, 其他的 Remote-SSH 和 Remote-WSL 也类似, 具体可以参考官方文档说明.

## 使用 Remote-Containers

![Remote-Containers](https://code.visualstudio.com/assets/docs/remote/containers/architecture-containers.png)

上图是官方文档上的构架图, 可以看到源代码是通过卷映射进去的,
命令行和运行 APP 和 Debugger 都是在容器中完成的.

[系统要求](https://code.visualstudio.com/docs/remote/containers#_system-requirements)
直接看官方文档吧, 这里不再解释.
通常本地装好 docker 就行了.

## 在容器中打开项目

要在容器中打开项目, 会面临多种选择:

- 新建配置
  - 从预定义好的配置中
  - 从 docker-compose.yml 中
  - 从 Dockerfile 中
- 附加到已运行的容器中

详细梳理一下流程:

1. 选择 `Remote-Containers: Reopen Folder in Container`,
   如果此时本地还没有对应的配置, 就会触发 `新建配置` 的过程.
   这时可能有多种选项, 取决于本地项目中是否存在
   `Dockerfile` 和 `docker-compose.yml` 文件.

   1. 必有的选项: `From a predefined container configuration definition...`.
      这个选择会显示预定好的配置文件, 可以根据自己使用的语言或技术栈选择对应的配置.
   2. 本地项目中存在 `Dockerfile`: `From Dockerfile`.
      这会使用本地的 `Dockerfile` 构建容器.
   3. 本地项目中存在 `docker-compose.yml`: `From docker-compose.yml`.
      这个选择本地的 `docker-compose.yml` 中的其中一个容器作为开发环境.

2. 选择 `Remote-Containers: Attach to Running Container...`,
   会进入到 `附加到已运行的容器中` 的过程.
   此时, 选择对应的本地容器就行了.

其他的打开方式包括:

- 在容器中打开文件夹
  `Remote-Containers: Open Folder in Container...`
- 在容器中打开工作空间
  `Remote-Containers: Open Workspace in Container...`
- 在容器中打开 Repository(克隆代码且不同步改动到本地)
  `Remote-Containers: Open Repository in Container...`

## 修改配置

配置文件保存在 `.devcontainer` 中:

- `devcontainer.json` 配置选项
- `Dockerfile` 如果新建时选择 `Dockerfile`
- `docker-compose.yml` 如果新建时选择 `docker-compose.yml`

更多配置选项可以参考
[devcontainer.json reference](https://code.visualstudio.com/docs/remote/containers#_devcontainerjson-reference)

## 特定配置

对于 C++, Go, Rust 等使用 ptrace-based debugger 的语言,
需要开启以下配置:

修改 `devcontainer.json` 文件

```json
"runArgs": ["--cap-add=SYS_PTRACE", "--security-opt", "seccomp=unconfined"]
```

如果使用 `docker-compose.yml` 则
修改 `docker-compose.yml` 文件

```yaml
# Required for ptrace-based debuggers like C++, Go, and Rust
cap_add:
  - SYS_PTRACE
security_opt:
  - seccomp:unconfined
```

## 管理扩展

扩展会被分为两个部分, 本地和远程的.

本地的扩展主要是一些 UI 相关的, 其他扩展都安装在容器中的.

这可能对于多语言开发者(洁癖患者)来说是个不错的情况,
只要在容器中安装相应语言所需的扩展就行了,
可以无缝切换工作流.

对于想要完全同步本地扩展的用户, 可以使用命令
`Install Local Extensions in Dev Container:`
将所有的本地扩展都安装在容器中.

有时候, 很多扩展是必装的, 可以在设置中修改`默认扩展`:

```json
"remote.containers.defaultExtensions": [
    "eamodio.gitlens",
    "mutantdino.resourcemonitor"
]
```

扩展只能运行在一个位置, `ui` 或 `workspace`,
可以在 `devcontainer.json` 中强制指定扩展运行的位置.

```json
"settings": {
    "remote.extensionKind": {
        "ms-azuretools.vscode-docker": "workspace"
    }
},
```

## 端口转发

如果你在开发 web 应用, 很多时候都需要进行端口转发,
以便在本地的浏览器中调试, 这个时候可以使用端口转发功能.

在命令面板(F1)中选择 `Remote-Containers: Forward Port from Container...`

上面的操作通常用于临时转发端口.

如果你有固定的端口要转发, 可以选择以下两种方式之一:

修改 `devcontainer.json` 配置文件:

```json
"appPort": [ 3000, "8921:5000" ]
```

或者如果使用的是 `docker-compose.yml` 文件, 直接添加端口:

```yaml
ports:
  - "3000"
  - "8921:5000"
```

## 终端

终端已经变成了容器中的终端, 而不是本地的终端了.

享受 linux 的便利吧.

## 容器设置

现在设置文件已经有三份了, 相比普通的用户设置和工作空间设置,
多了一个容器设置, 优先级是最高的.

也可以在 `devcontainer.json` 设置一些默认的容器设置:

```json
"settings": {
    "java.home": "/docker-java-home"
}
```

## 在容器环境中使用 Git

### 共享 Git 凭据

如果使用 HTTPS 克隆项目, 且使用
[凭据助手](https://help.github.com/en/articles/caching-your-github-password-in-git)
则无需任何操作就可以共享凭据.

如果使用 SSH 密钥, 主要将 `~/.ssh` 安装到容器中就行了.

不过 windows 上有点小问题, `.ssh` 中的内容不会直接复制到容器中.

不过官方文档也提供了解决方案.

**第一步**:

复制 `.ssh` 文件夹到容器中.

如果使用 Dockerfile 或预定义的配置, 将下面的内容添加到
`devcontainer.json` 文件中:

```json
"runArgs": [ "-v", "${env:HOME}${env:USERPROFILE}/.ssh:/root/.ssh-localhost:ro" ]
```

如果使用 `docker-compose.yml`, 更新 `volumes` 字段:

```yaml
version: "3"
services:
  your-service-name-here:
    # ...
    volumes:
      - ~/.ssh:/root/.ssh-localhost:ro
```

**第二步**:

复制密钥, 并设置权限.

将以下内容复制到 `devcontainer.json` 文件中:

使用 root 身份运行容器时

```json
"postCreateCommand": "mkdir -p /root/.ssh && cp -r /root/.ssh-localhost/* /root/.ssh && chmod 700 /root/.ssh && chmod 600 /root/.ssh/*"
```

使用非 root 身份运行容器时

```json
"postCreateCommand": "sudo cp -r /root/.ssh-localhost ~ && sudo chown -R $(id -u):$(id -g) ~/.ssh-localhost && mv ~/.ssh-localhost ~/.ssh && chmod 700 ~/.ssh && chmod 600 ~/.ssh/*"
```

如果已经创建了容器了, 需要运行命令
`Remote-Containers：Rebuild Container`
重新构建容器, 以使得改变生效.

### 解决换行符问题

在项目的 `.gitattributes` 文件中添加以下内容:

```
* text=auto eol=lf
*.{cmd,[cC][mM][dD]} text eol=crlf
*.{bat,[bB][aA][tT]} text eol=crlf
```

## 配置文件示例

以下配置文件来自于
[go_web](https://github.com/zhenhua32/go_web),
这是一个使用 Go 语言建立 web 应用的示例项目.

`.devcontainer/devcontainer.json`

```json
// If you want to run as a non-root user in the container, see .devcontainer/docker-compose.yml.
{
  "name": "Existing Docker Compose (Extend)",

  // Update the 'dockerComposeFile' list if you have more compose files or use different names.
  // The .devcontainer/docker-compose.yml file contains any overrides you need/want to make.
  "dockerComposeFile": ["..\\docker-compose.yml", "docker-compose.yml"],

  // The 'service' property is the name of the service for the container that VS Code should
  // use. Update this value and .devcontainer/docker-compose.yml to the real service name.
  "service": "web",

  // The optional 'workspaceFolder' property is the path VS Code should open by default when
  // connected. This is typically a file mount in .devcontainer/docker-compose.yml
  "workspaceFolder": "/workspace",

  // Use 'settings' to set *default* container specific settings.json values on container create.
  // You can edit these settings after create using File > Preferences > Settings > Remote.
  "settings": {
    // This will ignore your local shell user setting for Linux since shells like zsh are typically
    // not in base container images. You can also update this to an specific shell to ensure VS Code
    // uses the right one for terminals and tasks. For example, /bin/bash (or /bin/ash for Alpine).
    "terminal.integrated.shell.linux": null
  },

  // Uncomment the next line if you want start specific services in your Docker Compose config.
  // "runServices": [],

  // Uncomment the next line if you want to keep your containers running after VS Code shuts down.
  // "shutdownAction": "none",

  // Uncomment the next line to run commands after the container is created - for example installing git.
  // "postCreateCommand": "apt-get update && apt-get install -y git",

  // Add the IDs of extensions you want installed when the container is created in the array below.
  "extensions": [],

  // Mount your .ssh folder to /root/.ssh-localhost so we can copy its contents
  "runArgs": [
    // "-v",
    // "${env:HOME}${env:USERPROFILE}/.ssh:/root/.ssh-localhost:ro"
  ],

  // Copy the contents to the correct location and set permissions
  "postCreateCommand": "mkdir -p ~/.ssh && cp -r ~/.ssh-localhost/* ~/.ssh && chmod 700 ~/.ssh && chmod 600 ~/.ssh/*"
}
```

`.devcontainer/docker-compose.yml`

```yaml
--- #-------------------------------------------------------------------------------------------------------------

#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
version: "3.7"
services:
  # Update this to the name of the service you want to work with in your docker-compose.yml file
  web:
    # You may want to add a non-root user to your Dockerfile. On Linux, this will prevent
    # new files getting created as root. See https://aka.ms/vscode-remote/containers/non-root-user
    # for the needed Dockerfile updates and then uncomment the next line.
    # user: vscode

    # Uncomment if you want to add a different Dockerfile in the .devcontainer folder
    # build:
    #   context: .
    #   dockerfile: Dockerfile

    # Uncomment if you want to expose any additional ports. The snippet below exposes port 3000.
    # ports:
    #   - 3000:3000

    volumes:
      # Update this to wherever you want VS Code to mount the folder of your project
      - .:/workspace
      - ~/.ssh:/root/.ssh-localhost:ro

      # Uncomment the next line to use Docker from inside the container. See https://aka.ms/vscode-remote/samples/docker-in-docker-compose for details.
      # - /var/run/docker.sock:/var/run/docker.sock

    environment:
      GOPROXY: "https://goproxy.io"

    # Uncomment the next four lines if you will use a ptrace-based debugger like C++, Go, and Rust.
    cap_add:
      - SYS_PTRACE
    security_opt:
      - seccomp:unconfined

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity
```

`docker-compose.yml`

```yaml
version: "3.7"

services:
  web:
    image: golang:1.13
    # https://docs.docker.com/compose/compose-file/#init
    init: true
    volumes:
      - .:/home/web
    environment:
      GOPROXY: "https://goproxy.io"

  mysql:
    image: mysql:8
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: "1234"
    ports:
      - 3306:3306

  adminer:
    image: adminer:4
    ports:
      - 8080:8080

  dbclient:
    image: mysql:8
    command: mysql -hmysql -uroot -p1234 -D db_apiserver
    # mysql -hmysql -uroot -p1234
    # source /home/script/db.sql
    # select * from tb_users \G;
    volumes:
      - ./script:/home/script

  nginx:
    image: nginx:stable-alpine
    ports:
      - 80:80
    volumes:
      - ./conf/nginx_web.conf:/etc/nginx/conf.d/nginx_web.conf
    command: nginx -g 'daemon off;'
```

## 参考

[VS Code Remote Development](https://code.visualstudio.com/docs/remote/remote-overview)

[Remote Development Tips and Tricks](https://code.visualstudio.com/docs/remote/troubleshooting)

[Docker Compose dev container definitions](https://code.visualstudio.com/docs/remote/containers#_docker-compose-dev-container-definitions)
