# Halo建站（阿里云ECS）

## 安装Halo

### 环境说明

- 阿里云ECS，Ubuntu 24.04
<!-- - 8.152.199.10 -->

- 创建用户

    用 root 用户登录 ECS 后，执行 adduser 增加普通用户 gdv2，并将 gdv2 用户添加到 sudo 用户组。后续操作以 gdv2 用户来进行。

    ```bash
    adduser --home /home/gdv2 --shell /bin/bash gdv2
    adduser gdv2 sudo
    ```

### 安装docker 和 docker compose

1. 安装docker

    新的 ECS，还没有 docker。根据系统提示，执行 `sudo apt install docker.io` 安装 docker。
    
    ```bash
    gdv2@iZ2ze7frf973fqzg6dgqlmZ:~$ docker
    Command 'docker' not found, but can be installed with:
    sudo apt install docker.io      # version 28.2.2-0ubuntu1~24.04.1, or
    sudo apt install podman-docker  # version 4.9.3+ds1-1ubuntu0.2
    gdv2@iZ2ze7frf973fqzg6dgqlmZ:~$ sudo apt install docker.io
    ```

2. 安装docker compose

    大模型建议执行如下指令安装 `docker compose`：

    ```bash
    sudo apt update
    sudo apt install docker-compose-plugin
    ```

    但遇到“没有找到”的错误，说明你的系统（如 Ubuntu 或 Debian）没有配置 Docker 官方的软件源，导致 apt 找不到 docker-compose-plugin 这个包。解决方案是：添加 Docker 官方源（推荐）。这是标准的安装方法。请按顺序执行以下命令来添加源并安装：

    ```bash
    # 1. 更新系统包并安装工具
    
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg
    ```

    ```bash
    # 2. 添加 Docker 的官方 GPG 密钥和软件源
    
    # 添加密钥
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # 添加软件源
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

    ```bash
    # 3. 更新包列表并安装插件
    
    sudo apt-get update
    sudo apt-get install docker-compose-plugin
    # 安装成功后，运行 docker compose version 即可查看版本。
    ```

### 创建容器

1. 创建目录 `~/aihalo`，用于存放网站内容。

    ```bash
    mkdir ~/aihalo && cd ~/aihalo
    ```

    > **注意：**后续操作中，Halo 产生的所有数据都会保存在这个目录，请妥善保存。




2. 先拉取镜像

    在 ~/aihalo 目录中执行 sudo docker compose up，屏幕输出提示拉取镜像超时。因此先执行 sudo docker pull 拉取镜像到 ECS。

    执行以下命令拉取 Halo 镜像： 
    
    ```bash
    sudo docker pull registry.fit2cloud.com/halo/halo:2
    ```

    拉取 mysql 镜像时提示超时，应是 docker 镜像源相关。编辑（或新建） `/etc/docker/daemon.json`，复制如下国内镜像源信息到文件中。

    ```json
    {
      "registry-mirrors": [
        "https://docker.1ms.run",
        "https://docker.m.daocloud.io",
        "https://docker.1panel.live",
        "https://dockerproxy.net",
        "https://docker.xuanyuan.me",
        "https://docker-0.unsee.tech",
        "https://registry.cyou"
        ]
    }
    ```

    > 可以用普通用户比如 gdv2 执行 `sudo vim /etc/docker/daemon.json` 编辑（或新建）文件，或者用 VSCode 以 root 用户远程连接阿里云 ECS 后编辑文件。

    然后依次执行如下命令，重启 docker 并拉取 mysql 镜像，就可以了。

    ```bash
    # 重启docker
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    
    # 拉取 mysql 镜像
    sudo docker pull mysql:8.1.0
    ```



3. 创建 `docker-compose.yaml`

    参考 [Halo安装指南↗]，选择 Halo + MySQL。还有其他几种场景：Halo + PostgreSQL（推荐）、Halo + H2、使用外部数据库，等。

    在样例 yaml 文件上，做了以下几处修改：

    - 镜像地址。修改为：`image: registry.fit2cloud.com/halo/halo:2`。
    - 容器名称。新增了 2 处容器名称，`container_name: gdhalo_master` 和 `container_name: gdhalo_dbmysql`。
    - 数据库密码。root 密码修改为 `rootmdb8`。

    修改后的 `docker-compose.yaml` 保存在 `~/aihalo` 目录中，内容如下：

    ```yml
    # version: "3"
    
    services:
      halo:
        # image: registry.fit2cloud.com/halo/halo-pro:2.22
        image: registry.fit2cloud.com/halo/halo:2
        # image: halohub/halo:2
        container_name: aihalo_master # 增加了容器的名字
        restart: on-failure:3
        depends_on:
          halodb:
            condition: service_healthy
        networks:
          halo_network:
        volumes:
          - ./halo2:/root/.halo2
        ports:
          - "8090:8090"
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8090/actuator/health/readiness"]
          interval: 30s
          timeout: 5s
          retries: 5
          start_period: 30s
        environment:
          # JVM 参数，默认为 -Xmx256m -Xms256m，可以根据实际情况做调整，置空表示不添加 JVM 参数
          - JVM_OPTS=-Xmx256m -Xms256m
        command:
          - --spring.r2dbc.url=r2dbc:pool:mysql://halodb:3306/halo
          - --spring.r2dbc.username=root
          # MySQL 的密码，请保证与下方 MYSQL_ROOT_PASSWORD 的变量值一致。
          # - --spring.r2dbc.password=o#DwN&JSa56
          - --spring.r2dbc.password=rootmdb8
          - --spring.sql.init.platform=mysql
          # 外部访问地址，请根据实际需要修改
          - --halo.external-url=http://localhost:8090/
    
      halodb:
        image: mysql:8.1.0
        container_name: aihalo_dbmysql # 增加了容器的名字
        restart: on-failure:3
        networks:
          halo_network:
        command: 
          - --default-authentication-plugin=caching_sha2_password
          - --character-set-server=utf8mb4
          - --collation-server=utf8mb4_general_ci
          - --explicit_defaults_for_timestamp=true
        volumes:
          - ./mysql:/var/lib/mysql
          - ./mysqlBackup:/data/mysqlBackup
        healthcheck:
          test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "--silent"]
          interval: 3s
          retries: 5
          start_period: 30s
        environment:
          # 请修改此密码，并对应修改上方 Halo 服务的 SPRING_R2DBC_PASSWORD 变量值
          # - MYSQL_ROOT_PASSWORD=o#DwN&JSa56
          - MYSQL_ROOT_PASSWORD=rootmdb8
          - MYSQL_DATABASE=halo
    
    networks:
      halo_network:
    ```

4. 启动 Halo 服务

    在 ~/aihalo 目录下执行以下命令，可后台方式启动 Halo 服务（即关闭运行命令的终端程序， Halo服务也不会停止）：

    ```bash
    sudo docker compose up -d
    ```

    实时查看日志：
    ```bash
    sudo docker compose logs -f
    ```

<!--  -->
## 外部访问

**要开放 8090 端口后，才能被外部访问。** 按以下步骤操作：

1. 登录阿里云后，进入 **控制台**，选择 **云服务器ECS**，并 **切换至简捷版**。

2. 建议新建安全组（防火墙）。左侧导航栏中选择 **安全组**，并点击右侧页面的左上角的 **创建安全组** 按钮。

3. 在 [创建安全组] 页面做相关配置：

    [基本信息] 输入以下信息：
    
    |配置项|建议取值|
    |:---|:---|
    |安全组名称|*默认值即可，维持不变*|
    |网络|*默认值即可，维持不变*|
    |安全组类型|*默认值`普通安全组`即可，维持不变*|
    |高级配置|*暂不配置*|

    [规则配置]。点击 `增加规则` 按钮，输入以下信息：

    |配置项|建议取值|
    |:---|:---|
    |流量方向|选择 `入方向`|
    |授权策略|选择 `允许`|
    |优先级|*先维持默认值 `1` 不变*|
    |协议|选择 `自定义TCP`|
    |访问来源|选择 `IPv4` 和 `0.0.0.0/0(任何位置)`| 
    |访问目的(本实例)|选择 `端口`，输入 `8090`|
    |描述|输入比如 `halo 8090`|

    设置完成后，点底部的 **提交** 按钮。

4. ⚠确保新建的安全组，应用到了 ECS上。否则外部还是无法问问 ECS 的 8090 端口。

    - 点击左侧导航栏的 **安全组**，再点击刚新建的安全组
    - 点击 **实例列表**，选择 **云服务器ECS实例**
    - 点击 **实例加入安全组** 按钮，将 ECS（实例）加入安全组。

经过上述一系列操作后，最终可在本机浏览器输入 `http://8.152.199.10:8090/`（8.152.199.10 是 ECS 的公网 IP），访问 Halo 搭建的网站了。

## oauth 认证

参考 [Halo插件OAuth2↗] 安装 OAuth2 插件，并做相关配置。

（待续）

<!--  -->
[Halo安装指南↗]: https://docs.halo.run/getting-started/install/docker-compose
[Halo插件OAuth2↗]: https://www.halo.run/store/apps/app-ESVDK