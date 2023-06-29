## qBittorrent部署记录

### 初衷

&ensp;&ensp;主要是因为电脑硬盘有限，然后下载机也一直闲置干别的，之前部署qBittorrent也出现了不能下载的问题，还是想实现下载机的下载功能，所以才有了这次的操作。

<!--more-->

### docker-compose配置文件内容

```yml
  qbittorrent:
    #PT下载软件
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8090
    volumes:
      - /home/dylan/config/qBittorrent:/config
      - /WDC-disk:/downloads
      - /home/dylan/logs/qBittorrent:/config/qBittorrent/logs
      - /home/dylan/cache/qBittorrent:/config/.cache/qBittorrent
    ports:
      - 8090:8090
      - 14521:14521
      - 14521:14521/udp
    restart: unless-stopped
    networks:
      docker-network:
        ipv4_address: 172.21.0.30     
```

---

```yml
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8090
```

以上定义：

- PUID
  
  > 指定容器内运行的进程的用户ID

- PGID
  
  > 指定容器内运行的进程的组ID

- TZ
  
  > 设置容器的时区

- WEBUI_PORT
  
  > 设置web管理界面的端口

---

```yml
    volumes:
      - /home/dylan/config/qBittorrent:/config
      - /WDC-disk:/downloads
      - /home/dylan/logs/qBittorrent:/config/qBittorrent/logs
      - /home/dylan/cache/qBittorrent:/config/.cache/qBittorrent
```

 以上定义：

- /home/dylan/config/qBittorrent:/config
  
  > 映射配置目录到宿主机

- /WDC-disk:/downloads
  
  > 映射下载目录到宿主机

- /home/dylan/logs/qBittorrent:/config/qBittorrent/logs
  
  > 映射日志目录到宿主机

- /home/dylan/cache/qBittorrent:/config/.cache/qBittorrent
  
  > 映射缓存目录到宿主机（不确定是否为缓存目录，仅从名字上去识别的）

---

```yml
    ports:
      - 8090:8090
      - 14521:14521
      - 14521:14521/udp
```

以上定义：

- 8090:8090
  
  > 将docker中qBittorrent服务开启的web管理界面端口映射到宿主机

- 14521:14521
  
  > 将docker中qBittorrent服务需要的传输TCP端口映射到宿主机

- 14521:14521/udp
  
  > 将docker中qBittorrent服务需要的传输UDP端口映射到宿主机

---

```yml
    networks:
      docker-network:
        ipv4_address: 172.21.0.30     
```

以上定义：

- networks:
  
  > 网络设置

- docker-network:
  
  > 想要加入的网络名称

- ipv4_address: 172.21.0.30
  
  > 设定加入网络中IP地址

---

其他设定参数为常规项，我就不进行赘述了，保持不动即可。

### 说明

- &ensp;&ensp;将配置目录映射出来，是为了如果出现升级、重新构建容器等操作，不至于前期的参数以及下载的相关信息都没有了。

- &ensp;&ensp;将日志文件映射出来，是为了更好查看日志信息，不用每次都进入docker容器中，或着使用命令行进行查看。

- &ensp;&ensp;将缓存目录映射出来，可能是强迫症作祟，不需要的话，可以不进行映射。

- 单独设置网络，是因为在每次docker重启后，容器不一定使用的就是上次的IP，会导致nginx反向代理出现问题，所以将IP固定下来，这样就不会出现跳IP的情况。

- &ensp;&ensp;**environment中的WEBUI_PORT，需要与ports中的web管理界面端口一致，至少是ports的web管理界面端口映射的后面的需要与WEBUI_PORT一致，否则无法访问，前面的是映射出来到宿主机的端口，访问的时候访问的是“宿主机IP:前面的端口号”。**

- &ensp;&ensp;**建议先建立好从docker容器映射出来的目录，因为docker如果映射出来的目录没有的话，它会自己建立，按照上面的设置，建立出来的目录以及所属组是root的，所以一般用户权限改变不了，会有一定麻烦。**

- &ensp;&ensp;如果想设定网络的话，需要先在docker-compose配置文件中定义一个网络
  
  ```yml
  #建立虚拟网络
  networks:
    docker-network:
      driver: bridge
      ipam:
        driver: default
        config:
          - subnet: 172.21.0.1/24 
  ```
  
  以上定义：
  
  - `networks:`
    
    > 这是一个关键字，表示接下来要定义一个或多个网络。
  
  - `docker-network:`
    
    > 这是网络的名称，可以自定义。在这个例子中，网络的名称为docker-network。
  
  - `driver: bridge`
    
    > 指定网络的驱动程序为bridge。bridge是Docker默认的网络驱动程序，它允许容器通过宿主机进行通信。
  
  - `ipam:`
    
    > 这是一个关键字，表示接下来要定义IP地址管理器（IP Address Management）。
  
  - `driver: default`
    
    > 指定IP地址管理器的驱动程序为default。default是Docker默认的IP地址管理器，它可以为容器分配IP地址。
  
  - `config:`
    
    > 这是一个关键字，表示接下来要定义网络配置。
  
  - ` subnet: 172.21.0.1/24`
    
    > 指定网络的子网地址为172.21.0.1/24。这意味着该网络的IP地址范围为172.21.0.1到172.21.0.254，子网掩码为255.255.255.0。
