# 基于JWT的用户登录态管理


jwt管理用户登录态 —— 以`expressjs`和`vuejs`的前后端分离论坛项目作为实例

## session

很久很久以前，Web基本上就是文档的浏览而已

既然是浏览，作为服务器，不需要记录谁在某一段时间里都浏览了什么文档，每次请求都是一个新的HTTP协议，就是请求加响应

尤其是我不用记住是谁刚刚发了HTTP请求，每个请求对我来说都是全新的。这段时间很嗨皮

但是随着交互式Web应用的兴起，像在线购物网站，需要登录的网站等等，马上就面临一个问题，那就是要管理**会话**，必须记住哪些人登录系统，哪些人往自己的购物车中放商品，也就是说我必须把每个人区分开，这就是一个不小的挑战，

因为HTTP请求是无状态的，所以想出的办法就是给大家发一个会话标识(session id)，说白了就是一个随机的字串，每个人收到的都不一样，每次大家向我发起HTTP请求的时候，把这个字符串给一并捎过来，这样我就能区分开谁是谁了

{{< admonition digest "流程" false >}}
1. 用户向服务器发送用户名和密码。
2. 服务器验证通过后，在当前对话（session）里面保存相关数据，比如用户角色登录时间等等。
3. 服务器向用户返回一个session_id，写入用户的Cookie。
4. 用户随后的每一次请求，都会通过 Cookie，将session_id传回服务器。
5. 服务器收到session_id，找到前期保存的数据，由此得知用户的身份。
{{< /admonition >}}

这样大家很嗨皮了，可是服务器就不嗨皮了，每个人只需要保存自己的session id，而服务器要保存所有人的session id！如果访问服务器多了，就得由成千上万，甚至几十万个。

这对服务器说是一个巨大的开销，严重的限制了服务器扩展能力，比如说我用两个机器组成了一个集群，小F通过机器A登录了系统，那session id会保存在机器A上，假设小F的下一次请求被转发到机器B怎么办？  机器B可没有小F的session id啊。

有时候会采用一点小伎俩：session sticky，就是让小F的请求一直粘连在机器A上，但是这也不管用，要是机器A挂掉了，还得转到机器B去。
那只好做session的复制了，把session id在两个机器之间搬来搬去，快累死了。

{{<image src="http://qiniustorage.joyinn.top/20200331191538.png" title="session sticky">}}


后来有个叫Memcached的支了招： 把session id集中存储到一个地方，所有的机器都来访问这个地方的数据，这样一来，就不用复制了，但是增加了单点失败的可能性，要是那个负责session 的机器挂了，所有人都得重新登录一遍，估计得被人骂死

{{<image src="http://qiniustorage.joyinn.top/20200331191630.png" title="session集中存放">}}

## token

于是有人就一直在思考，我为什么要保存这可恶的session呢，只让每个客户端去保存该多好？
可是如果不保存这些session id，怎么验证客户端发给我的session id的确是我生成的呢？如果不去验证，我们都不知道他们是不是合法登录的用户，那些不怀好意的家伙们就可以伪造session id，为所欲为了。
嗯，对了，关键点就是验证 ！

比如说，小F已经登录了系统，我给他发一个令牌(token)，里边包含了小F的user id，下一次小F再次通过Http请求访问我的时候，把这个token通过Http header带过来不就可以了。

不过这和session id没有本质区别啊，任何人都可以可以伪造，所以我得想点儿办法，让别人伪造不了。

那就对数据做一个签名吧，比如说我用SHA256算法，加上一个只有我才知道的密钥，对数据做一个签名，把这个签名和数据一起作为token，由于密钥别人不知道，就无法伪造token了。

{{<image src="http://qiniustorage.joyinn.top/20200331191652.png" title="hash加密得到token">}}

这个token 我不保存，当小F把这个token 给我发过来的时候，我再用同样的HMAC-SHA256 算法和同样的密钥，对数据再计算一次签名，和token 中的签名做个比较，如果相同，我就知道小F已经登录过了，并且可以直接取到小F的user id，如果不相同，数据部分肯定被人篡改过，我就告诉发送者：对不起，没有认证。

{{<image src="http://qiniustorage.joyinn.top/20200331191727.png" title="token验证">}}

Token 中的数据是明文保存的（虽然我会用Base64做下编码，但那不是加密），还是可以被别人看到的，所以我不能在其中保存像密码这样的敏感信息。

当然，如果一个人的token被别人偷走了，那我也没办法，我也会认为小偷就是合法用户，这其实和一个人的session id被别人偷走是一样的。

这样一来，我就不保存session id了，我只是生成token，然后验证token，我用我的CPU计算时间获取了我的session存储空间 ！

## cookie

cookie 是一个非常具体的东西，指的就是浏览器里面能永久存储的一种数据，仅仅是浏览器实现的一种数据存储功能。

cookie由服务器生成，发送给浏览器，浏览器把cookie以kv形式保存到某个目录下的文本文件内，下一次请求同一网站时会把该cookie发送给服务器。

由于cookie是存在客户端上的，所以浏览器加入了一些限制确保cookie不会被恶意使用，同时不会占据太多磁盘空间，所以每个域的cookie数量是有限的。

## Token的身份验证

{{< admonition digest "流程" false >}}
1. 用户通过用户名和密码发送请求。
2. 程序验证。
3. 程序返回一个签名的token 给客户端。
4. 客户端储存token，并且每次用于每次发送请求。
5. 服务端验证token并返回数据。
{{< /admonition >}}

{{<image src="http://qiniustorage.joyinn.top/20200331191802.png" title="token验证">}}

## JWT

JSON Web Token（缩写 JWT）是目前最流行的跨域认证解决方案。

### JWT的原理

JWT 的原理是，服务器认证以后，生成一个 JSON 对象，发回给用户。以后，用户与服务端通信的时候，都要发回这个 JSON 对象。服务器完全只靠这个对象认定用户身份。为了防止用户篡改数据，服务器在生成这个对象的时候，会加上签名。

JWT 的数据结构
它是一个很长的字符串，中间用点（.）分隔成三个部分。

JWT 的三个部分依次如下。

{{<image src="http://qiniustorage.joyinn.top/20200331191820.png" title="token验证">}}

Header 部分是一个 JSON 对象，描述 JWT 的元数据，
Payload 部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段供选用。
Signature 部分是对前两部分的签名，防止数据篡改。

## 项目实例

我们现在看下项目中的应用实例

### 生成token

先来看看后端是如何生成一个token的

```javascript
router.post("/login", async function(req, res, next) {
  const { username, password } = req.body;
  const userinfo_pw = await query(my_user_info_with_password, [username]);
  const md5_pw = md5(password);
  if (userinfo_pw.length === 0) {
    res.json({
      code: 4001,
      msg: "no such user"
    });
  } else if (md5_pw === userinfo_pw[0].password) {
    const token = jwt.sign(
      {
        iss: "joyinn",
        aud: username,
        uid: userinfo_pw[0].uid
      },
      myconfig.jwtSecret,
      {
        // 授权时效1day
        expiresIn: 60 * 60 * 24
      }
    );
    // update last login time
    await query(update_logintime, [userinfo_pw[0].uid]);
    // get userinfo
    const userinfo = await query(my_user_info, [username]);
    res.json({
      code: 0,
      msg: "login ok",
      token,
      user: userinfo[0]
    });
  } else {
    res.json({
      code: 4002,
      msg: "wrong password"
    });
  }
});
```

当响应login handler，比对密码正确后，调用`jsonwebtoken`依赖库生成token

```javascript
const jwt = require("jsonwebtoken");
const token = jwt.sign(
    {
    iss: "joyinn",
    aud: username,
    uid: userinfo_pw[0].uid
    },
    myconfig.jwtSecret,
    {
    // 授权时效1day
    expiresIn: 60 * 60 * 24
    }
);
```

这里`jwt.sign(payload, secretOrPrivateKey, [options, callback])`，具体文档可以参考[jsonwebtoken](https://github.com/auth0/node-jsonwebtoken)

### 判断token正确性

依旧是后端，我们怎么判断用户的请求携带的登录态信息是正确的
根据上面陈述的token原理，我们只需要利用secret对JWT的前两段加密得到签名，比对JWT给出的签名即可

具体实现我们可以采用express-jwt依赖库，会对headers中的Authorization字段的token进行检验

```javascript
// filename: app.js
const expressJWT = require("express-jwt");

// ... other code

// 设置允许跨域访问该服务.
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Methods", "GET,HEAD,OPTIONS,POST,PUT");
  res.header(
    "Access-Control-Allow-Headers",
    "Origin, X-Requested-With, Content-Type, Accept, Authorization"
  );
  next();
});
// jwt auth
app.use(
  expressJWT({
    secret: myconfig.jwtSecret
  }).unless({
    path: [
      "/user/login",
      "/user/register",
      "/user/mailtozju",
      "/user/checkvalidcode"
    ] //除了这个地址，其他的URL都需要验证
  })
);
```

我们这里将jwt验证中间件放在url响应函数之前，同时通过unless参数设置不需要jwt检验的api，实现了一个拦截层中间件


### 访问jwt内部payload

中间件会将验证解析好的数据放在request里面，比如我们在handler里面可以通过req.user访问到token的所有信息

```javascript
// post one say
router.post("/", async (req, res) => {
  const { photo, say_text } = req.body;
  const uid = req.user.uid;
  console.log(req.user);

  const photoNum = JSON.parse(photo).length;
  console.log("photoNum", photoNum);
  let type;
  if (photoNum === 0) type = 0;
  else if (say_text === "") type = 1;
  else type = 2;
  const insertResult = await query(post_say, [type, say_text, photo, uid, 1]);
  res.json({
    code: 0,
    msg: "insert success",
    insertId: insertResult.insertId
  });
});
```

### 前端http通信的设置

这里我的实例项目是`vuejs`，http通信采用了`axios`，作为一个plugin被调用

我们为了实现jwt加载，需要在发送数据包和接收数据包的时候均对axios模块添加一些逻辑

```javascript
// 在发送之前检查localstorage里面有无token字段，如果有就写入Authorization字段
_axios.interceptors.request.use(
    function(config) {
        const my_token = window.localStorage.getItem("token");
        if (my_token) {
        config.headers["Authorization"] = `Bearer ${my_token}`;
        }
        return config;
    },
    function(error) {
        return Promise.reject(error);
    }
);
// 在收到以后检查response中有无token字段，如果有就将token写入到localstorage中
// 如果响应发生了错误（一般是由于在未登录状态下访问api、或者token有误导致被express-jwt拦截）
// 那么就移除token并且跳转到登录页面
_axios.interceptors.response.use(
    function(response) {
      if (response.data.token) {
        window.localStorage.setItem("token", response.data.token);
      }
      return response;
    },
    function(error) {
      const errRes = error.response;
      if (errRes.status === 401) {
        window.localStorage.removeItem("token");
        router.push("/login");
      }
      return Promise.reject(error);
    }
);
```

我对vue默认的axios plugin做了二次封装，添加了传入自定义config以及token支持，详细代码可见[gist](https://gist.github.com/wkk5194/26ce5ecaeaf08c0e428c4533c9844217)

{{< gist wkk5194 26ce5ecaeaf08c0e428c4533c9844217 >}}

### bonus: 路由守卫

我们有了用户登录态，对vue路由守卫实践起来自然就容易了

```javascript
Vue.use(Router);
let router = new Router({
  mode: "history",
  base: process.env.BASE_URL,
  routes: [
    {
      path: "/",
      name: "home",
      component: Home,
      meta: {
        requireAuth: true
      }
    },
    {
      path: "/login",
      name: "login",
      component: Login,
      meta: {
        requireAuth: false
      }
    },
    // ... more routes
  ]
});
router.beforeEach((to, from, next) => {
  const token = localStorage.getItem("token") || null;

  if (to.matched.some(record => record.meta.requireAuth)) {
    if (!Auth.loggedIn(token)) {
      next("/login");
    } else {
      if (!store.state.user.isLogin) {
        axios.get("/user/getuserinfo").then(res => {
          store.dispatch("setUser", res.data.user);
          next();
        });
      } else next();
    }
  } else if (to.path === "/login" && Auth.loggedIn(token)) {
    next("/");
  } else {
    next();
  }
});
export default router;
```

我们对路由的meta对象加入requireAuth，设置布尔值表示是否需要路由守卫

接下来在`router.beforeEach`函数中对token进行检验，剩余对state更新以及路由跳转等逻辑我想不需要再赘述

### btw

如果对这一工程实例感兴趣的话可以查看[joyinn](https://github.com/ZJU-PAPIC/joyinn)

这是一个论坛的demo，采用expressjs+mysql+vue全家桶，说不定你能有所启发呢！
