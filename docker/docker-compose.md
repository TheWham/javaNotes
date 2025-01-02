## DockerCompose

### dockerCompose书写规范

![img](D:\桌面文件\ALL\java pictures\docker\dockerCompose.jpg)

部署一个简单的java项目，其中包含3个容器：

- MySQL
- Nginx
- Java项目

而稍微复杂的项目，其中还会有各种各样的其它中间件，需要部署的东西远不止3个。如果还像之前那样手动的逐一部署，就太麻烦了。

而Docker Compose就可以帮助我们实现**多个相互关联的Docker容器的快速部署**。它允许用户通过一个单独的 docker-compose.yml 模板文件（YAML 格式）来定义一组相关联的应用容器。

* 正常部署mysql命令为

  ```bash
  docker run -d \
    --name mysql \
    -p 3306:3306 \
    -e TZ=Asia/Shanghai \
    -e MYSQL_ROOT_PASSWORD=123 \
    -v ./mysql/data:/var/lib/mysql \
    -v ./mysql/conf:/etc/mysql/conf.d \
    -v ./mysql/init:/docker-entrypoint-initdb.d \
    --network hmall
    mysql
  ```

* 用docker-compose.yml为

  ```yaml
  version: "3.8"					# 版本
  
  services:
    mysql:
      image: mysql     			# 镜像
      container_name: mysql		# 容器名称
      ports:						# 端口号映射
        - "3306:3306"
      environment:				# 环境设置
        TZ: Asia/Shanghai			# 时区设置
        MYSQL_ROOT_PASSWORD: 123  # 登录密码
      volumes:					# 挂在卷设置
        - "./mysql/conf:/etc/mysql/conf.d"
        - "./mysql/data:/var/lib/mysql"
      networks:					# mysql网路设置
        - new
  networks:						# 整个网络设置
    new:
      name: hmall					# 网络名称
  ```

部署java项目,包括mysql,nginx

> 在 `docker-compose.yml` 文件中，`build: context` 的作用是**指定 Docker 构建镜像时的构建上下文目录**。构建上下文是 Docker 构建镜像时可以访问的所有文件的根目录，Docker 引擎只能访问在这个目录及其子目录中的文件。
>
> ### **关键点说明**
>
> #### 1. **`context` 的作用**
>
> - **定义范围**：限制 Docker 构建镜像时可以使用的文件范围。
> - **指定位置**：`context` 通常是包含 Dockerfile 和其他依赖文件的目录，Docker 会把这些内容上传到 Docker 引擎用于镜像构建。

```yaml
version: "3.8"

services:
  mysql:						
    image: mysql
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: 123
    volumes:		
      - "./mysql/conf:/etc/mysql/conf.d"			
      - "./mysql/data:/var/lib/mysql"
      - "./mysql/init:/docker-entrypoint-initdb.d"
    networks:
      - hm-net
  hmall:									 # java项目构建镜像步骤
    build: 			
      context: /root/hmall					 # 指定构建目录 注意这里指定的目录将作为Dockerfile中xxx.jar包的根目录
      dockerfile: /root/hmall/Dockerfile  	 # dockerfile文件	
    container_name: hmall					 # 容器名称
    ports:						
      - "8080:8080"
    networks:
      - hm-net
    depends_on:								 # 依赖容器, 需要先创建
      - mysql
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "18080:18080"
      - "18081:18081"
    volumes:
      - "./nginx/nginx.conf:/etc/nginx/nginx.conf"
      - "./nginx/html:/usr/share/nginx/html"
    depends_on:
      - hmall
    networks:
      - hm-net
networks:									 # 网络部署需要一致
  hm-net:
    name: hmall		
```

### dockerCompose运行命令

<img src="D:\桌面文件\ALL\java pictures\docker\dockerCompose 命令.jpg" alt="img" style="zoom: 50%;" />

基本语法如下：

```Bash
docker compose [OPTIONS] [COMMAND]
```

其中，OPTIONS和COMMAND都是可选参数，比较常见的有：

| **类型** | **参数或指令**                                               | **说明**                    |
| :------- | :----------------------------------------------------------- | :-------------------------- |
| Options  | -f                                                           | 指定compose文件的路径和名称 |
| -p       | 指定project名称。project就是当前compose文件中设置的多个service的集合，是逻辑概念 |                             |
| Commands | up                                                           | 创建并启动所有service容器   |
| down     | 停止并移除所有容器、网络                                     |                             |
| ps       | 列出所有启动的容器                                           |                             |
| logs     | 查看指定容器的日志                                           |                             |
| stop     | 停止容器                                                     |                             |
| start    | 启动容器                                                     |                             |
| restart  | 重启容器                                                     |                             |
| top      | 查看运行的进程                                               |                             |
| exec     | 在指定的运行中容器中执行命令                                 |                             |

```bash
# 1.进入root目录
cd /root

# 2.删除旧容器
docker rm -f $(docker ps -qa)

# 3.删除hmall镜像
docker rmi hmall

# 4.清空MySQL数据
rm -rf mysql/data

# 5.启动所有, -d 参数是后台启动 如果docker-compose.yml文件在当前目录则不需要添加 -f 参数
docker compose up -d
# 结果：
[+] Building 15.5s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                    0.0s
 => => transferring dockerfile: 358B                                                    0.0s
 => [internal] load .dockerignore                                                       0.0s
 => => transferring context: 2B                                                         0.0s
 => [internal] load metadata for docker.io/library/openjdk:11.0-jre-buster             15.4s
 => [1/3] FROM docker.io/library/openjdk:11.0-jre-buster@sha256:3546a17e6fb4ff4fa681c3  0.0s
 => [internal] load build context                                                       0.0s
 => => transferring context: 98B                                                        0.0s
 => CACHED [2/3] RUN ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo   0.0s
 => CACHED [3/3] COPY hm-service.jar /app.jar                                           0.0s
 => exporting to image                                                                  0.0s
 => => exporting layers                                                                 0.0s
 => => writing image sha256:32eebee16acde22550232f2eb80c69d2ce813ed099640e4cfed2193f71  0.0s
 => => naming to docker.io/library/root-hmall                                           0.0s
[+] Running 4/4
 ✔ Network hmall    Created                                                             0.2s
 ✔ Container mysql  Started                                                             0.5s
 ✔ Container hmall  Started                                                             0.9s
 ✔ Container nginx  Started                                                             1.5s

# 6.查看镜像
docker compose images	
# 结果
CONTAINER           REPOSITORY          TAG                 IMAGE ID            SIZE
hmall               root-hmall          latest              32eebee16acd        362MB
mysql               mysql               latest              3218b38490ce        516MB
nginx               nginx               latest              605c77e624dd        141MB

# 7.查看容器
docker compose ps
# 结果
NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS              PORTS
hmall               root-hmall          "java -jar /app.jar"     hmall               54 seconds ago      Up 52 seconds       0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
mysql               mysql               "docker-entrypoint.s…"   mysql               54 seconds ago      Up 53 seconds       0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp
nginx               nginx               "/docker-entrypoint.…"   nginx               54 seconds ago      Up 52 seconds       80/tcp, 0.0.0.0:18080-18081->18080-18081/tcp, :::18080-18081->18080-18081/tcp
```

