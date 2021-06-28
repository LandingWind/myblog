# ownCloud+mysql+docker搭建个人云盘


记录三小时搭建ownCloud云盘全过程🆒

话不多说, 先放一张成功的登录界面图👌

{{<image src="https://i.loli.net/2020/12/26/RhlsSXaLQp2jqMt.png" title="ownCloud登录界面">}}


## step 1: 选择一个适合的开源云盘

这里有很多选择 比如老牌劲旅ownCloud, 分裂继任者nextCloud或者是国产招牌seaFile, 都在开源云盘市场上占有一席之地

这里我主要参考了v2ex还有知乎网友的一些评论, 结合我自身的设备情况和维护代价选择了**ownCloud** 两个字——省心!

## step 2: 选择安装方式

首先去看一下ownCloud的官方文档, 主要是提供了以下几种安装方式:

 - Installation with docker
 - Manual Installation

Manual Installation的话其实就是完整走一遍流程: Install ownCloud、Configure WebServer、Set Relationship...十分麻烦

我个人一直推崇使用docker进行项目的管理, 一方面docker中的配置不会影响到宿主机的环境配置, 另一方面docker中运行便于维护管理, 而且因为docker相当于梳理了一遍流程所以可以重复使用(因此就可以直接拿别人的**docker配置单**用啦🆒)

这里需要强调的一点是, ownCloud官方提供了docker包含了server和apache+php, 所以我就不舍近求远了直接用官方的docker image

{{<image src="https://i.loli.net/2020/12/26/NBscyDa5IuqEMPw.png" title="docker search owncloud">}}

(如果有其他需要可以去docker hub搜索其他docker image)

ownCloud docker允许使用内部自带的**sqlite**, 但是毕竟针对小文件还马马虎虎, 大文件就完犊子 所以我们还是需要一个**mysql/mariadb**(当然用**postgresql**也可以), 既然需要两个docker而且服务于同一目标——我们的个人云盘, 所以我直接采用**docker-compose**的方式

```yaml
version: '2'
services:
    owncloud:
    image: owncloud
    links: 
        - mysql:mysql
    volumes:
        - "/data/db/owncloud:/var/www/html/data"
    ports:
        - <webport>:80
    mysql:
    image: mysql
    volumes:
        - "/data/db/mysql:/var/lib/mysql"
    ports:
        - <port>:3306
    environment:
        MYSQL_ROOT_PASSWORD: "<password>"
        MYSQL_DATABASE: ownCloud
```
## step 3: 配置Nginx

我的阿里云机器所有项目都跑在各自的docker上, 然后对外端口有宿主机的Nginx同一映射管理, 所以我在/etc/nginx/conf.d目录下新建了一个针对ownCloud的代理转发conf:

```nginx
server {
    listen 80;
    server_name cloud.joyinn.top;

    proxy_set_header X-Forwarded-For $remote_addr;

    location / {
        proxy_set_header    Host $host;
        proxy_set_header    X-Real-IP $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto $scheme;
        add_header Cache-Control  "no-cache";

        proxy_pass http://127.0.0.1:8001;

        limit_rate 256m;
        client_max_body_size 0;
    }
}
```

这里需要注意的是两点, **一个是二级域名记得去域名解析那里配置好**, **另一个是注意要取消Nginx的带宽大小限制,** 否则ownCloud稍微大一点的文件就没有办法上传成功了

## step 4: 配置ownCloud

进入配置好的webServer网址, 第一次访问会需要填写ownCloud的基本配置, 包括管理员账户名和密码, 以及数据库的相关配置 这里只需要对照在docker-compose.yml文件里配置的信息填写即可, **localhost**这一项应该填写**mysql**

## step 5: php版本过低 与 mysql8 的兼容性问题

这里我遇到了一个大坑, 花了很多时间才解决:

> "Error while trying to create admin user: Failed to connect to the database: An exception occured in driver: SQLSTATE[HY000] [2054] The server requested authentication method unknown to the client "

搜索了谷歌, 看了许多帖子有了点头绪 

遇到的问题就是ownCloud docker中的web目前使用的php版本比较低, 还没有兼容mysql8.0以上的全部特性, 比如mysql8.0以上使用的用户密码机制不同于原来, 因此导致这里访问的错误

摸索了以下, 得到以下的解决方案:

1. 进入到mysql的container中, 增加 **my.cnf** 中一句配置: **default_authentication_plugin=mysql_native_password**
2. 进入mysql, 执行以下命令: **ALTER USER 'root'@'%' IDENTIFIED  WITH mysql_native_password BY 'password';**
3. 重启mysql的container

## step 6: 愉快的玩耍ownCloud吧

{{<image src="https://i.loli.net/2020/12/26/U9w8TzvuZCAVjnp.png" title="ownCloud存储页面">}}
