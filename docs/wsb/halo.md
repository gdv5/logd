# Halo 建站

本文档描述如何用 Halo 建站，供感兴趣者参考。

## 安装

### 环境说明

- 安装过程，主要参考了 [Halo安装指南]，使用 Docker Compose 部署。
- 版本选择社区版，镜像地址：`registry.fit2cloud.com/halo/halo`。
- 操作环境是 MacOS。后续补充 Linux 环境（目标是在 Ubuntu 搭建学院用内部网站）。

### 创建容器

1. 新建文件夹，此文档以 `~/gdhalo` 为例。

    ```bash
    mkdir ~/gdhalo && cd ~/gdhalo
    ```

    > **注意：**后续操作中，Halo 产生的所有数据都会保存在这个目录，请妥善保存。

2. 创建 docker-compose.yml

    参考 [Halo安装指南]，选择了 Halo + MySQL。还有 “Halo + PostgreSQL（推荐）”、“Halo + H2”等。
    
    从安装指南复制样例文档，并修改 2 个数值：
    - 镜像地址。修改为 `image: registry.fit2cloud.com/halo/halo:2`。
    - MySQL 的 root 密码。修改为 `rootmdb8`。
    
    修改后的文件保存在 `~/gdhalo` 目录中。文件内容如下：

    ```yml
    version: "3"
    
    services:
      halo:
        # image: registry.fit2cloud.com/halo/halo-pro:2.22
        image: registry.fit2cloud.com/halo/halo:2
        container_name: gdhalo_master
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
        container_name: gdhalo_dbmysql
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

3. 启动 Halo 服务

    执行以下命令启动 Halo：

    ```bash
    ~/gdhalo % docker compose up -d
    ```

    实时查看日志：
    ```bash
    docker-compose logs -f
    ```

    可以看到屏幕输出很多信息，并最终启动成功。

### 遗留事宜

有以下遗留事宜，待后续跟进：

- 修改了 docker-compose.yml 中 的 MySQL 的密码，`docker compose up` 重启时不成功。把密码改回来再重启，是可以成功的。期望密码修改后重启可以成功。

<!--  -->
## 初始化

参考 docker-compose.yml 中的 `- --halo.external-url=http://localhost:8090/`，在本机浏览器访问 `http://localhost:8090/`。在出现的 `Halo 系统初始化` 页面，参考文档 [Halo初始化]，填写相关信息，初始化 Halo 网站。

- **语言**。选择 `简体中文`。
- **外部访问地址**。维持 http://localhost:8090/ 不变。
- **站点标题**：网站的名称，将会显示在浏览器标签页上。输入 `人工智能与计算机学院（软件学院）（内部）`。
- **用户名**：初始管理员的用户名。输入自己的用户名比如 `gdv2`。
- **电子邮箱**：初始管理员的邮箱地址。输入自己的邮箱比如 `georgedonnev2@outlook.com`。
- **密码** / **重复密码**。输入密码，并重复输入密码。

初始化完成后，会弹出登录页面。输入用户名和密码登录后，就可以使用 Halo 网站了。


[Halo安装指南]: https://docs.halo.run/getting-started/install/docker-compose/
[Halo初始化]: https://docs.halo.run/getting-started/setup