---
title: 我的第一个Node项目(学习记录)
date: 2021-03-19 15:16:10
tags: Node
categories:
  - Node
---

## 以人为镜可以明得失 以代码为镜可以通逻辑

<!-- more -->
## 1、创建文件夹，初始化项目

```
mkdir koa-app # 创建目录
cd koa-app    # 进入项目
npm init -y # 生成package.json
npm install koa koa-router koa-bodyparser  mysql2 sequelize require-directory nodemon # 下载所需依赖包
touch app.js # 生成入口文件
```

## 2、定义项目启动命令

```
{
  "name": "koa-app",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "start:dev": "nodemon app.js",
    "start:prod": "node app.js"
  },
  "author": "Joker",
  "license": "ISC",
  "dependencies": {
    "koa": "^2.13.1",
    "koa-router": "^10.0.0",
    "mysql2": "^2.2.5",
    "require-directory": "^2.1.1",
    "sequelize": "^6.5.1"
  },
  "devDependencies": {
    "koa-bodyparser": "^4.3.0",
    "nodemon": "^2.0.7"
  }
}

```

## 3、定义入口文件(app.js)

```app.js
const Koa = require('koa')
const bodyParser = require('koa-bodyparser')

const InitManger = require('./core/init')
const catchErros = require('./middlewares/exception')

const app = new Koa()

// 全局错误处理
app.use(catchErros)
app.use(bodyParser())

// 初始化导入路由
InitManger.initCore(app)


const PORT = process.env.PORT || 3000
app.listen(PORT, () => {
  console.log(`Server is runing on ${PORT}!`)
})

```

## 4、sequelize 连接 mysql

#### 安装相关依赖

```
  npm install mysql2 sequelize -S
```

#### 创建数据库配置文件

```
  mkdir config
  touch config/db.js
```

#### 定义数据库连接配置

```
// config/db.js

 module.exports = {
  database: {
    host: 'localhost',
    database: 'dbName',
    user: 'root',
    password: 'root-password'
  }
}
```

#### 定义 sequelize 文件

```
// config/index.js

const Sequelize = require('sequelize')
const { host, database, user, password } = require('../config/config').database


const sequelize = new Sequelize(database, user, password, {
  host,
  dialect: 'mysql',
  logging: true,
  timezone: '+08:00',
})

sequelize.sync({
  force: true,
})

module.exports = sequelize

```

#### 定义 model

```
mkdir models
touch models/User.js
```

```
const { Sequelize, Model } = require('sequelize')
const { sequelize } = require('../config/sequelize')

class User extends Model { }

User.init({
  nickname: Sequelize.STRING(32),
  age: Sequelize.INTEGER(3),
  sex: Sequelize.STRING(1)
}, {
  sequelize,
  tableName: 'user'
})

module.exports = User

```

## 5、定义路由

```
mkdir app/api/v1/
touch app/api/v1/user.js
```

```
const Router = require('koa-router')
const router = new Router({
  prefix: '/v1/user'
})

router.get('/post', async (ctx) => {
  const body = await ctx.request.body

  await User.create(body)
})

module.exports = router
```

## 6、定义全局异常处理中间件

```
mkdir middlewares
touch middlewares/exception.js
touch utils/http-exception.js # 定义已知异常类
```

```
const catchError = async (ctx, next) => {
  try {
    await next()
  } catch (error) {
    console.log(error, 'this is error...')
  }
}

module.exports = catchError
```

### 6.1、明确已知错误还是未知错误

```
// utils/http-exception.js
/**
 * 默认的异常
 */
class HttpException extends Error {
  constructor(msg = '错误请求', errorCode = 10000, code = 400) {
    super()
    this.errorCode = errorCode
    this.code = code
    this.msg = msg
  }
}

class ParameterException extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 400
    this.msg = msg || '参数错误'
    this.errorCode = errorCode || 10000
  }
}

class AuthFailed extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 401
    this.mag = msg || '授权失败'
    this.errorCode = errorCode || 10004
  }
}

class NotFound extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 404
    this.msg = msg || '未找到该资源'
    this.errorCode = errorCode || 10005
  }
}

class Forbidden extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 403
    this.msg = msg || '禁止访问'
    this.errorCode = errorCode || 10006
  }
}

class Oversize extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 413
    this.msg = msg || '上传文件过大'
    this.errorCode = errorCode || 10007
  }
}

class InternalServerError extends HttpException {
  constructor(msg, errorCode) {
    super()
    this.code = 500
    this.msg = msg || '服务器出错'
    this.errorCode = errorCode || 10008
  }
}

module.exports = {
  HttpException,
  ParameterException,
  AuthFailed,
  NotFound,
  Forbidden,
  Oversize,
  InternalServerError
}

```

### 6.2、 定义异常处理中间件

```
// middlewares/exception.js
const { HttpException } = require('../utils/http-exception')

// 全局异常监听
const catchError = async(ctx, next) => {
  try {
    await next()
  } catch(error) {
    // 已知异常
    const isHttpException = error instanceof HttpException
    // 开发环境
    const isDev = global.config.service.enviroment === 'dev'

    // 在控制台显示未知异常信息：开发环境下，不是HttpException 抛出异常
    if (isDev && !isHttpException) {
      throw error
    }

    /**
     * 是已知错误，还是未知错误
     * 返回：
     *      msg 错误信息
     *      error_code 错误码
     */
    if (isHttpException) {
      ctx.body = {
        msg: error.msg,
        error_code: error.errorCode
      }
      ctx.response.status = error.code
    } else {
      ctx.body = {
        msg: '未知错误',
        error_code: 9999
      }
      ctx.response.status = 500
    }
  }
}

module.exports = catchError

```

## 7、全局自动注册路由

```
touch utils/initRouter.js
```

```
const requireDirectory = require('require-directory')
const Router = require('koa-router')

class InitManger {
  static initCore(app) {

    InitManger.app = app
    InitManger.initLoadRouters()
  }

  static initLoadRouters() {
    const apiDirectory = `${process.cwd()}/app/api`
    requireDirectory(module, apiDirectory, {
      visit: whenLoadModule
    })

    function whenLoadModule(obj) {
      if (obj instanceof Router) {
        InitManger.app.use(obj.routes())
      }
    }
  }
}


module.exports = InitManger
```

## 8、访问接口

访问接口地址 localhost:3000/api/v1/user/add
参数: {"nickname": "JOKER","age": "18","sex": "男"}

```
{
    "message": "用户添加成功",
    "code": 10000
}
```
