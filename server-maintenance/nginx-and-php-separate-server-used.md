## nginx与php服务独立服务器使用

&ensp;&ensp;一直在使用Oracle的薅羊毛得来的服务器，但是薅过的都知道E2服务器的内存，就1G不到，小的可怜。装上几个docker服务几乎没有了，而且PHP的docker服务占用一般都在200M左右，影响了性能发挥的极限，要不可以多装几个服务使用。

&ensp;&ensp;幸好，在当时薅羊毛的时候，还薅出来一个ARM的服务器，所以就想把PHP服务放到ARM服务器，然后web服务器连接ARM服务器的PHP使用，这就分担了web服务器的内存压力，而且ARM服务器的内存，24G，PHP的占用，毛毛雨了。

### 映射nginx网站文件目录

&ensp;&ensp;因为PHP如果需要解析PHP文件，需要可以连接到文件目录，但是分离了服务器使用，PHP就无法连接到nginx的文件目录了，所以需要先把这个问题解决。

&ensp;&ensp;经过询问GPT，给了我一个大概的方向，就是使用NFS服务，将nginx服务器上面的目录映射到PHP服务器中，这样就解决了PHP需要访问解析文件目录的问题。

#### 在nginx服务器上安装NFS

```shell
sudo apt-get install nfs-kernel-server
```

**因为服务器使用的是ubuntu，所以可能其他linux系统需要按照对应的安装方式进行安装。**

#### 设置共享目录

#### 新建一个想要共享的目录

使用命令``sudo mkdir www``，新建一个www的文件夹

#### 修改NFS配置

```shell
sudo vim /etc/exports
```

**在上面的这个文件中，添加一行以下语句**

``/home/ubuntu/www 运行访问的IP(rw,sync,no_subtree_check)``

> /shared_directory是要共享的目录的路径，*表示允许所有主机访问该目录，rw表示该目录可读写，sync表示同步写入，no_subtree_check表示不检查子目录。

示例

``/home/ubuntu/www 10.0.0.138(rw,sync,no_subtree_check)``

### 固定NFS服务端口（重要）

#### 查看当前NFS使用的端口

使用`rpcinfo -p`

![rpcinfo -p](http://img.liaofei.org/2023/06/21/1687315766.jpg)

#### 编辑 /etc/services

使用命令``sudo vim  /etc/services``在最下方添加固定端口的设置

![/etc/services](http://img.liaofei.org/2023/06/21/1687315767.jpg)

> nfs 和 portmapper两个服务是固定端口的，nfs为2049，portmapper为111。
> 
> 其他的3个服务我是用的是以下端口
> 
> mountd 55467/tcp
> mountd 39369/udp
> rquotad 966/tcp
> nlockmgr 39459/tcp
> nlockmgr 56023/udp

### 重新启动NFS服务

```shell
sudo systemctl restart nfs-kernel-server
```

### 开放对应端口

```shell
sudo firewall-cmd --zone=public --add-port=2049/tcp --permanent && /
sudo firewall-cmd --zone=public --add-port=2049/udp --permanent && /
sudo firewall-cmd --zone=public --add-port=111/tcp --permanent && /
sudo firewall-cmd --zone=public --add-port=111/udp --permanent && /
sudo firewall-cmd --zone=public --add-port=55467/tcp --permanent && /
sudo firewall-cmd --zone=public --add-port=39369/udp --permanent && /
sudo firewall-cmd --zone=public --add-port=966/tcp --permanent && /
sudo firewall-cmd --zone=public --add-port=39459/tcp --permanent && /
sudo firewall-cmd --zone=public --add-port=56023/udp --permanent && /
sudo firewall-cmd --reload && /
sudo firewall-cmd --list-all
```

> "/"用途是可以一次性输入多行命令

### 用户端安装服务

```shell
sudo apt-get install nfs-common
```

#### 创建一个本地目录，用于挂载共享目录

```shell
sudo mkdir /www
```

#### 使用mount命令将共享目录挂载到本地目录

```shell
sudo mount 服务器IP:/shared_directory /local_directory
```

> server是NFS服务器的IP地址或主机名
> 
> /shared_directory是要共享的目录的路径
> 
> /local_directory是本地目录的路径。

示例：

```shell
sudo mount 10.0.0.25:/home/ubuntu/www /home/ubuntu/www
```

#### 卸载共享的挂载（备用）

```shell
sudo umount /local_directory
```

> /local_directory是本地被挂载目录的路径

### 调整nginx解析配置

#### 修改网站配置文件的PHP服务器IP地址

```json
    location ~ \.php$ {
        fastcgi_pass   服务器IP:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
```

#### 重载nginx配置文件

使用命令进入docker

```shell
docker exec -it nginx /bin/bash
```

重载配置文件

```shell
nginx -s reload
```

#### 启动PHP服务

docker-compose配置文件中添加以下语句

```json
###########################################################     
  php:
    #PHP支持服务
    image: php:fpm
    volumes:
      - ~/config/php:/usr/local/etc/php # 宿主机php配置文件目录
      - ~/www:/var/www/html # 宿主机网站根目录
      - ~/logs/php:/var/log/php-fpm # 宿主机php日志目录
    container_name: php
    restart: always    
    ports:
      - "9000:9000" # 映射9000端口到宿主机   
    networks:
      docker-network:
        ipv4_address: 172.21.0.9   
###########################################################
```

> networks部分可以不使用，非必要项。

#### php相关配置初始化（重要）

首次启动前需要将volumes部分在每行前加#进行注释，否则直接映射会出现没有配置文件的情况。

```json
#    volumes:
#      - ~/config/php:/usr/local/etc/php # 宿主机php配置文件目录
#      - ~/www:/var/www/html # 宿主机网站根目录
#      - ~/logs/php:/var/log/php-fpm # 宿主机
```

启动php服务

```shell
docker-compose up -d php
```

然后使用以下语句将配置文件拷贝出来，

```shell
docker cp php:/usr/local/etc/php ~/config/php/
```

然后将docker-compose配置中的#去掉，

```json
    volumes:
      - ~/config/php:/usr/local/etc/php # 宿主机php配置文件目录
      - ~/www:/var/www/html # 宿主机网站根目录
      - ~/logs/php:/var/log/php-fpm # 宿主机
```

停止PHP服务

```shell
docker-compose stop php
```

删除PHP容器

```shell
docker-compose rm php
```

> 需要确认，点击Y，回车即可。

重新启动php服务

```shell
docker-compose up -d php
```

#### 开放PHP端口

```shell
sudo firewall-cmd --zone=public --add-port=9000/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

---

**到此基本完成了迁移，大家视自己的情况，上面的步骤酌情进行调整。**
