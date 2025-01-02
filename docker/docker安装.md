# docker安装

![img](D:\桌面文件\ALL\java pictures\docker\1.jpg)

## Ubuntu 22.04安装

### 准备条件

```yml
# 安装前先卸载操作系统默认安装的docker
sudo apt-get remove docker docker-engine docker.io containerd runc 

# 安装必要支持
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release
```

> 1. **`apt-transport-https`**
>    - 允许 `apt` 使用 HTTPS 协议下载软件包，这是 Docker 仓库所需要的安全协议。
> 2. **`ca-certificates`**
>    - 提供可信任的 SSL/TLS 证书，用于验证 HTTPS 连接。
> 3. **`curl`**
>    - 用于从 Docker 官方仓库下载 GPG 公钥和其他文件。
> 4. **`software-properties-common`**
>    - 提供 `add-apt-repository` 命令，方便管理和添加 PPA（个人包存档）或其他软件源。
> 5. **`gnupg`**
>    - 提供 GPG 加密工具，用于确保下载的 Docker 软件包的安全性。
> 6. **`lsb-release`**
>    - 显示发行版相关信息，有助于动态配置软件源，避免版本不匹配。

### 准备安装

```yml
#添加 Docker 官方 GPG key （可能国内现在访问会存在问题）
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 阿里源（推荐使用阿里的gpg KEY）
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg



#添加 apt 源:
#Docker官方源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


#阿里apt源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


#更新源 更新之后才能进行安装 否则可能出现依赖错误
sudo apt update
sudo apt-get update

```

### 安装docker

```yml
#安装最新版本的docker
sudo apt install docker-ce docker-ce-cli containerd.io
#等待安装完成

#查看docker版本
sudo docker version 

#查看docker 运行状态
sudo systemctl status docker
```

### 安装docker命令补全工具

```yml
sudo apt-get install bash-completion

sudo curl -L https://raw.githubusercontent.com/docker/docker-ce/master/components/cli/contrib/completion/bash/docker -o /etc/bash_completion.d/docker.sh

source /etc/bash_completion.d/docker.sh
```

### 允许非root用户执行docker命令

>当我们安装好了Docker之后，有两种方式来执行docker 命令
>
>在docker命令前加上sudo, 比如：sudo docker ps
>sudo -i 切换至root用户，再执行docker 命令
>是不是可以让当前用户在不切root，或者不加sudo 的情况下正常使用 docker 命令呢？答案是有的。

* 添加docker 用户组

  `sudo groupadd docker`

* 将当前用户添加到用户组

  `sudo usermod -aG docker $USER`

* 使权限生效

  `newgrp docker`

* 测试一下

  `docker ps -a`

* 最后一步, 更新.bashrc文件

  ```yml
  #如果没有此行命令，你会发现，当你每次打开新的终端
  #你都必须先执行一次 “newgrp docker” 命令
  #否则当前用户还是不可以执行docker命令
  groupadd -f docker
  ```

### 镜像加速 (截止到 2024.12.20有效)

> 找到你的daemon.json 文件，通常在 /etc/docker/daemon.json 这个位置 如果没有手动创建即可

* 在daemon.json中添加 

  ```json
  {
      "registry-mirrors": [
      "https://docker.m.daocloud.io",
      "https://hub.geekery.cn",	
      "https://hub.littlediary.cn",	
      "https://docker.rainbond.cc",	
      "https://docker.unsee.tech",
      "https://docker.m.daocloud.io",	
      "https://hub.crdz.gq",
      "https://docker.nastool.de",	
      "https://hub.firefly.store",	
      "https://registry.dockermirror.com",	
      "https://docker.1panelproxy.com",	
      "https://hub.rat.dev",
      "https://docker.udayun.com",	
      "https://docker.kejilion.pro",	
      "https://dhub.kubesre.xyz",	
      "https://docker.1panel.live"	
    ]
  }
  ```

  

  