#openresty-study
Contents
===========
* [Description](#description)
* [CentOS 安装openresty](#centos安装openresty)
* [安装火焰图工具](#安装火焰图工具)
* [postgres 使用遇到的错误](#postgres使用遇到的错误)
* [源码安装tmux遇到的一些问题](#源码安装tmux遇到的一些问题)
* [安装zsh](#安装zsh)
* [配置vim](#配置vim)
* [CentOS7 安装OpenVPN](#centos7安装openvpn)

Description
===========
记录一些问题和方法，供以后查阅，有错误之处欢迎交流。

[Back to TOC](#contents)

CentOS安装openresty
--------

(1)官网下载tar包：
[https://openresty.org/en/download.html](https://openresty.org/en/download.html)
选择合适的版本，如：
```
    wget https://openresty.org/download/openresty-1.11.2.1.tar.gz
```
(2)解压缩
```
    tar -xzf openresty-1.11.2.1.tar.gz
    cd  openresty-1.11.2.1
```
(3)按照官网提供的选项进行编译安装：
```
    ./configure --prefix=/opt/openresty\
    --with-luajit\
    --without-http_redis2_module \
    --with-http_iconv_module
```

一般会提示错误：
```
    ./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.
```
一次性安装所缺库：
```
yum install -y readline-devel pcre-devel openssl-devel perl
```

然后：
```
    gmake
    gmake install
```
即可<br>
（4）可以为openresty设置环境变量，我一般使用软链接的形式，你也可以把
```
    /opt/openresty/nginx/sbin/
```
添加到/etc/profile文件里面，在末尾加上：
```
 export PATH=$PATH:/opt/openresty/nginx/sbin 
```
然后执行source /etc/profile即可
使用软链接的形式：
```
ln -s /opt/openresty/nginx/sbin/nginx /usr/bin/nginx 
```

（5）测试openresty是否安装成功：
```
$ nginx
$ curl 127.0.0.1
```
使用默认的配置，这个时候访问的结果是：
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to OpenResty!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to OpenResty!</h1>
<p>If you see this page, the OpenResty web platform is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="https://openresty.org/">openresty.org</a>.<br/>

<p><em>Thank you for flying OpenResty.</em></p>
</body>
</html>
```
[Back to TOC](#contents)


安装火焰图工具
--------
1、安装SystemTap
首先需要安装内核开发包和调试包（这一步非常重要并且最为繁琐）：
```
# #Installaion:
# rpm -ivh kernel-debuginfo-($version).rpm
# rpm -ivh kernel-debuginfo-common-($version).rpm
# rpm -ivh kernel-devel-($version).rpm
```
其中$version使用linux命令 uname -r 查看，需要保证内核版本和上述开发包版本一致才能使用systemtap<br>

先看看自己系统的型号，可以查看：
如我的的机器:
```
[root@VM_52_89_centos nginx]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@VM_52_89_centos nginx]# 

则是CentOS7，查看内核版本：
[root@localhost ~]# uname -r                     
 ->3.10.0-327.22.2.el7.x86_64 
```
则$version= 3.10.0-327.22.2.el7.x86_64 

再去官网下载rpm包：
[http://debuginfo.centos.org/](http://debuginfo.centos.org/)
找到对应的下载链接，wget安装即可。（其中kernel-devel-($version).rpm一般都已经安装了）<br>

测试是否安装成功：
```
# yum install systemtap
# ...
# 测试systemtap安装成功否：
# stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'
Pass 1: parsed user script and 103 library script(s) using 201628virt/29508res/3144shr/26860data kb, in 10usr/190sys/219real ms.
Pass 2: analyzed script: 1 probe(s), 1 function(s), 3 embed(s), 0 global(s) using 296120virt/124876res/4120shr/121352data kb, in 660usr/1020sys/1889real ms.
Pass 3: translated to C into "/tmp/stapffFP7E/stap_82c0f95e47d351a956e1587c4dd4cee1_1459_src.c" using 296120virt/125204res/4448shr/121352data kb, in 10usr/50sys/56real ms.
Pass 4: compiled C into "stap_82c0f95e47d351a956e1587c4dd4cee1_1459.ko" in 620usr/620sys/1379real ms.
Pass 5: starting run.
read performed
Pass 5: run completed in 20usr/30sys/354real ms.
```
2、下载ngx工具包：[https://github.com/openresty/nginx-systemtap-toolkit](https://github.com/openresty/nginx-systemtap-toolkit)；直接clone下来即可;
这里工具有很多，使用：
```
# ps -ef | grep nginx （ps：得到类似这样的输出，其中15010即使worker进程的pid，后面需要用到）
hippo 14857 1 0 Jul01 ? 00:00:00 nginx: master process /opt/openresty/nginx/sbin/nginx -p /home/hippo/skylar_server_code/nginx/main_server/ -c conf/nginx.conf
hippo 15010 14857 0 Jul01 ? 00:00:12 nginx: worker process
# ./ngx-sample-lua-bt -p 15010 --luajit20 -t 5 > tmp.bt （-p 是要抓的进程的pid --luajit20|--luajit51 是LuaJIT的版本 -t是探测的时间，单位是秒， 探测结果输出到tmp.bt）
# ./fix-lua-bt tmp.bt > flame.bt (处理ngx-sample-lua-bt的输出，使其可读性更佳)
```

3、下载下载Flame-Graphic生成包：[https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)；
使用ngx工具包抓取想要的信息;
```
# stackcollapse-stap.pl flame.bt > flame.cbt
# flamegraph.pl flame.cbt > flame.svg;
```
生成火焰图.<br>

常见错误：
```
# sample at 1K Hz for 5 seconds, assuming the Nginx worker
#   or master process pid is 9768.
$ ./ngx-sample-lua-bt -p 9768 --luajit20 -t 5 > tmp.bt
WARNING: Tracing 9766 (/opt/nginx/sbin/nginx) for LuaJIT 2.0...
WARNING: Time's up. Quitting now...(it may take a while)

$ ./fix-lua-bt tmp.bt > a.bt  #使用这个修复

The resulting output file a.bt can then be used to generate a Flame Graph by using Brendan Gregg's FlameGraph tools:
stackcollapse-stap.pl a.bt > a.cbt
flamegraph.pl a.cbt > a.svg

```

[Back to TOC](#contents)

postgres使用遇到的错误
---------

```shell
使用psql -U postgres -d skylar -h 127.0.0.1 -p 5432连接数据库时，出现以下错误：

(1)
psql: 致命错误:  用户 "postgres" Ident 认证失败

解决方法：
    vim /var/lib/pgsql/data/pg_hba.conf（如果是9.5，则是/var/lib/pgsql/9.5/data
/pg_hba.conf）

打开修改：
#IPv4 lcoal connections:
host all all 127.0.0.1/32 ident

为
#IPv4 lcoal connections:
host all all 127.0.0.1/32 trust

```

```shell
(2)执行psql -U postgres skylar出现
psql: 致命错误:  对用户"postgres"的对等认证失败
解决方法：
vi /etc/PostgreSQL/9.4/main/pg_hba.conf

# "local" is for Unix domain socket connections only
local all all peer

将peer改成 md5
即：
local all all md5
重启PostgreSQL数据服务：（9.5版本）
systemctl restart postgresql-9.5.service

再登陆即可
```

(3)openresty 使用postgres一般采用的是upstream的形式，在nginx.conf加配置：
```nginx
location /postgres {
            internal;
            default_type text/html;
            set_by_lua $query_sql '
                if ngx.var.arg_sql then
                    return ngx.unescape_uri(ngx.var.arg_sql)
                end               
                local ngx_share     = ngx.shared.ngx_cache_sql
                return ngx_share:get(ngx.var.arg_id)
                ';
            postgres_pass   database;
            rds_json          on;
            rds_json_buffer_size 16k;
            postgres_query  $query_sql;
            postgres_connect_timeout 1s;
            postgres_result_timeout 2s;
        }
       upstream database {
        postgres_server  127.0.0.1:5360  dbname=skylar
         user=postgres password=postgres;        
        postgres_keepalive max=80 mode=single overflow=reject;
    }
```
简单介绍下:
```
internal 这个指令指定所在的 location 只允许使用于处理内部请求，否则返回 404 。
set_by_lua 这一段内嵌的 Lua 代码用于计算出 $query_sql 变量的值，即后续通过指令postgres_query 发送给 PostgreSQL 处理的 SQL 语句。这里使用了GET请求的 query 参数作为 SQL 语句输入。
postgres_pass 这个指令可以指定一组提供后台服务的 PostgreSQL 数据库的 upstream
块。
rds_json 这个指令是 ngx_rds_json 提供的，用于指定 ngx_rds_json 的 output 过滤器的
开关状态，其模块作用就是一个用于把 rds 格式数据转换成 json 格式的 output filter。这个指令在这里出现意思是让 ngx_rds_json 模块帮助 ngx_postgres 模块把模块输出数据转换成 json 格式的数据。
rds_json_buffer_size 这个指令指定 ngx_rds_json用于每个连接的数据转换的内存大小.
默认是 4/8k,适当加大此参数，有利于减少 CPU 消耗。
postgres_query 指定 SQL 查询语句，查询语句将会直接发送给 PostgreSQL 数据库。
postgres_connect_timeout 设置连接超时时间。
postgres_result_timeout 设置结果返回超时时间。
```

(这里自己遇到了一个坑，要注意postgres的默认端口，在/var/lib/pgsql/9.5/data/postgresql.conf里面修改)

(4)ndk.set_var.set_quote_pgsql_str(md5_sha1)
```
ndk.set_var.set_quote_pgsql_str(md5_sha1)的作用是用来转义成适合pg存储格式的字符串
一般用在拼接sql字符串时使用

```
例如：
```lua
local sql = [[SELECT size FROM file where md5_sha1 =]]..ndk.set_var.set_quote_pgsql_str(md5_sha1)
```
[Back to TOC](#contents)

源码安装tmux遇到的一些问题
--------
记录下：
（1）clone 源代码仓库：
```
$ git clone https://github.com/tmux/tmux.git
```
(2) 编译之前先安装libevent，去官网下载tar包：
[http://libevent.org](http://libevent.org)

选择需要下载的版本复制链接地址，使用wget下载到本地（图形化的也可以直接下载），如（选择合适的版本，一般选stable即可）：
```
    wget https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz
```

```
    tar -xzf  libevent-2.0.22-stable.tar.gz
    cd  libevent-2.0.22-stable/
    $ ./configure && make
    $ sudo make install
```

(3) 编译tmux：
```
    cd tmux/
    sh autogen.sh
   ./configure && make
```
安装编译过程可能会提示一些错误：<br>
1）aclocal command not found
原因：自动编译工具未安装，安装上即可：
```
centOS： yum install automake
```
2) configure: error: "curses or ncurses not found"
```
ubuntu：apt-get install libncurses5-dev
centos: yum install ncurses-devel
```

(4) 编译成功之后会在tmux下生成一个可执行程序：tmux
```
    ./tmux
```
执行的时候可能会出现找不到库的情况：
```shell
./tmux: error while loading shared libraries: libevent-2.0.so.5: cannot open shared object file: No such file or directory
```
把安装好的libevent库的路径使用软链接到对应的目录：
```
    64位：
        ln -s /usr/local/lib/libevent-2.0.so.5 /usr/lib64/libevent-2.0.so.5
    32位：
        ln -s /usr/local/lib/libevent-2.0.so.5 /usr/lib/libevent-2.0.so.5
```

（5）设置环境变量
    现在使用tmux必须在编译好的目录下执行才可以，我们设置个环境变量即可：


    ln -s /home/hcw/Package/tmux/tmux /usr/bin/

    /home/hcw/Package/tmux/ 为你编译好的路径，因为/usr/bin/已经添加到系统环境变量，所以不需要再设置，如此即可使用tmux


（6）下面是我常用的tmux配置：
```
    wget https://github.com/huchangwei/dotfiles/raw/master/.tmux.conf  -P ~
```

其中需要用到插件管理，需要先安装插件管理器：
[https://github.com/tmux-plugins/tpm](https://github.com/tmux-plugins/tpm)
```
    $ git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```
在.tmux.conf添加：（如果你是使用我的配置，下面可以省略）
```
#List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'

# Other examples:
# set -g @plugin 'github_username/plugin_name'
# set -g @plugin 'git@github.com/user/plugin'
# set -g @plugin 'git@bitbucket.com/user/plugin'
```
```
# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
Reload TMUX environment so TPM is sourced:

# type this in terminal
$ tmux source ~/.tmux.conf
```
[Back to TOC](#contents)

安装ZSH
--------

(1)先安装zsh包
```
    yum install zsh
```
安装成功之后其实就可以用，但是为了配置个性化，我们一般使用oh my zsh,这里面已经设置好常用的配置
（2）安装oh my zsh
```
 curl -L https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh | sh
```

安装完成会出现一个oh my zsh的图案，表示已经安装成功，这个时候再使用zsh你会发现很多常见的配置都基本有了。

（3）我的.zshrc配置
下面是我的常用的.zshrc配置，有需要的可以参考下：
[https://github.com/huchangwei/dotfiles/raw/master/.zshrc](https://github.com/huchangwei/dotfiles/raw/master/.zshrc)
或者你想直接使用：
```
    wget https://github.com/huchangwei/dotfiles/raw/master/.zshrc -P ~
```

然后source一下即可：
```
    source ~/.zshrc
```
[Back to TOC](#contents)

配置vim
--------

github上有个不错的配置，也提供了自动安装方式，
[https://github.com/ma6174/vim](https://github.com/ma6174/vim)

```
    wget -qO- https://raw.github.com/ma6174/vim/master/setup.sh | sh -x
```
等待安装完即可，至于vim的插件后续添加可以去了解一下vim的插件管理<br>
[Back to TOC](#contents)


CentOS7安装OpenVPN
--------
(1) 安装openvpn包(源里面已经有了)<br>
```
    yum install openvpn
```

（2）查找openvpn安装路径<br>
我们需要查找的是server.conf文件：
```
    rpm -ql openvpn | grep server.conf
```

查找结果：

```
/usr/share/doc/openvpn-2.3.10/sample/sample-config-files/roadwarrior-server.conf
/usr/share/doc/openvpn-2.3.10/sample/sample-config-files/server.conf
/usr/share/doc/openvpn-2.3.10/sample/sample-config-files/xinetd-server-config
```

把server.conf复制到etc：
```
    cp /usr/share/doc/openvpn-2.3.10/sample/sample-config-files/server.conf /etc/openvpn
    cd /etc/openvpn
    vim server.conf
```

(3) 修改配置文件
取消以下5个语句的注释：
```
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
user nobody
group nobody
```

(4) 使用easy-rsa生成证书和秘钥
上述配置ok，安装easy-rsa：
```
    yum install easy-rsa
    cp -p /usr/share/easy-rsa /etc/openvpn
```
配置easy-rsa
```
cd /usr/share/easy-rsa/2.0
vim vars
```
一些个人信息需要配置一下，按自己需求：
```
export KEY_COUNTRY="US"
export KEY_PROVINCE="NY"
export KEY_CITY="New York"
export KEY_ORG="Organization Name"
export KEY_EMAIL="administrator@example.com"
export KEY_CN=droplet.example.com
export KEY_NAME=server
export KEY_OU=server
export KEY_SIZE=2048
```

修改完之后source一下：
```
    source ./vars
```

(5) 生成密钥
生成ca：
```
在/etc/openvpn/easy-rsa/2.0目录中执行：
./clean-all
./build-ca
```
之后会有提示：
```
-----
Country Name (2 letter code) [US]:
...
```
一路回车即可，使用我们配置好的设置。<br>
然后会在/etc/openvpn/easy-rsa/2.0/keys目录生成ca。
接着生成服务器生成密钥：
```
    ./build-key-server server
```
过程同上，一路回车，有了服务器密钥，再生成Diffie Hellman key exchange文件，这里生成的长度由之前的KEY_SIZE决定：
```
    ./build-dh
```
执行完成会在keys目录产生dh2048.pem (如果你的KEY_SIZE = 1024，这里产生的文件是dh1024.pem)：
最后将生成的密钥复制到openvpn的目录：
```
    cd keys/
    cp  dh2048.pem ca.crt server.crt server.key /etc/openvpn
```

(6)生成客户端证书
在/etc/openvpn/easy-rsa/2.0目录中执行：
```
    ./build-key client
```

会在keys目录生成client证书。

（7）配置防火墙，启动openvpn server
7-1 防火墙设置<br>
在CentOS 7中，iptables防火墙已经被firewalld所取代，需要使用如下方法：
首先启动firewalld
```
systemctl start firewalld.service
```
查看哪些服务允许通过：
```
$ firewall-cmd --list-services     
dhcpv6-client ssh
```
我的允许通过的只有：dhcpv6-client ssh<br>
添加openvpn：
```
$ firewall-cmd --add-service openvpn
success
$ firewall-cmd --permanent --add-service openvpn
success
```
看看是否添加成功：
```
$ firewall-cmd --list-services      
dhcpv6-client openvpn ssh

```

最后添加masquerade：
```
$ firewall-cmd --add-masquerade
success
$ firewall-cmd --permanent --add-masquerade
success
检查masquerade是否添加成功：
$ firewall-cmd --query-masquerade
yes
```

7-2 允许IP转发 <br>
在sysctl中开启IP转发
```
vim /etc/sysctl.conf

# Controls IP packet forwarding
net.ipv4.ip_forward = 1

```

7-3 启动openvpn服务
```
启动OpenVPN服务器并添加自动启动项：

sysctl -p
systemctl start openvpn@server
systemctl enable openvpn@server

```
至此服务端搭建完成。<br>
[Back to TOC](#contents)



Author
---------
huchangwei(codjust)<br>
mail：hcwzqmail@gmail.com<br>
[Back to TOC](#contents)


