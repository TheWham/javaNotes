## docker数据卷

### 什么是数据卷

![img](D:\桌面文件\ALL\java pictures\docker\数据卷.jpg)

> 数据卷是一个虚拟目录, 是docker容器内目录与宿主机目录之间的映射.
>
> 以Nginx为例，我们知道Nginx中有两个关键的目录：
>
> - `html`：放置一些静态资源
> - `conf`：放置配置文件
>
> 现在假如需要更改html中的静态资源, 一般来说可以直接进入docker容器中在相应的目录中更改, 但是docker容器只提供了镜像运行的基本环境和命令没有提供其他可编译操作.所以需要数据卷来将容器中的目录跟宿主机映射起来双向绑定. 这样只需在宿主机中操作即可.

### 数据卷常见命令

数据卷的相关命令有：

| **命令**              | **说明**             | **文档地址**                                                 |
| :-------------------- | :------------------- | :----------------------------------------------------------- |
| docker volume create  | 创建数据卷           | [docker volume create](https://docs.docker.com/engine/reference/commandline/volume_create/) |
| docker volume ls      | 查看所有数据卷       | [docs.docker.com](https://docs.docker.com/engine/reference/commandline/volume_ls/) |
| docker volume rm      | 删除指定数据卷       | [docs.docker.com](https://docs.docker.com/engine/reference/commandline/volume_prune/) |
| docker volume inspect | 查看某个数据卷的详情 | [docs.docker.com](https://docs.docker.com/engine/reference/commandline/volume_inspect/) |
| docker volume prune   | 清除数据卷           | [docker volume prune](https://docs.docker.com/engine/reference/commandline/volume_prune/) |

* **演示**

  ```yml
  # 以挂载nginx为例
  
  # 注意挂载必须在创建容器时候挂载才能生效, 已经创建的未挂载的容器不能进行挂载
  # 这里的/usr/share/nginx/html为容器中的nginx中的html目录 -v必须要加 后面的html为挂在卷的名称
  # 挂载卷位置在一般在宿主机/var/lib/docker/volumes/html/_data
  # 容器中html的内容跟_data中内容双向绑定
  docker run -d -p 80:80 -v html:/usr/share/nginx/html nginx
  
  
  # 2.然后查看数据卷
  docker volume ls
  # 结果
  DRIVER    VOLUME NAME
  local     29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f
  local     html
  
  # 3.查看数据卷详情
  docker volume inspect html
  # 结果
  [
      {
          "CreatedAt": "2024-05-17T19:57:08+08:00",
          "Driver": "local",
          "Labels": null,
          "Mountpoint": "/var/lib/docker/volumes/html/_data",
          "Name": "html",
          "Options": null,
          "Scope": "local"
      }
  ]
  
  # 4.查看/var/lib/docker/volumes/html/_data目录
  ll /var/lib/docker/volumes/html/_data
  # 可以看到与nginx的html目录内容一样，结果如下：
  总用量 8
  -rw-r--r--. 1 root root 497 12月 28 2021 50x.html
  -rw-r--r--. 1 root root 615 12月 28 2021 index.html
  
  # 5.进入该目录，并随意修改index.html内容
  cd /var/lib/docker/volumes/html/_data
  vi index.html
  
  # 6.打开页面，查看效果
  
  # 7.进入容器内部，查看/usr/share/nginx/html目录内的文件是否变化
  docker exec -it nginx bash
  
  ```

* **演示mysql匿名数据卷**

  > 如果你查看数据卷时候发现有一串长的字符串, 那就是匿名数据卷id

  ```yml
  # 1.查看MySQL容器详细信息
  docker inspect mysql
  # 关注其中.Config.Volumes部分和.Mounts部分
  ```

  **我们关注两部分内容，第一是`.Config.Volumes`部分：**

  ```yml
  {
    "Config": {
      // ... 略
      "Volumes": {
        "/var/lib/mysql": {}
      }
      // ... 略
    }
  }
  ```

  **可以发现这个容器声明了一个本地目录，需要挂载数据卷，但是数据卷未定义。这就是匿名卷。**

  **然后，我们再看结果中的`.Mounts`部分：**

  ```yml
  {
    "Mounts": [
      {
        "Type": "volume",
        "Name": "29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f",
        "Source": "/var/lib/docker/volumes/29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f/_data",
        "Destination": "/var/lib/mysql",
        "Driver": "local",
      }
    ]
  }
  ```

  可以发现，其中有几个关键属性：

  - Name：数据卷名称。由于定义容器未设置容器名，这里的就是匿名卷自动生成的名字，一串hash值。
  - Source：宿主机目录
  - Destination : 容器内的目录

  上述配置是将容器内的`/var/lib/mysql`这个目录，与数据卷`29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f`挂载。于是在宿主机中就有了`/var/lib/docker/volumes/29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f/_data`这个目录。这就是匿名数据卷对应的目录，其使用方式与普通数据卷没有差别。

  接下来，可以查看该目录下的MySQL的data文件：

  ```Bash
  ls -l /var/lib/docker/volumes/29524ff09715d3688eae3f99803a2796558dbd00ca584a25a4bbc193ca82459f/_data
  ```

  注意：每一个不同的镜像，将来创建容器后内部有哪些目录可以挂载，可以参考DockerHub对应的页面