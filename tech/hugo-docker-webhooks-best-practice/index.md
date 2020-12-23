# Hugo+Docker+Webhooks最佳实践




记录迁移个人博客到Hugo的过程

## 写在前面

在这之前，我的技术博客经历过wordpress、hexo、typecho

{{< admonition tip "为什么选择Hugo" >}}

- build速度快
- 符合个人写作习惯（typora locally）
- golang大法好

{{< /admonition >}}

开始的想法是单纯部署在Github Pages上，但是github响应速度太慢

于是决定采用github作为仓库，实际部署在自己的阿里云服务器上，用Docker+Webhooks的形式实现高效管理



## Docker For Static

### `docker`

docker是隔离项目和宿主机的最佳解决方案，采用docker可以避免宿主机环境和项目环境的互相影响，同样有助于项目的版本迭代

### `方案`

hugo build生成静态网页文件，存放在public文件夹中

通过git版本管理，线上仓库用github

阿里云服务器构建一个nginx的docker，将`usr/share/nginx/html`挂载到宿主机

### `bash code`

```bash
# 查找nginx docker源
docker search nginx
# 下载想要的nginx
docker install nginx:(tag)
# 可以通过docker images查看是否下载完成
docker images

# 运行docker
docker run -dt \
--restart=always \
--name hugoblog \
-p 8010:80 \
-v ~/hugoblog/html:/usr/share/nginx/html  \
-d nginx
```



## Webhooks

### `webhooks`

github被微软收购以后推出的免费功能

当用户注册好webhooks后，github会在repo发生变动后向用户注册的URL发送一个post请求

{{<image src="http://qiniustorage.joyinn.top/webhook-setting.png" title="webhook设置页面">}}

**send me everything** 的选项将会涉及几十种事件，可参见webhooks的文档，这里我只选择了push事件需要通知

**payload URL** 中需要填写可接受post请求的网络地址

**Content type** 是post请求的body格式，这里我选择了application/json

**Secret** 是自行设定的一个密钥，收到post请求后在服务器端可以通过密钥对packet进行验证



### `server handler`

接下来就是服务端对post请求响应处理的逻辑

我用的是Expressjs，是一个Node框架，主要业务逻辑如下（偷懒没有写检验）

```javascript
// hugoblog webhook
// 偷懒不写校验了
app.post('/hugoblog', (req, res) => {
    console.log(req.body);
    const eventName = req.get('X-GitHub-Event')
    const sign = req.get('X-Hub-Signature')
    const delivery = req.get('X-GitHub-Delivery')
    const refHead = req.body.ref
    console.log("Receive a event: ", eventName);
    console.log('sign:', sign)
    console.log('delivery:', delivery)
    console.log('refHead:', refHead, typeof(refHead),refHead === 'refs/heads/master')
    if(refHead === 'refs/heads/master') {
        const shell = 'app/bash/hugoblog.sh'
        setTimeout(() => {
            console.log('new thread execute shell', shell)
            exec(shell, (err,stdout,stderr) => {
                if (stdout) {
                console.log('stdout out >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>')
                console.log(stdout)
                console.log('stdout over >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>')
                }
                err && console.log('hasErr: ', err)
                stderr && console.log('stderr: ', stderr)
            })
        }, 500)
    }
    res.send("ok");
})
```

当接收到post请求后，判断是否为master分支的请求，然后执行准备好的bash脚本

bash脚本内容如下

```bash
WEB_PATH='/root/hugoblog/html'
 
echo "Start deployment"
cd $WEB_PATH
echo "pulling source code..."
git pull origin master
echo "restart docker..."
docker restart hugoblog
echo "Finished."
```

{{< admonition note "注意" >}}

记得要重启承载静态网页文件的nginx docker哦

{{< /admonition >}}



## 让Blog更新只需要一行命令

bash，让你的生活更美好:smile:

```bash
# filename: fire.sh
# path: hugo站点根目录下

echo "ready to fire..."

echo "hugo build"
hugo

echo "cd public dir"
cd public

echo "git add"
git add .

echo "git commit"
git commit -m "hugo modify - auto commit"

echo "git push"
git push origin master

echo "fire finished"
```



写好文章只需要 `./fire.sh` 即可



## Possible Error & Solution

#### req.body没有data

主要是expressjs 4.0以后的版本将bodyparser模块功能独立了出来，需要自行添加中间件

```javascript
// for json parser
app.use(bodyParser.json())
// for x-www-form-urlencoded parser
app.use(bodyParser.urlencoded({extended:true}));
// 根据需要使用这两个中间件即可
```



#### bash脚本执行无效

起初我将响应webhook的expressjs同样做成了一个docker

这就导致执行bash是在容器内执行的，对宿主机没有任何影响

解决方案包括执行ssh远程命令、将响应webhook的框架直接架设在宿主机中、docker之间建立关联等等






