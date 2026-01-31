---
title: 在 Ubuntu 环境中基于 Halo 开源工具的静态网站建站
---

# Halo 建站（Ubuntu）

本文描述在 Ubuntu 环境中基于 Halo 开源工具的静态网站建站过程。

## 安装Halo

**环境说明**

- 安装过程，主要参考了 [Halo安装指南↗]，使用 Docker Compose 部署。
- 版本选择社区版，镜像地址：`registry.fit2cloud.com/halo/halo`。
- 操作环境是 Ubuntu（`Ubuntu 22.04.5 LTS`），目标是在 Ubuntu 搭建学院用内部网站。
- 登录内部服务器：
    - IP地址：`172.18.144.14`
    - 账号：`waicss`

**创建容器**

1. 创建目录 `~/aihalo`，用于存放网站内容。

    ```bash
    mkdir ~/aihalo && cd ~/aihalo
    ```

    > **注意：**后续操作中，Halo 产生的所有数据都会保存在这个目录，请妥善保存。

2. 创建 `docker-compose.yaml`

    参考 [Halo安装指南↗], 选择 Halo + MySQL。还有其他几种场景：Halo + PostgreSQL（推荐）、Halo + H2、使用外部数据库，等。

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

3. 拉取 Halo 镜像

    由于服务器网络环境等原因，无法直接拉取 Halo 镜像。采取了变通方式，即从 Mac 笔记本上导出已下载的镜像的 tar 包，然后上传 tar 包到服务器上，然后在服务器上导入 tar 包。依次执行如下命令：

    在 Mac 笔记本上先执行如下操作：

    ```bash
    # Mac 笔记本已有镜像，因此跳过拉取操作。
    # docker pull registry.fit2cloud.com/halo/halo:2
    
    docker save -o halo.tar registry.fit2cloud.com/halo/halo:2
    ```

    docker save 导出镜像为 tar 包完成后，在 Mac 笔记本上执行 scp 命令，上传 tar 包到服务器上：

    ```bash
    ~/tmp2601 % scp halo.tar gdv2@172.18.144.14:/home1/gdv2/tmp2601
    ```

    

3. 启动 Halo 服务

    在 `~/aihalo` 目录执行如下指令启动 Halo 服务：

    ```bash
    docker compose up -d
    ```






<!--  -->
[Halo安装指南↗]: https://docs.halo.run/getting-started/install/docker-compose

<!--  -->
- api发布文章。https://github.com/halo-sigs/vscode-extension-halo/blob/main/README.zh-CN.md
- oauth2相关。https://github.com/halo-sigs/plugin-oauth2/issues?q=is%3Aissue+sort%3Aupdated-desc+is%3Aclosed
