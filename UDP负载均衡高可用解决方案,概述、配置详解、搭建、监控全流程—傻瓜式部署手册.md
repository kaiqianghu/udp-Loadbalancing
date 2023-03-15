@[TOC](UDP负载均衡高可用解决方案,概述、配置详解、搭建、监控全流程)
# 一、概述
## 1.1 nginx简介
> Nginx是一款轻量级的Web 服务器/反向代理服务器及电子邮件（IMAP/POP3）代理服务器，在BSD-like 协议下发行。其特点是占有内存少，并发能力强
## 1.2 为什么使用Nginx
> 在传统的Web项目中，并发量小，用户使用的少。
> 所以在低并发的情况下，用户可以直接访问tomcat服务器，然后tomcat服务器返回消息给用户。
> 
> 因单个tomcat默认并发量有限制。如果请求量过大，会产生如下问题👇：
● Tomcat8 默认配置的最大请求数是 150，也就是说同时支持 150 个并发，也可以将其改大。
● 当某个应用拥有 250 个以上并发的时候，应考虑应用服务器的集群。
● 具体能承载多少并发，需要看硬件的配置，CPU 越多性能越高，分配给 JVM 的内存越多性能也就越高，但也会加重 GC 的负担。
● 操作系统对于进程中的线程数有一定的限制：
&emsp;&emsp;&emsp;○ Windows 每个进程中的线程数不允许超过 2000
&emsp;&emsp;&emsp;○ Linux 每个进程中的线程数不允许超过 1000 (在 Java 中每开启一个线程需要耗用 1MB 的 JVM 内存空间用于作为线程栈之用。)
&emsp;&emsp;&emsp;○ Tomcat的最大并发数是可以配置的，实际运用中，最大并发数与硬件性能和CPU数量都有很大关系的。
```shell
# 更好的硬件，更多的处理器都会使Tomcat支持更多的并发。
maxThreads="150" # 最大并发数 
minSpareThreads="10"		# 初始化时创建的线程数
maxSpareThreads="500"		# 一旦创建的线程超过这个值，Tomcat就会关闭不再需要的socket线程。
```
## 1.3 高并发
> 通常是指，通过设计保证系统能够同时并行处理很多请求。
高并发相关常用的一些指标有响应时间（Response Time），吞吐量（Throughput），每秒查询率QPS（Query Per Second），并发用户数等。
● 响应时间：系统对请求做出响应的时间。例如系统处理一个HTTP请求需要200ms，这个200ms就是系统的响应时间。
● 吞吐量：单位时间内处理的请求数量。
● QPS：每秒响应请求数。在互联网领域，这个指标和吞吐量区分的没有这么明显。
● 并发用户数：同时承载正常使用系统功能的用户数量。
## 1.4 正向代理反向代理
### 1.4.1  正向代理
>&emsp;&emsp;是一个位于客户端和原始服务器(origin server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端。客户端才能使用正向代理。

### 1.4.2  反向代理
>&emsp;&emsp;是一个位于客户端和原始服务器(origin server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端。客户端才能使用正向代理。

### 1.4.3  正向代理和反向代理的区别
> **正向代理**：
&emsp;&emsp;是在客户端的。比如需要访问某些国外网站，我们可能需要购买vpn。并且vpn是在我们的用户浏览器端设置的(并不是在远端的服务器设置)。浏览器先访问vpn地址，vpn地址转发请求，并最后将请求结果原路返回来。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ceb70eabb3b04e15b8a06f7276d55b83.png)
<br>
**反向代理**：
&emsp;&emsp;反向代理是作用在服务器端的，是一个虚拟ip(VIP)。对于用户的一个请求，会转发到多个后端处理器中的一台来处理该具体请求。
![在这里插入图片描述](https://img-blog.csdnimg.cn/fc5406454761497b9c47dbc4a2464777.png)

## 1.5 HTTP常见状态码列表
![在这里插入图片描述](https://img-blog.csdnimg.cn/38e0fe13508f4243aba4745f1194ab3a.png)
## 1.6 软件包下载地址
```shell
nginx: 
	http://nginx.org/en/download.html
keepalived：
	https://www.keepalived.org/download.html
```

# 二、nginx详解
## 2.1 nginx配置详解
```conf
├── conf                                             # 这是nginx所有配置文件的目录
│   ├── fastcgi.conf                                     # fastcgi 相关参数的配置文件
│   ├── fastcgi.conf.default                       #   fastcgi.conf 的原始备份
│   ├── fastcgi_params                             #   fastcgi的参数文件
│   ├── fastcgi_params.default
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types                                  #  媒体类型
│   ├── mime.types.default
│   ├── nginx.conf                                  #  nginx默认的主配置文件
│   ├── nginx.conf.default
│   ├── scgi_params                               # scgi 相关参数文件
│   ├── scgi_params.default
│   ├── uwsgi_params                          #  uwsgi相关参数文件
│   ├── uwsgi_params.default
│   └── win-utf
​
├── fastcgi_temp                               #  fastcgi临时数据目录
​
├── html                                            #  这是编译安装时nginx的默认站点目录，类似Apache的默认站点htdocs  
│   ├── 50x.html                                 # 错误页面优雅替代显示文件，例如：出现502错误时会调用此页面error_page 500 502 503 504 /50x.html
│   └── index.html                               # 默认的首页文件，index.html\index.asp\index.jsp来做网站的首页文件
​
├── logs                                              # nginx默认的日志路径，包括错误日志及访问日志
│   ├── access.log                              #  nginx的默认访问日志文件，使用tail -f access.log,可以实时观看网站的用户访问情况信息
│   ├── error.log                              #   nginx的错误日志文件，如果nginx出现启动故障可以查看此文件
│   └── nginx.pid                             #   nginx的pid文件，nginx进程启动后，会把所有进程的ID号写到此文件
​
├── nginx-1.6.3 -> /application/nginx-1.6.3
├── proxy_temp                                # 临时目录
├── sbin                                            # 这是nginx命令的目录，如nginx的启动命令nginx
│   ├── nginx                                    # Nginx的启动命令nginx
│   └── nginx.old
├── scgi_temp                                 # 临时目录
└── uwsgi_temp                             # 临时目录
```
### 2.1.1 nginx默认路径
```shell
#  Nginx配置路径(生产环境部署在了/usr/local/nginx/下 )
/etc/nginx/

#  PID目录
/var/run/[nginx.pid](https://www.centos.bz/tag/nginx-pid/)

#  错误日志
/var/log/nginx/[error](https://www.centos.bz/tag/error/).log

#  访问日志
/var/log/nginx/access.log

#  默认站点目录
/usr/share/nginx/html
```

### 2.1.2 nginx.conf详解
```conf
user  probe;	# 运行用户
worker_processes  4;   # 启动进程,通常设置成和cpu的数量相等

# 全局错误日志及PID文件
error_log  /var/log/nginx/error.log;
pid        /var/run/nginx.pid;

# 工作模式及连接数上限
events {
    use   epoll;   # epoll是多路复用IO(I/O Multiplexing)中的一种方式,但是仅用于linux2.6以上内核,可以大大提高nginx的性能
    worker_connections  1024;   #  单个后台worker process进程的最大并发链接数
		keepalive_timeout 60;  # keepalived超时时间,单位是秒
}
# 设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
    access_log    /var/log/nginx/access.log; # 设定日志格式
    # sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，必须设为 on；如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime.
    sendfile  on;

	# 文件扩展名与文件类型映射表
    include mime.types;

    # 默认文件类型
    default_type application/octet-stream;

    # 默认编码
    charset utf-8;

    stream {
        upstream udp-cluster {
            fair;
            server 10.126.8.17:514 type=udp;
            server 10.126.8.41:514 type=udp;
        }
        server {
            listen 514 udp;
            proxy_connect_timeout 5s;  # 与被代理服务器建立连接的超时时间为5s
            proxy_timeout 10s;	# 获取被代理服务器的响应最大超时时间为10s
            proxy_next_upstream on;	# 当被代理的服务器返回错误或超时时，将未返回响应的客户端连接请求传递给upstream中的下一个服务器
            proxy_next_upstream_tries 3;   # 转发尝试请求最多3次
            proxy_next_upstream_timeout 10s;    # 总尝试超时时间为10s
            proxy_pass udp-cluster;   # 调用集群服务器
            proxy_socket_keepalive on;  # 开启SO_KEEPALIVE选项进行心跳检测
            proxy_bind $remote_addr transparent;
        }
    }
}
```
### 2.1.3 upstream分配方式
> nginx的upstream支持5种分配方式，如下👇（其中，前3种为nginx原生支持的分配方式，后两种为第三方支持的分配方式）：
#### 2.1.3.1  轮询
> 轮询是upstream的默认分配方式，即每个请求按照实际顺序轮流分配到不同的后端服务器，如果某个后端服务器down掉后，能自动剔除。
```shell
upstream backend {
    server 192.168.1.101:514 type=udp;
    server 192.168.1.102:514 type=udp;
    server 192.168.1.103:514 type=udp;
}
```
#### 2.1.3.2 weight权重
> 轮询的加强版，即可以指定轮询比率，weight和访问几率成正比，主要应用于后端服务器异质的场景下。
```shell
upstream backend {
    server 192.168.1.101 weight=1;
    server 192.168.1.102 weight=2;
    server 192.168.1.103 weight=3;
}
```
#### 2.1.3.3 ip_hash
> 每个请求按照访问ip（即Nginx的前置服务器或者客户端IP）的hash结果分配，这样每个访客会固定访问一个后端服务器，可以解决session一致问题。
```shell
upstream backend {
    ip_hash;
    server 192.168.1.101:7777;
    server 192.168.1.102:8888;
    server 192.168.1.103:9999;
}
```
#### 2.1.3.4 fair
> fair顾名思义，公平地按照后端服务器的响应时间（rt）来分配请求，响应时间短即rt小的后端服务器优先分配请求。 
```shell
upstream backend {
    server 192.168.1.101;
    server 192.168.1.102;
    server 192.168.1.103;
    fair;
}
```
#### 2.1.3.5 url_hash
> 与ip_hash类似，但是按照访问url的hash结果来分配请求，使得每个url定向到同一个后端服务器，主要应用于后端服务器为缓存时的场景下。
```shell
upstream backend {
    server 192.168.1.101;
    server 192.168.1.102;
    server 192.168.1.103;
    hash $request_uri;
    hash_method crc32;
}
```
**其中，hash_method为使用的hash算法，需要注意的是：此时，server语句中不能加weight等参数。**
### 2.1.4  server参数详解
```shell
# 与被代理服务器建立连接的超时时间为5s
proxy_connect_timeout 5s;

# 获取被代理服务器的响应最大超时时间为10s
proxy_timeout 10s;

# 当被代理的服务器返回错误或超时时，将未返回响应的客户端连接请求传递给upstream中的下一个服务器
proxy_next_upstream on;

# 转发尝试请求最多3次
proxy_next_upstream_tries 3;

# 总尝试超时时间为10s
proxy_next_upstream_timeout 10s;

# 调用集群服务器
proxy_pass udp-cluster;

# 开启SO_KEEPALIVE选项进行心跳检测
proxy_socket_keepalive on;
```
## 2.2  基础命令如下👇
```shell
/usr/local/nginx/sbin/nginx  # 启动nginx

/usr/local/nginx/sbin/nginx -s stop # 快速关闭Nginx，可能不保存相关信息，并迅速终止web服务。

/usr/local/nginx/sbin/nginx -s quit # 平稳关闭Nginx，保存相关信息，有安排的结束web服务。

# 重启nginx(因改变了Nginx相关配置，需要重新加载配置而重载)
/usr/local/nginx/sbin/nginx -s reload 

/usr/local/nginx/sbin/nginx -s reopen # 重新打开日志文件

/usr/local/nginx/sbin/nginx -c filename # 为 Nginx 指定一个配置文件，来代替缺省的。 

# 不运行，而仅仅测试配置文件。nginx 将检查配置文件的语法的正确性，并尝试打开配置文件中所引用到的文件。 
/usr/local/nginx/sbin/nginx -t

/usr/local/nginx/sbin/nginx -v #  显示 nginx 的版本。

/usr/local/nginx/sbin/nginx -V #  显示 nginx 的版本，编译器版本和配置参数
```
# 三、开始搭建nginx
## 1.1 环境检查(nginx1、nginx2都要操作)
### 1.1.1 关闭selinux
 > 配置 nginx前需要检查selinux是否关闭
 
 ```shell
 # 查看 selinux状态
getenforce

# 关闭selinux
vi /etc/selinux/config
将SELINUX=enforcing改为SELINUX=disabled

#  临时关闭selinux
setenforce 0
 ```
 
### 1.1.2  创建相关用户
```shell
groupadd probe

useradd -g probe probe
```
## 1.2 修改linux内核参数(nginx1、nginx2都要操作)
```shell
# 查看内核限制
ulimit -a

#  修改内核限制（临时）
ulimit -n 65535

#  永久规则，修改限制，重启后生效，所以临时限制也要做
vim /etc/security/limits.conf
# 用户或组 软限制或硬限制 项目 值
* soft nofile 65535
* hard nofile 65535
```
### 1.3 安装所需依赖（nginx1、nginx2都要操作 ）
```shell
yum install -y gcc pcre pcre-devel openssl openssl-devel
```

### 1.4 安装 nginx（nginx1、nginx2都要操作 ）
```shell
#  1、解压tar包
tar -xf nginx-1.22.1.tar.gz -C /home/zzitcj/

# 2、创建程序相关目录
mkdir /usr/local/nginx


#  3、创建相关数据目录
mkdir -p /usr/local/nginx/log

mkdir -p /usr/local/nginx/run

# 4、编译
cd /home/zzitcj/nginx-1.22.1

# prefix指定安装目录，user、group指定用户和用户组，--with-http_ssl_module支持加密功能，--with-stream安装stream模块，--with-http_stub_status_module开启模块功能，用于后续nginx监控
./configure --prefix=/usr/local/nginx --user=probe --group=probe --pid-path=/usr/local/nginx/run/nginx.pid --with-http_ssl_module --with-stream --with-http_stub_status_module --with-http_gzip_static_module

make && make install
```

## 1.5 配置nginx.conf（nginx1、nginx2都要操作 ）
```shell
user  probe;
worker_processes  4;

error_log  /usr/local/nginx/log/error.log;
pid        /usr/local/nginx/run/nginx.pid;

events {
    use   epoll;
    worker_connections  1024;
}

stream {
    upstream udp-syslog-cluster-10514 {
        server 10.126.8.17:11514 ;
        server 10.126.8.41:11514 ;
    }
    server {
        listen 10514 udp;
        proxy_connect_timeout 5s;
        proxy_timeout 10s;
        proxy_next_upstream on;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
        proxy_pass udp-syslog-cluster-10514;
        proxy_socket_keepalive on;
    }
    upstream udp-syslog-cluster-10519 {
        server 10.126.8.17:10519 ;
        server 10.126.8.41:10519 ;
    }
    server {
        listen 10519 udp;
        proxy_connect_timeout 5s;
        proxy_timeout 10s;
        proxy_next_upstream on;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
        proxy_pass udp-syslog-cluster-10519;
        proxy_socket_keepalive on;
    }
}
http {
    access_log    /usr/local/nginx/log/access.log;
    charset utf-8;
}
```
## 1.6 配置systemctl管理nginx服务
```shell
vim /usr/lib/systemd/system/nginx_TrapSyslog.service

[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
User=probe
Group=probe
PIDFile=/usr/local/nginx/run/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
```
## 1.7 检查配置文件是否规范无报错
```shell
/usr/local/nginx/sbin/nginx -t
```
## 1.8 修改数据目录权限并启动服务
### 1.8.1 修改数据目录权限
```shell
chown -R probe.probe /usr/local/nginx
```
### 1.8.2 启动服务
```shell
# 加载 
systemctl daemon-reload

# 配置开机自启并启动
systemctl enable --now nginx_TrapSyslog.service

#  开启
systemctl status nginx_TrapSyslog.service
```
# 四、搭建keepalived
## 1.1 keepalived(2台同样的操作)
### 1.1.1 上传源码包至要部署程序的采集机，进行程序部署
```shell
# 上传源码包至要部署程序的采集机
scp -r keepalived-2.2.7.tar.gz root@10.126.8.17:~/
```


### 1.1.2 配置依赖环境与源码安装keepalived，安装版本2.2.7
```shell
# 安装依赖包
yum -y install gcc openssl-devel libnl libnl-devel libnfnetlink-devel

# 解压源码包
tar -xf /home/zzitcj/keepalived-2.2.7.tar.gz -C /home/zzitcj/

cd /home/zzitcj/keepalived-2.2.7

# 配置Makefile，指向安装目录/usr/local/keepalived
./configure --prefix=/usr/local/keepalived --with-init=systemd --with-systemdsystemunitdir=/usr/lib/systemd/system

# 编译并安装
make && make install

chown -R probe.probe /usr/local/keepalived
```


### 1.1.3 修改启动参数
```shell
sed -i 's@KEEPALIVED_OPTIONS=.*@KEEPALIVED_OPTIONS="-f /usr/local/keepalived/etc/keepalived/keepalived.conf -p /usr/local/keepalived/temp/keepalived.pid -D"@' /usr/local/keepalived/etc/sysconfig/keepalived
```

## 1.2 修改keepalived配置
### 1.2.1  master
#### 1.2.1.1 编写keepalived.conf
```shell
vim /usr/local/keepalived/etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
  router_id HOSTNAME
  vrrp_iptables
  vrrp_skip_check_adv_addr
  vrrp_strict
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
vrrp_script check_nginx_alived {
  script "/usr/local/keepalived/bin/nginx_check.sh"
  interval 20
  fall 3    # require 3 failures for KO
}
vrrp_instance VI_1 {
  state MASTER
  interface bond1
  virtual_router_id 51
  priority 100
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass Ng1NXCLUSTER
  }
  track_script {
    check_nginx_alived
  }
  virtual_ipaddress {
    vipaddress
  }
}
```

#### 1.2.1.2 编写nginx_check.sh
```shell
vim /usr/local/keepalived/bin/nginx_check.sh
#!/bin/bash
# Author: hu.xiao
# Date: 2023.03.14
# Last_editdate: 2023.03.14
# Descrition: check nginx service status
check_log="/usr/local/nginx/log/nginx_check-`date +%Y-%m-%d-%H%M`.log"
> ${check_log}

pid_file="/usr/local/nginx/run/nginx.pid"

nginx_system_status=`systemctl status nginx_TrapSyslog.service  | grep Active | grep running | wc -l`

if  [ [ ${nginx_system_status} == 1] && [ -f ${pid_file} ] ];then
	echo -e "`date +%Y-%m-%d-%H%M`:  nginx服务为健康状态." >> ${check_log}
	exit 0
else
	echo -e "`date +%Y-%m-%d-%H%M`:  nginx服务异常，连续3次异常Keepliaved将发生切换 ." >> ${check_log}
	exit 1
fi
```
### 1.2.2  slave
#### 1.2.2.1 编写keepalived.conf
```shell
vim /usr/local/keepalived/etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
  router_id HOSTNAME
  vrrp_iptables
  vrrp_skip_check_adv_addr
  vrrp_strict
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
vrrp_script check_nginx_alived {
  script "/usr/local/keepalived/bin/nginx_check.sh"
  interval 20
  fall 3    # require 3 failures for KO
}
vrrp_instance VI_1 {
  state BACKUP
  interface bond1
  virtual_router_id 51
  priority 50
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    auth_pass Ng1NXCLUSTER
  }
  track_script {
    check_nginx_alived
  }
  virtual_ipaddress {
    vipaddress
  }
}
```

#### 1.2.2.2 编写nginx_check.sh
```shell
vim /usr/local/keepalived/bin/nginx_check.sh

#!/bin/bash
# Author: hu.xiao
# Date: 2023.03.14
# Last_editdate: 2023.03.14
# Descrition: check nginx service status
check_log="/usr/local/nginx/log/nginx_check-`date +%Y-%m-%d-%H%M`.log"
> ${check_log}

pid_file="/usr/local/nginx/run/nginx.pid"

nginx_system_status=`systemctl status nginx_TrapSyslog.service  | grep Active | grep running | wc -l`

if  [ [ ${nginx_system_status} == 1] && [ -f ${pid_file} ] ];then
	echo -e "`date +%Y-%m-%d-%H%M`:  nginx服务为健康状态." >> ${check_log}
	exit 0
else
	echo -e "`date +%Y-%m-%d-%H%M`:  nginx服务异常，连续3次异常Keepliaved将发生切换 ." >> ${check_log}
	exit 1
fi
```


### 1.2.3 修改2台主机的 keepalived配置文件 route_id与vip
```shell
# 1、修改hostname
sed -i 's/HOSTNAME/'${HOSTNAME}'/g'  /usr/local/keepalived/etc/keepalived/keepalived.conf

# 2、修改vip
sed -i 's/vipaddress/10.126.8.17/g'  /usr/local/keepalived/etc/keepalived/keepalived.conf
```
## 1.3 创建依赖目录并创建软连接 👇
```shell
mkdir /usr/local/keepalived/temp/

mkdir /etc/keepalived

ln -s /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf

ln -s /usr/local/keepalived/sbin/keepalived /usr/bin/keepalived

keepalived -t
#附加：
#+ keepalived -t 可以检查配置文件的语法，默认检查的是/etc/keepalived/keepalived.conf
#+ 所以，我这里配置了一个软连接。
```


## 1.4 修改启动文件 keepalived.service
```shell
vim /usr/lib/systemd/system/keepalived.service

[Unit]
Description=LVS and VRRP High Availability Monitor
After= network-online.target syslog.target
Wants=network-online.target

[Service]
Type=forking
User=probe
Group=probe
PIDFile=/usr/local/keepalived/temp/keepalived.pid
KillMode=control-group
EnvironmentFile=-/usr/local/keepalived/etc/sysconfig/keepalived
ExecStart=/usr/local/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```


## 1.5 firewalld开启组播通信（注意按照网卡开通，此处是bond1）
```shell
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface bond1 --protocol vrrp -j ACCEPT

firewall-cmd --permanent --zone=public --add-port=16162/udp
firewall-cmd --permanent --zone=public --add-port=18162/udp

firewall-cmd --reload

firewall-cmd --list-all
```


## 1.6启动服务
### 1.6.1 添加脚本执行权限
```shell
chmod 755 /usr/local/keepalived/bin/*.sh
```
### 1.6.2 设置开机自启动并启动服务
```shell
chown -R probe.probe /usr/local/keepalived

systemctl daemon-reload

# 开机自启动
systemctl enable --now  keepalived

# 启动keepalived服务（由于咱们配置的是不抢占模式，注意需先停止所有节点的keepalived，然后优先启动master节点的keepalived服务）
systemctl restart keepalived

systemctl stop keepalived

systemctl start keepalived

systemctl status keepalived
```
# 五、添加告警自监控
> 此处文档只介绍了本次高可用改造使用程序的 监控步骤，实例的监控方式按照之前端口探测的方式进行监控
## 1.1 编写监控脚本
### 1.1.1  nginx监控脚本
```shell
vim /home/zabbix_scripts/nginx_system_status-check.sh

#!/bin/bash
# Author: hu.xiao
# Date: 2023.03.14
# Last_editdate: 2023.03.14
# Descrition: check nginx service status
status=$1

pid_file="/usr/local/nginx/run/nginx.pid"
nginx_system_status=$(systemctl status nginx_TrapSyslog |awk '{if (NR==3){print$2}}')
if [[ ${nginx_system_status} == $status ]  && [ -f ${pid_file} ]];then
    echo "1"
else
    echo "0"
fi
```


### 1.1.2  keepalived监控脚本
```shell
vim /home/zabbix_scripts/keepalived_system_status-check.sh

#!/bin/bash
# Author: hu.xiao
# Date: 2023.03.14
# Last_editdate: 2023.03.14
# Descrition: check keepalived  service status
status=$1

pid_file="/usr/local/keepalived/temp/keepalived.pid"
keepalived_system_status=$(systemctl status keepalived |awk '{if (NR==3){print$2}}')
if [[ ${keepalived_system_status} == $status ]  && [ -f ${pid_file} ]];then
    echo "1"
else
    echo "0"
fi
```
### 1.1.3 vip存活探测监控
```shell
vim /home/zabbix_scripts/keepalived_vip-check.sh

#!/bin/bash
# Author: hu.xiao
# Date: 2023.03.14
# Last_editdate: 2023.03.14
# Descrition: check VIP address status
IP=$1

if [ `ip a s | grep $IP | wc -l ` -eq 1 ];then
    echo "1"
else
    echo "0"
fi
```


## 1.2 配置zabbix
### 1.2.1 开启自定义监控
```shell
vim /etc/zabbix/zabbix_agentd.conf

UnsafeUserParameters=1
Include=/etc/zabbix/etc/zabbix_agentd.conf.d/*.conf
```

### 1.2.2 配置自定义监控参数
> 可以写在一个conf文件中，也可以拆开写；个人习惯以及推荐拆开写，遇到问题好排查 

#### 1.2.2.1  keepalived_system_status-check.conf
```shell
UserParameter=keepalived_system_status-check[*],/home/zabbix_scripts/keepalived_system_status-check.sh $1
```
#### 1.2.2.2  nginx_system_status-check.conf
```shell
UserParameter=nginx_system_status-check[*],/home/zabbix_scripts/nginx_system_status-check.sh $1
```
#### 1.2.2.3  keepalived_vip-check.conf
```shell
UserParameter=keepalived_vip-check[*],/home/zabbix_scripts/keepalived_vip-check.sh $1
```

### 1.2.3 重启zabbix
```shell
# 查看pid
ss -ntulp | grep 10050

#  kill -9 pid,杀死进程，此步一定确认好端口号
kill -9 $pid

#启动服务
zabbix_agentd
```


## 1.3 配置systemctl管理zabbix（附加）
> 由于现zabbix管理特别繁琐，特加此步，以便后续运维<br>
**注意：**
	欧拉以语雀文档安装的请不要再执行此操作；
```shell
vim /usr/lib/systemd/system/zabbix_agentd.service

[Unit]
Description=Zabbix Agent
After=syslog.target network.target network-online.target
Wants=network.target network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf
RemainAfterExit=yes
PIDFile=/var/run/zabbix/zabbix_agentd.pid

[Install]
WantedBy=default.target
```
### 1.3.1  systemctl管理命令
```shell
systemctl daemon-reload

# 设置开机自启并启动zabbix_agentd
systemctl enable --now zabbix_agentd

# 启动zabbix_agentd
systemctl start zabbix_agentd

# 停止 zabbix_agentd
systemctl stop zabbix_agentd

# 查看服务状态
systemctl status zabbix_agentd
```
## 1.4 页面添加监控
### 1.4.1 定义监控模板
> 添加好监控项、触发器规则、图形
### 1.4.2 关联业务主机
### 1.4.3 添加告警动作
> 短信告警并尝试自恢复业务
