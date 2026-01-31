---

---

# Halo 建站

`更新：26-01-30` \| `发布：26-01-26`

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
        container_name: gdhalo_master # 增加了容器的名字
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

    执行以下命令启动 Halo， 并以后台方式运行（即关闭终端所在窗口， Halo也不会停止）：

    ```bash
    ~/gdhalo % docker compose up -d
    ```

    如需要实时查看 Halo 的日志，可执行如下命令：
    ```bash
    docker-compose logs -f
    ```

<!--  -->
## 初始化

如果是采用上述默认的 docker-compose.yml 启动 Halo 的，则在本机浏览器访问本地Halo地址 [^2]，就会出现的 `Halo 系统初始化` 页面。参考文档 [Halo初始化]，填写相关信息，初始化 Halo 网站。

- **语言**。选择 `简体中文`。
- **外部访问地址**。维持 http://localhost:8090/ 不变。
- **站点标题**：网站的名称，将会显示在浏览器标签页上。输入 `AI学院`。
- **用户名**：初始管理员的用户名。输入自己的用户名比如 `gdv2`。
- **电子邮箱**：初始管理员的邮箱地址。输入自己的邮箱比如 `georgedonnev2@outlook.com`。
- **密码** / **重复密码**。输入密码，并重复输入密码。

初始化完成后，会弹出登录页面。输入用户名和密码登录后，就可以使用 Halo 网站了。

## 安装插件

初始化完成后，用超级管理员安装必要的插件，让网站可用或更方便使用。要安装的插件有：

1. 插件#1：`Vditor编辑器`。使用该编辑器，可以编写 markdown 文档来生成网站的文章。默认编辑器不能编写 markdown 文档。
2. 插件#2：`文章导入导出`。
3. 插件#3：`内容助手`。

还有几个插件，是默认主题/初始化时已经安装的，不用额外再安装。


<!--  -->
## 修改菜单

在本机浏览器中输入学院内部网站地址 [^1] 访问网站，然后按以下步骤操作：

1. 点击网站右上角 <img src="./halo.assets/default-avatar.svg" style="width:1.25em;"> 图标，再点 **登录**。以`超级管理员`（或有超级管理员权限的用户）登录。

2. 登录后，再点右上角图标，再点 **控制台**。

3. 点击左侧导航栏的 **菜单** 后，可对菜单做修改。

### 设置主菜单

参考以下步骤设置主菜单。

1. 点击左侧导航栏的 **菜单** 后，页面显示为左、中、右三列。点击中列底部的 **新建** 按钮新建一个菜单，输入如下信息：

    - 菜单名称：输入`导航`，然后按底部的 [提交] 按钮。

    <br>
    > 菜单名称取做 `导航`，是为了复用到页脚的菜单组中，会出现 `导航` 字样。主菜单则不会出现 `导航` 字样。如果不复用到页脚的菜单组，则菜单名称随意取即可。
    
    > 以主题 Earth 1.14.0 为例。用超级管理员登录后进入控制台，然后点击左侧导航栏的 **主题**，在右侧顶部找到 **页脚**，在页脚页面可选择 **菜单组**。 

2. 将 `导航` 菜单设置为主菜单。回到中列，点击菜单 `导航` 右边的 [...] 按钮，点击出现的 **设置为主菜单**。

3. 新建菜单项。点击中列的 `导航` 菜单，再点击右列的右上角的 **新建** 按钮，增加菜单项到 导航 菜单中。重复多次操作，依次输入以下信息：

    |上级菜单项|类型|名称|链接地址|打开方式|
    |:---|:---|:---|:---|:---|
    |-|自定义链接|`首页`|/|当前窗口|

    |上级菜单项|类型|分类|打开方式|
    |:---|:---|:---|:---|
    |-|分类|`公告`|当前窗口|
    |-|分类|`就业`|当前窗口|
    |就业|分类|`入职招聘`|当前窗口|
    |就业|分类|`实习岗位`|当前窗口|
    |-|分类|`手册`|当前窗口|
    |-|分类|`笔记`|当前窗口|

    |上级菜单项|类型|自定义页面|打开方式|
    |:---|:---|:---|:---|
    |-|自定义页面|`关于`|当前窗口|

    > 分类。需要参考 [Halo文章] 之 “文章标签管理”先定义好，才能在菜单定义时被选择到。

    > 页面。需要参考 [Halo页面] 先定义好，才能在菜单定义时被选择到。


### 设置友情链接

和上述“设置主菜单”类似，新建菜单“友情链接”。

1. 点击左侧导航栏的 **菜单** 后，页面显示为左、中、右三列。点击中列底部的 **新建** 按钮新建一个菜单，输入如下信息：

    - 菜单名称：输入`友情链接`，然后按底部的 [提交] 按钮。

2. 新建菜单项。点击中列的 `友情链接` 菜单，再点击右列的右上角的 **新建** 按钮，增加菜单项到 导航 菜单中。重复多次操作，依次输入以下信息：

    |上级菜单项|类型|名称|链接地址|打开方式|
    |:---|:---|:---|:---|:---|
    |-|自定义链接|`江南大学`|`https://www.jiangnan.edu.cn/`|新窗口|
    |-|自定义链接|`AI学院`|`https://ai.jiangnan.edu.cn/`|新窗口|


<!--  -->
## 站点设置

访问网站，用超级管理员（或有超级管理员权限的用户）登录，进入控制台，点击左侧导航栏的 **设置**，在设置页面做如下改动：

1. 基本设置

    - 站点标题。输入 `AI学院`。
    - 站点副标题。输入 `智能学院祝全体师生春节快乐！幸福安康！`。后续考虑经常修改。
    - Logo。可以参考 [Halo附件] 之“上传附件”，然后从附件中选取。
    - Favicon。可以参考 [Halo附件] 之“上传附件”，然后从附件中选取。
    - 首选语言。选 `简体中文`。

    上述修改完成后，点击底部的 **保存** 按钮。

2. 文章设置
    
    先维持不变。

3. SEO设置
    
    先维持不变。

4. 用户设置
    
    - 开放注册。先不勾选，即不开放注册。后续要和学校的统一鉴权认证对接。
    - 其他。先维持不变。

5. 附件配置
    
    先维持不变。

6. 评论设置
    
    - 启动评论。要勾选，即允许评论。
    - 新评论审核。要勾选，即新评论需要管理员审核后才会显示。
    - 仅允许注册用户评论。不勾选，即所有用户都可以评论。

7. 主题路由设置
    
    先维持不变。

8. 代码注入
    
    后续再配置。

9. 通知设置
    
    后续再启用邮件通知器。

<!--  -->
## 主题设置

以主题 Earth 1.14.0 为例。用超级管理员登录后进入控制台，然后点击左侧导航栏的 **主题**，对以下内容做设置：

1. 布局

    - 文章列表布局：选 `网格（一行三列）`。
    - 首页顶部模块：选 `站点标题`。
    - 首页顶部背景：选 `图片`。
    - 首页顶部背景图片：可以参考 [Halo附件] 之“上传附件”，然后从附件中选取。
    - 标题文字颜色。参考背景图片的颜色做相应调整。
    - 显示文章页面顶部背景。暂未清楚效果，先勾选。

2. 全局

    - 菜单栏 Logo 类型。选择 `图片`，网站左上角会显示网站 Logo。
    - 显示滚动到顶部按钮。要勾选。勾选后，在网站页面右下角显示浮动的圆形按钮，点击后可回到网站页面的顶部。

3. 样式

    - 默认配色：选 `跟随系统`。
    - 允许访客切换配色。不重要，先勾选吧。

4. 文章

    修改后未见效果。先维持默认配置。

5. 侧边栏

    是网页右边的侧边栏。把可以呈现的 4 个小部件，都显示了。

    - 热门文章。放在第 1 个。
    - 文章分类。放在第 2 个。
    - 文章标签。放在第 3 个。
    - 站点资料。放在第 4 个。
        - 站点资料 Logo。后续再加。
        - 社交媒体。先展现了学院的官微图片。

6. 页脚

    - 页脚风格：选 `风格二`。
    - Logo。暂不设置。
    - 标题。暂不设置。
    - 标语。随意写一些，比如 `欢迎访问学院内部网站`。
    - 菜单组。可多选。先选 `导航`，再选 `友情链接`。
    - 社交媒体。先展现了学院的官微图片。

    <br>
    
    > 菜单组 `导航`。就是本文 [设置主菜单](#设置主菜单) 中定义的主菜单。
    
    > 菜单组 `友情链接`。就是本文 [设置友情链接](#设置友情链接) 中定义的菜单。

7. 备案设置

    今后再设置。

8. 插件集成

    今后再设置。

<!--  -->
## 发布文章

如果只是浏览学院内部网站，在浏览器里输入学院内部网站地址 [^1] 后即可浏览。如果要发布文章，则还需按以下步骤操作：

1. 点击网站右上角 <img src="./halo.assets/default-avatar.svg" style="width:1.25em;"> 图标，再点 **登录**。

2. 登录后，再点右上角图标，再点 **控制台**。

3. 点击左侧导航栏的 **文章**，再点击右上角 **新建** 按钮，就可以新建文章了。

接下来可参考 Halo 官网文档 [Halo文章] 来操作。有几点说明如下：

- 官网文档描述有 4 个按钮：编辑器切换，预览，保存，发布。**预览** 已不可见，其他 3 个还在。新文章保存后，还会出现第 4 个按钮 **设置**。
- 默认编辑器（点击 **编辑器切换** 选择）。可以像 Word 那样编辑文章，但不能编写 markdown 文档。
- Vditor编辑器（点击 **编辑器切换** 选择）。可以编辑 markdown 文档。
- 如果是使用 Vditor编辑器 编辑 markdown 格式生成文章，发布前请先保存（保存后出现设置按钮），然后点击 **设置** 按钮，在 **标题** 输出文章的标题，否则默认为“未命名文章”。使用默认编辑器则不存在此注意事项。


## 遗留事宜

有以下遗留事宜，待后续跟进：

- 修改了 docker-compose.yml 中 的 MySQL 的密码，`docker compose up` 重启时不成功。把密码改回来再重启，是可以成功的。期望密码修改后重启可以成功。

- 发布的文章，保存在哪个目录了？当前查看本机的目录，没有发现。

- 站点 Logo。如何适应浅色和深色。网站切换到深色模式后，站点 Logo 不容易被看清。


<!--  -->
[Halo安装指南]: https://docs.halo.run/getting-started/install/docker-compose/
[Halo初始化]: https://docs.halo.run/getting-started/setup
[Halo文章]: https://docs.halo.run/user-guide/posts
[Halo页面]: https://docs.halo.run/user-guide/pages
[Halo附件]: https://docs.halo.run/user-guide/attachments/

<!--  -->
## 参考信息
[^1]: 学院内部网站地址 https://aicss.jiangnan.edu.cn
[^2]: 网站本机地址 http://localhost:8090
