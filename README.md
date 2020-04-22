# Docker redis2mysql crawler

## 功能
    该Docker爬虫镜像使用龙船网接口获取数据，先缓存到redis，后保存到mysql数据库中。本文档介绍服务器环境的配置，爬虫镜像的获取、配置以及运行方式。
## 需求
- 原有服务器数据丢失，并且服务器也需迁移，因此将AIS数据接口迁移到新的服务器。
- 考虑到docker的便捷性，使用docker部署爬虫，使用宿主机的mysql与redis。
## 前期工作
### 环境配置
在新服务器上需要安装Mysql，redis，和docker。

- 首先查看CentOS版本：

    ``` C++
    lsb_release -a
    CentOS Linux release 7.6.1810 (Core)
    ```

- 安装python3.6.8(docker部署不需)
    ``` C++
    # 调试需要，安装python3，并修改为默认python
    yum install epel-release
    yum install python36
    cd /usr/bin/ 
    rm python 
    ln -s python3.6 python
    ```

- mysql

    * 查看是否已安装mysql
        ```
        rpm  -qa | grep  mysql
        mysql-community-client-5.7.26-1.el7.x86_64
        ...
        mysql-community-libs-5.7.26-1.el7.x86_64
        ```
	* 登录管理员账户，新建用户
        ```
        CREATE USER 'aistest'@'%' IDENTIFIED BY 'user_password';
        ```
    * 授予权限：
        ```
        GRANT ALL PRIVILEGES ON *.* TO 'aistest'@'%' identified by 'user_password' with grant option;
        ```
    * 显示权限：
        ```
        SHOW GRANTS FOR 'aistest'@'%';	
        ```

- redis
    * 安装
        ```
        # 下载
        wget http://download.redis.io/releases/redis-4.0.6.tar.gz
        # 解压压缩包
        tar -zxvf redis-4.0.6.tar.gz
        mv redis-4.0.6/ /usr/local/
        # 安装依赖
        yum install gcc
        # 编译安装
        cd /usr/local/redis-4.0.6/
        make MALLOC=libc　
        cd src && make install
        ```
    * 配置redis
        ```
        # 移动redis conf文件到统一文件夹中，方便管理
        mkdir -p /usr/local/redis/bin
        mkdir -p /usr/local/redis/etc
        mv redis-4.0.6/redis.conf /usr/local/redis/etc
        cd redis-4.0.6/src
        mv mkreleasehdr.sh redis-benchmark.c 
            redis-benchmark.o 	redis-check-aof.c 
            redis-check-aof.o redis-check-dump.c 
        	redis-check-dump.o redis-cli.c 
            redis-cli.o redis-server 
            /usr/local/redis/bin
        vim /usr/local/redis/etc/redis.conf
        # 以守护进程运行
        daemonize yes
        # 设置密码
        requirepass 后修改密码，重新启动
        # 允许远程访问
        protected-mode no
        # 并注释掉  
        bind 127.0.0.1
        # 设置持久化文件存放位置
        dir 路径
        # 后台启动redis
        redis-server /usr/local/redis/etc/redis.conf
        # 查看redis版本
        redis-server -v
        # 开启一个客户端连接
        redis-cli -a 密码
        # redis 默认端口为 6379 
        # 添加开机启动
        vim /etc/rc.local 添加
        /usr/local/redis/bin/redis-server 
        /usr/local/redis/etc/redis.conf
        pkill redis  //停止redis
        kill -9 PID  //禁止忽略-停止redis
        ``` 
    * docker
        ```
        # 安装所需依赖包
        sudo yum install -y yum-utils device-mapper-persistent-data lvm2
        # 使用国内源
        sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
        # 更新yum缓存并安装Docker-ce
        sudo yum makecache fast
        sudo yum -y install docker-ce
        # 启动Docker-ce
        sudo systemctl enable docker
        sudo systemctl start docker
        ```
### 数据库
- 导出603服务器旧版本数据库，迁移到test服务器，方便修改（docker部署不需要此步骤）
    ```
    # 导出
    mysqldump -u root -p ship_info (table) > ship_info.txt
    # 导入
    mysql -u aistest -p ship_info < ship_info.txt
    ```
- 数据库版本不同（8->5.7），修改方法：

        打开sql文件，将文件中的所有
        utf8mb4_0900_ai_ci替换为utf8_general_ci
        utf8mb4替换为utf8
- 数据库表

| 脚本 | 功能 | 更新频率 | 数据库 | 
| :----:| :----: | :----: | :----: |
| getShipLastPosition.py | 获取最新船位，添加到数据库中，批量20条 | sh_ship_last_position |
| getShipHistoryPosition.py | 根据船舶ID、时间范围，获取前4个月的航运数据 | sh_ship_history_position |
| getShipLastLocation.py | 获取最新船位，更新到数据库中，批量20条 | sh_ship_location_latest |



## 镜像使用
    镜像构建细节详见Dockerfile
### image下载及配置
- pull爬虫镜像
    ```
    docker pull lauhub/shipinfo:tag # tag为具体的标签
    ```
- 运行镜像
    ```
    # 重命名为shipinfo，并在后台运行
    docker run --name shipinfo -d lauhub/shipinfo:tag
    ```
- 查看容器信息
    ```
    docker ps
    8bc6856ace2b        5e0aed3c386a        "/bin/sh -c 'sh /cra…" ···
    ```
- 容器内目录结构

    code ：存放爬虫脚本，定时任务文件
    
    Dockerfile ：构建image的控制文件
    
    README.md 
    
    requirements.txt ：爬虫需要的包信息
- 配置服务器信息
    ```
    # 宿主机查看docker网卡ip
    ifconfig
    # docker0 对应的inet即作为docker_ip
    docker exec -it 8bc6856ace2b(容器ID或容器自定义名称) /bin/bash
    # 修改config中的服务器参数：
        server_ip : 服务器公网ip
        docker_ip : docker对应的网卡ip
        mysql_pwd : mysql数据库密码
        redis_pwd : redis数据库密码
        mysql_user : mysql用户名
        mysqldb : mysql数据库名称
        redisdb : redis数据库号
    vim code/config.py
    # 修改定时任务
    vim code/crontabfile
    # 每条命令前5个参数分别为分钟，小时，天，月，周
    # 使定时任务生效
    crontab crontabfile
    # 查看定时任务
    crontab -l
    # 退出并重启容器
    exit
    docker restart 8bc6856ace2b(容器ID或容器自定义名称)
    ```

### 日志
    ```
    # 进入容器后,进入日志目录
    cd var/log
    tail -f example.log
    # 以下为日志对应的脚本：
    history.log <-- getShipHistoryPosition.py
    position.log <-- getShipLastPosition.py
    location.log <-- getShipLastLocation.py
    phistory.log <-- parse_redis.py HistoryPosition
    pposition.log <-- parse_redis.py LastLocation
    plocation.log <-- parse_redis.py LastLocation
    ```

## Docker常用命令
其他命令参数见[Docker 命令](https://www.runoob.com/docker/docker-command-manual.html)  
- 容器生命周期管理
    ```
    docker start CONTAINER : 启动一个或多个已经被停止的容器
    docker stop CONTAINER : 停止一个运行中的容器
    docker restart CONTAINER : 重启容器
    docker run --name myname -d IMAGE : 命名并后台运行镜像
    docker exec CONTAINER COMMAND : 在运行的容器中执行命令
    docker events : 从服务器获取实时事件
    docker commit CONTAINER name:tag : 保存容器为镜像
    docker info : 显示 Docker 系统信息
    ```
- 本地镜像管理
    ```
    docker images : 列出本地镜像
    docker ps : 列出容器
    docker rm CONTAINER : 删除一个或多个容器
    docker rmi : 删除本地一个或多个镜像
    docker tag  IMAGE:TAG hub/name:tag : 标记镜像并归入某一仓库
    docker save IMAGE : 将指定镜像保存成 tar 归档文件
    docker load : 导入使用 docker save 命令导出的镜像
    docker import file : 从归档文件中创建镜像
    # 批量删除镜像
    docker images|grep none|awk '{print $3 }'|xargs docker rmi
    ```
- 镜像仓库
    ```
    docker login : 登陆到一个Docker镜像仓库 
    docker logout : 登出一个Docker镜像仓库
    docker pull : 从镜像仓库中拉取或者更新指定镜像
    docker push : 将本地的镜像上传到镜像仓库
    ```
- 容器与宿主机文件交互
    ```
    从主机复制到容器sudo docker cp host_path containerID:container_path
    从容器复制到主机sudo docker cp containerID:container_path host_path
    ```
## 注意事项
- 设计定时任务时，尽量将不同脚本运行时间间隔开，减轻并发压力
- 目标船次更新后，及时在sh_target_ship表中更新


