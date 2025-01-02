## docker基本命令

![img](D:\桌面文件\ALL\java pictures\docker\常见命令.jpg)

| **命令**       | **说明**                       | **文档地址**                                                 |
| :------------- | :----------------------------- | :----------------------------------------------------------- |
| docker pull    | 拉取镜像                       | [docker pull](https://docs.docker.com/engine/reference/commandline/pull/) |
| docker push    | 推送镜像到DockerRegistry       | [docker push](https://docs.docker.com/engine/reference/commandline/push/) |
| docker images  | 查看本地镜像                   | [docker images](https://docs.docker.com/engine/reference/commandline/images/) |
| docker rmi     | 删除本地镜像                   | [docker rmi](https://docs.docker.com/engine/reference/commandline/rmi/) |
| docker run     | 创建并运行容器（不能重复创建） | [docker run](https://docs.docker.com/engine/reference/commandline/run/) |
| docker stop    | 停止指定容器                   | [docker stop](https://docs.docker.com/engine/reference/commandline/stop/) |
| docker start   | 启动指定容器                   | [docker start](https://docs.docker.com/engine/reference/commandline/start/) |
| docker restart | 重新启动容器                   | [docker restart](https://docs.docker.com/engine/reference/commandline/restart/) |
| docker rm      | 删除指定容器                   | [docs.docker.com](https://docs.docker.com/engine/reference/commandline/rm/) |
| docker ps      | 查看容器                       | [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) |
| docker logs    | 查看容器运行日志               | [docker logs](https://docs.docker.com/engine/reference/commandline/logs/) |
| docker exec    | 进入容器                       | [docker exec](https://docs.docker.com/engine/reference/commandline/exec/) |
| docker save    | 保存镜像到本地压缩文件         | [docker save](https://docs.docker.com/engine/reference/commandline/save/) |
| docker load    | 加载本地压缩文件到镜像         | [docker load](https://docs.docker.com/engine/reference/commandline/load/) |
| docker inspect | 查看容器详细信息               | [docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/) |

* `docker ps` **命令拓展**: `docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"` 这个命令可以使docker容器信息格式化便于查看

* `docker run`中` -p` 为端口映射就是通过宿主机端口跟容器进行`端口映射`, 这样就可以开启多个容器

  <img src="D:\桌面文件\ALL\java pictures\docker\docker mysql 命令解读.jpg" alt="img" style="zoom: 50%;" />

* **docker开机自启**

  ```yml
  # Docker开机自启
  systemctl enable docker
  
  # Docker容器开机自启
  docker update --restart=always [容器名/容器id]
  ```

* **演示**

  ```yml
  # 第1步，去DockerHub查看nginx镜像仓库及相关信息
  
  # 第2步，拉取Nginx镜像
  docker pull nginx
  
  # 第3步，查看镜像
  docker images
  # 结果如下：
  REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
  nginx        latest    605c77e624dd   16 months ago   141MB
  mysql        latest    3218b38490ce   17 months ago   516MB
  
  # 第4步，创建并允许Nginx容器
  docker run -d --name nginx -p 80:80 nginx
  
  # 第5步，查看运行中容器
  docker ps
  # 也可以加格式化方式访问，格式会更加清爽
  docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
  
  # 第6步，访问网页，地址：http://虚拟机地址
  
  # 第7步，停止容器
  docker stop nginx
  
  # 第8步，查看所有容器
  docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
  
  # 第9步，再次启动nginx容器
  docker start nginx
  
  # 第10步，再次查看容器
  docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"
  
  # 第11步，查看容器详细信息
  docker inspect nginx
  
  # 第12步，进入容器,查看容器内目录
  docker exec -it nginx bash
  # 或者，可以进入MySQL
  docker exec -it mysql mysql -uroot -p
  
  # 第13步，删除容器
  docker rm nginx
  # 发现无法删除，因为容器运行中，强制删除容器
  docker rm -f nginx
  
  #查看nginx运行日志
  docker logs nginx
  ```

* **打包与加载**

  ```yml
  # 如果像将下载的镜像打包
  # docker save [OPTIONS] IMAGE [IMAGE...]
  # -o 表示要压缩到的文件 nginx:latest 表示nginx最新版本
  docker save -o nginx.tar nginx:latest
  
  # 加载打包好的镜像
  # docker load [OPTIONS]
  # -i 要加载的压缩文件  -q 不显示压缩过程
  docker load -i -q nginx.tar
  ```

* **起别名**

  > 如果觉得docker某些命令太长,可以给它起别名方便记忆与调用

  ```yml
  # 修改/root/.bashrc文件
  vi /root/.bashrc
  内容如下：
  # .bashrc
  
  # User specific aliases and functions
  
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  alias dpf='docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"'
  alias dis='docker images'
  
  # Source global definitions
  if [ -f /etc/bashrc ]; then
          . /etc/bashrc
  fi
  ```

  

