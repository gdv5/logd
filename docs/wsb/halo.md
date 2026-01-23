# Halo 建站

本文档描述如何用 Halo 建站，供感兴趣者参考。

## 安装

### 环境说明

- 安装过程，主要参考了 Halo 的 [安装指南]，使用 Docker Compose 部署。
- 版本选择社区版，镜像地址：`registry.fit2cloud.com/halo/halo`。
- 操作环境是 MacOS。

### 创建容器

1. 新建文件夹，此文档以 `~/gdhalo` 为例。

```bash
mkdir ~/gdhalo && cd ~/gdhalo
```

> **注意：**后续操作中，Halo 产生的所有数据都会保存在这个目录，请妥善保存。

2. 创建 docker-compose.yml

    选择了 Halo + MySQL。从安装指南复制了样例文档，并修改 2 处：镜像地址，和 MySQL 的 root 密码。修改后的文件保存在 `~/gdhalo` 目录中。文件内容如下：

    ```yml
    version: "3"

    services:
      halo:
        # image: registry.fit2cloud.com/halo/halo-pro:2.22
        image: registry.fit2cloud.com/halo/halo:2
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
          test: ["CMD", "curl", "-f", "http://localhost:8090/    actuator/health/readiness"]
          interval: 30s
          timeout: 5s
          retries: 5
          start_period: 30s
        environment:
          # JVM 参数，默认为 -Xmx256m -Xms256m，可以根据实际情况做    调整，置空表示不添加 JVM 参数
          - JVM_OPTS=-Xmx256m -Xms256m
        command:
          - --spring.r2dbc.url=r2dbc:pool:mysql://halodb:3306/halo
          - --spring.r2dbc.username=root
          # MySQL 的密码，请保证与下方 MYSQL_ROOT_PASSWORD 的变量值    一致。
          # - --spring.r2dbc.password=o#DwN&JSa56
          - --spring.r2dbc.password=rootmd8
          - --spring.sql.init.platform=mysql
          # 外部访问地址，请根据实际需要修改
          - --halo.external-url=http://localhost:8090/
    
      halodb:
        image: mysql:8.1.0
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
          test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1",     "--silent"]
          interval: 3s
          retries: 5
          start_period: 30s
        environment:
          # 请修改此密码，并对应修改上方 Halo 服务的     SPRING_R2DBC_PASSWORD 变量值
          # - MYSQL_ROOT_PASSWORD=o#DwN&JSa56
          - MYSQL_ROOT_PASSWORD=rootmd8
          - MYSQL_DATABASE=halo
    
    networks:
      halo_network:
    ```

3. 启动 Halo 服务

```bash
~/gdhalo % docker compose up
```

可以看到屏幕输出很多信息，并最终启动成功。

> 运行调测正常后，要执行 `docker compose up -d` 命令以后台方式运行。






[安装指南]: https://docs.halo.run/getting-started/install/docker-compose/