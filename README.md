


##利用koa、MongoDB编写前后端分离的api
用到的技术：*nodeJs、mongodb、mongoose、koa、koa-router、koa-bodyparser、vue*

[1、前后端环境的搭建](#1前后端环境的搭建)

[2、安装mongoose并构建schema](#2安装mongoose并构建schema)

[3、用户操作的路由模块化](#3用户操作的路由模块化)

[4、编写前端注册页面](#4编写前端注册页面)

[5、安装koa-bodyparser处理post请求数据](#5安装koa-bodyparser处理post请求数据)

[6、将用户输入的数据存储进数据库](#6将用户输入的数据存储进数据库)

[7、前端处理后台处理过来的相关数据](#7前端处理后台处理过来的相关数据)

[8、用户登录界面的实现](#8用户登录界面的实现)


### 1前后端环境的搭建：
1. 在d盘目录下创建文件myApi
`cd d:/myApi`
2. 使用vue-cli脚手架搭建前端环境
`npm install vue-cli -g`
3. 下载webpack前端模板
`vue init webpack`

进入到myApi目录下运行`npm run dev`然后到浏览器中运行*http://localhost:8080* 出现vue的欢迎页面则表示你已经搭建成功啦。

4. 在myApi目录下创建service文件夹。并进入到service目录下`cd service`
初始化：`npm init -y`
初始化完成以后会生成一个package.json的文件
5. 安装koa到项目中
`npm install koa --save`
新建一个index.js的文件,并写入一个hello的例子
```javascript
var Koa = reuqire('koa')
var app = new Koa()
app.use( ctx => {
    ctx.body = 'hello koa'
})
app.listen(3000, () => {
    console.log('服务已经启动')
})
```
进入浏览器输入*http://localhost:3000* 出现 hello koa 说明你安装成功了。


### 2安装mongoose并构建schema
进入到service目录下，安装mongoose `npm install mongoose --save`，我安装的版本是：5.2.8。
**连接mongodb数据库**
新建：service/database/init.js 连接数据库文件
```javascript
const mongoose = require('mongoose') 
const db = 'mongodb://localhost/myapi'

exports.connect = () => {
    // 连接数据库
    mongoose.connect(db)
    // 数据库断开时
    mongoose.connection.on('disconnected', () => {
        console.log('**********数据库断开连接**********')
        // 断开重连数据库
        mongoose.connect(db)
    })

    // 当数据库发生错误
    mongoose.connection.on('error', (err) => {
        console.log('数据库发生错误')
        console.log(err);
        // 断开重连
        mongoose.connect(db)
    })

    // 打开数据库
    mongoose.connection.once('open', () => {
        console.log('mongoDB is connected !')
    })
}
```
在service/index.js中引入
```javascript
const {connect} = require ('./database/init.js')

// 立即执行函数。测试数据库是否连接成功
;(async () => {
    await connect();
})()
```
**构建schema**
在service/database/schema目录下创建User.js
```javascript
// 引入mongoose
const  mongoose = require('mongoose')
const Schema = mongoose.Schema
// 可提前声明userId类型
const ObjectId = Schema.Types.ObjectId
// 声明Object类型
let ObjectId = Schema.Types.ObjectId
/** 用户表
 * userId：用户id
 * userName： 用户名， 唯一
 * password： 用户密码
 * createAt: 创建时间 有默认值
 * lastLoginAt： 最后登录时间 有默认值
 * 
*/
var myUser = new Schema({
    userId: ObjectId,
    userName: { unique: true, type: String },
    password: String,
    createAt: {type: Date, default: Date.now() },
    lastLoginAt: {type: Date, default: Date.now() }
},{
    // 不加入这句话，在数据库会变成  myusers
    collection: 'myuser'
})

// 发布模型
mongoose.model('myuser', myUser)
```

**载入schema**
下载glob，并引入resolve。
> glob 允许你使用*等符号写一个规则，
> resolve 允许你将一些路径或将路径解析为绝对路径

`npm install glob --save`  使用版本为7.1.2   

在 service1/database/init.js中引入glob和resolve
```javascript
const glob = require('glob')
const {resolve} = require('path')

// 一次性引入database中的所有schema下的文件
exports.initSchemas = ()=> {
    glob.sync(resolve(__dirname, './schema/', '**/*.js')).forEach(require)
}
```
**手动插入数据**
在service/index.js中手动插入数据
```javascript
const Koa = require('koa')
const app = new Koa()

const mongoose = require('mongoose')
//引入initSchemas 
const {connect, initSchemas} = require('./database/init.js')

;(async () => {
    await connect()
    initSchemas();
    // 手动插入一个数据：用户名qqq, 密码 123456

    const myUser = mongoose.model('myuser')
    let oneUser = new myUser({
        userNamr: 'qqq',
        password: '123456'
    })
    oneUser.save().then(()=>{
        console.log('数据插入成功')
    })

     let users = await myuser.findOne({}).exec() // 检索user表中的第一个数据
     console.log(users)
})()
```
*注意哦：await一定要是在async中使用，否则会报错的哦*
> 解释一下async是什么：async作为一个关键字放在函数前面，表示函数是一个异步函数，这意味着该函数的执行不会影响到后面函数的执行。它返回的是一个promise对象。

### 3用户操作的路由模块化
安装koa-router
`npm install koa-router --save` 版本号：7.4.0
在service/api下新建一个myuser.js文件
```javascript
// 引入koa-router
const Router = require('koa-router')
let router = new Router();

router.get('/', async (ctx) => {
    ctx.body = '用户操作注册页面'
})

router.get('/register', async (ctx) => {
    ctx.body = "用户注册接口"
})

module.exports = router
```

在service/index.js中引入koa-router和myuser
```javascript
const Router = require('koa-router')
let router = new Router()

// 引入用户路由
const myuser = require('./api/myuser.js')
// 装载所有子路由
router.use('/user', myuser.routes())
// 加载中间件
app.use( router.routes())
app.use(router.allowedMethods())
```

运行: `node index.js` 在浏览器中输入 http://localhost:3000/user 和 http://localhost:3000/user/register 进行测试。

**配置serviceApi.config.js **
在src目录下创建serviceApi.config.js
```javascript
const LOCALHOST = 'http://localhost:3000/'

var url = {
    // 注册用戶地址
    register: LOCALHOST + 'user/register'
}

module.exports = url
```

### 4编写前端注册页面
在src/components/pages新建一个register.vue 页面
```html
<template>
    <div>
        <div class="username">
            用户名：<input type="text" v-model="userName" placeholder="请输入用户名">
        </div>
        <div class="pwd">
            <input type="password" v-model="password"
            placeholder="请输入密码" id="">
        </div>
        <div>
            <input type="button" value="立即注册">
        </div>
    </div>
</template>
<script>
    export default {
        data() {
            return {
                userName: '',
                password: ''
            }
        }
    }
</script>
```

配置路由。在router/index.js中配置注册页面的路由
```javascript
import register from '@/components/pages/register'

export default new Router({
  routes: [
    {
      path: '/',
      name: 'HelloWorld',
      component: HelloWorld
    },{
      path: '/register',
      name: 'register',
      component: register
    }
  ]
})
```
安装axios,并在register.vue页面中加入方法 axiosRegister
`npm install axios --save`
在register.vue中引入axios和serviceApi.config.js，并写入axiosRegister方法
```javascript
import axios from 'axios'
import url from '@/serviceApi.config.js'

export default {
    methods:{
        axiosRegister() {
            axios({
                // 配置好的注册接口路径
                url: url.register,
                // 请求方法
                method: 'post',
                data: {
                    userName: this.userName,
                    password: this.password
                }
            })
            .then( response => {
                console.log(response)
            })
            .catch( err => {
                console.log(err)
            })
        }
    }
}
```
在按钮上绑定方法
```html
<input type="button" @click="axiosRegister" value="马上注册">
```
### 5安装koa-bodyparser处理post请求数据
`npm install koa-bodyparser --save`
安装完成之后在service/index.js中引入koa-bodyparser并注册使用
*注意： body-parser要在route之前调用，否则会接收不到数据！因为你要先获取数据，然后通过路由。*
```javascript
require bodyParser = require('koa-bodyparser')
app.use(bodyParser())
```

修改service/pai/myuser.js中注册接口的方法
```javascript
// 将以前的get方法变成post方法
router.post('/register', async (ctx) => {
    // 打印用户请求的数据
    console.log(ctx.request.body)
    ctx.body = ctx.resquest.body
})
```
做好这一切之后，运行后端服务器：进入service目录下
```javascript
cd service
node index.js
```
运行前端vue
```javascript
cd myApi
npm run dev
```
进入浏览器输入 http://localhost:8080/register 输入用户名和密码，点击提交，在终端上可以看到你所输入的值，则表示成功了。

### 6将用户输入的数据存储进数据库
进入到service/api/myuser.js 中修改相关代码
```javascript
// 引入mongoose
const mongoose = require('mongoose')

router.post('/register', async (ctx) => {
    // 获取模型
    const User = mongoose.model('myuser')
    // 将前端获取到的数据封装成一个新的user对象
    let newUser = new User(ctx.request.body)
    // 将数据保存到数据库
    newUser.save().then( () => {
        // 注册成功，返回200 状态码，并返回注册成功的消息
        ctx.body = {
            code: 200,
            message: "注册成功"
        }
    })
    .catch( error => {
        // 注册失败，则返回500状态码，并返回注册失败的消息
        ctx.body = {
            code: 500,
            message: "注册失败"
        }
    })
})
```

### 7前端处理后台处理过来的相关数据

在register.vue中在页面展示相应的注册状态：成功或者失败，注册成功则提示成功，并返回到首页，注册失败则提醒用户注册失败
```javascript
axiosRegister() {
    axios({
        url: url.registerUser,
        method: 'post',
        data: {
            userName: this.userName,
            password: this.password
        }
    })
    .then(response => {
        if(response.data.code == 200) {
            // 注册成功，就简单地用一个alert弹框来表示一下啦
            alert('您已经注册成功')
            // 跳转到首页
            this.$router.push('/');
            
        } else {
            // 注册失败了
            alert('注册失败，请重新注册')
        }
    })
    .catch(err => {
        console.log('出错啦');
        alert('注册失败')
    })
}
```
注册页面已经完成啦！撒花花。

### 8用户登录界面的实现
1、 在pages目录下新建一个login.vue，其实和注册页面类似，你可以直接复制粘贴，然后改一些文字即可
```html
<div>
    <div class="username">
        用户名：<input type="text" v-model="userName" placeholder="请输入用户名">
    </div>
    <div class="pwd">
        密码：<input type="password" v-model="password"
        placeholder="请输入密码" id="">
    </div>
    <div>
        <input type="button" @click="axiosRegister" value="登录">
    </div>
</div>
```
2、在路由中注册页面，便于访问.进入到router/index.js
```javascript
// 引入login
import login from '@/components/pages/login'
export default new Router({
  routes: [
    {path: '/',name: 'ShoppingMall',component: ShoppingMall},
    {path: '/register',name: 'Register',component: Register},
    {path: '/login',name: 'Login',component: Login},
  ]
})
```
3、在api/myuser.js中编写后台登录的逻辑
```javascript
router.post('/login', async (ctx) => {
    // 获取输入的用户名和密码
    const userName = ctx.request.data.userName
    const password = crx.request.data.password
    // 打印一下是否正确
    console.log('username: '+ userName+ ', password: ' + password)
    // 获取model
    const User = mongoose.model('myuser')
    // 查询用户名是否正确
    User.findOne({userName: userName}).exec()
    .then(result=> {
        if(result) {
            if(result.password == password) {
                ctx.body = {
                    code: 200,
                    message: "登录成功"
                }
            } else {
                ctx.body = {
                    code: 500,
                    message: '密码错误'
                }
            }
        }
    })

})
```

一个简单的前后端分离，就完成啦。
