---
title: About User
date: 2026-03-12
description: 微服务User部分思路讲解
#image: 1.png
categories:
  - 后端
  - 微服务
  - 项目实战

tags:
  - go-zero
  - User服务
  - Admin模块
  - API & RPC
  - Model
  - GORM
  - PostgreSQL
  - JWT
  - Snowflake
  - bcrypt
---



本篇文章主要是关于 **User 服务** (包括 user 和 admin 模块) 开发过程中的技术分享。

## 1. 开发流程与架构设计

### 1.1 前期准备

我们利用 **go-zero** 进行微服务开发，第一步是写好 `api` 和 `proto` 文件。这就需要提前构思好微服务中所涉及到的功能。

- **举例**：在 admin 部分设计了重置用户密码 (resetuserpassword)功能，我们就需要先设计好对应的 `Req` (请求) 和 `Resp` (响应)，再利用 `goctl` 语句生成其余的代码文件。

### 1.2 三层架构解析

本服务大致分为三层—— **API 层**、**RPC 层** 以及 **Model 层**，三层分工明确，不能混淆各个层之间的功能：

- **API 层**：用来和前端对接。负责将参数发送给 RPC 层，并将 RPC 层的结果返回给前端。**注意：该层不应涉及到任何与数据库有关的操作。**
- **RPC 层**：负责业务逻辑处理。将 API 层得到的数据发送给 Model 层进行处理，并将处理结果返回给 API 层。
- **Model 层**：负责数据持久化。处理 RPC 层发来的请求并操作数据库，包括服务中所涉及的数据库表设计文件、migration 操作以及数据库的增删查改等操作。

## 2. 配置文件详解 (.yaml 与 config)

配置文件需要将 API 和 RPC 分开讨论。

### 2.1 RPC 层配置

RPC 层一般需要配置日志、数据库、缓存 Redis、消息队列 Kafka 和 Etcd 等相关信息。以 user 部分为例(`user.yaml`)：

```YAML
Name: user.rpc
ListenOn: 0.0.0.0:8080
#日志信息
Log:
  ServiceName: user-rpc
  Mode: file
  Path: ../../../../log/user-rpc
#数据库信息
Postgres:
  Host: "127.0.0.1"
  Port: "35432"         #端口号
  User: "admin"         #用户名
  Password: "Sea-TryGo" #数据库密码
  DBName: "first_db"    #数据库名称
  Mode: "disable"    
#Etcd信息
Etcd:
  Hosts:
    - 127.0.0.1:32379
  Key: user.rpc
```

对应的 **Config 代码 (**`config.go`**)** 相对简单，只需填入响应的信息：

```Go
package config

import "github.com/zeromicro/go-zero/zrpc"

type Config struct {
    zrpc.RpcServerConf
    // 配置数据库信息
    Postgres struct {
        Host     string
        Port     string
        User     string
        Password string
        DBName   string
        Mode     string
    }
    // 可能还有其他配置,例如Redis,Kafka等
}
```

### 2.2 API 层配置

API 层一般需要配置日志、鉴权、涉及到的 RPC 服务等相关信息。以 user(`user-api.yaml`) 部分为例：

```YAML
Name: usercenter
Host: 0.0.0.0
Port: 8888
#日志信息
Log:
  ServiceName: usercenter
  Mode: file
  Path: ../../../../log/user  
#鉴权信息
UserAuth:
 AccessSecret: ae0536f9-6450-4606-8e13-5a19ed505da0 #密钥
 AccessExpire: 604800								#过期时间设为一周,86400*7=604800
#涉及到的UserRpc服务
UserRpc:
 Etcd:
  Hosts:
   - 127.0.0.1:32379
  Key: user.rpc
```

对应的 **Config 代码 ** (`config.go`) 需要填入鉴权信息及 RPC 客户端配置：

```Go
package config

import (
    "github.com/zeromicro/go-zero/rest"
    "github.com/zeromicro/go-zero/zrpc"
)

type Config struct {
    rest.RestConf
    // 配置鉴权信息
    UserAuth struct {
        AccessSecret string
        AccessExpire int64
    }
    // 与Rpc部分进行连接
    UserRpc zrpc.RpcClientConf
}
```

### 2.25 Model 层配置

在数据库连接之前，我们需要先完成 Model 层数据访问对象（**DAO**）的定义，以便后续将底层的 GORM 实例注入其中。

```Go
package model

import "gorm.io/gorm"

type UserModel struct {
    conn *gorm.DB
}

//新建User部分实例
func NewUserModel(db *gorm.DB) *UserModel {
    return &UserModel{
        conn: db,
    }
}
```

## 3. 依赖注入 (ServiceContext)

### 3.1 RPC 部分

RPC 需要连接数据库、Redis、Kafa等：

```Go
package svc

import (
    "sea-try-go/service/user/user/rpc/internal/config"
    "sea-try-go/service/user/user/rpc/internal/model"
)

type ServiceContext struct {
    Config    config.Config
    //数据库配置
    UserModel *model.UserModel
    //可能还有Redis配置、Kafka配置等
}

func NewServiceContext(c config.Config) *ServiceContext {
    dbConfig := model.DBConf{
        Host:     c.Postgres.Host,
        Port:     c.Postgres.Port,
        User:     c.Postgres.User,
        Password: c.Postgres.Password,
        DBName:   c.Postgres.DBName,
        Mode:     c.Postgres.Mode,
    }
    //数据库连接以及初始化
    db := model.InitDB(dbConfig)
    return &ServiceContext{
        Config:    c,
        UserModel: model.NewUserModel(db),
    }
}
```

### 3.2 API 部分

API 需要建立与 RPC 的连接：

```Go
package svc

import (
    "sea-try-go/service/user/user/api/internal/config"
    "sea-try-go/service/user/user/rpc/userservice"

    "github.com/zeromicro/go-zero/zrpc"
)

type ServiceContext struct {
    Config  config.Config
    //Rpc连接
    UserRpc userservice.UserService
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:  c,
        UserRpc: userservice.NewUserService(zrpc.MustNewClient(c.UserRpc)),
    }
}
```

## 4. 核心逻辑实现（以用户注册为例）

核心逻辑部分遵循 "**1.2 三层架构解析**" 中提到的分工进行。

### 4.1 API 层逻辑

```Go
func (l *RegisterLogic) Register(req *types.CreateUserReq) (resp *types.CreateUserResp, code int) {
    //接收前端发来的数据并打包
    rpcReq := &pb.CreateUserReq{
        Username:  req.Username,
        Password:  req.Password,
        Email:     req.Email,
        ExtraInfo: req.Extrainfo,
    }
    
    //把数据扔给rpc然后得到结果
    rpcResp, err := l.svcCtx.UserRpc.Register(l.ctx, rpcReq)
    
	//处理错误类型
    if err != nil {
        // ...
    }
    
    return &types.CreateUserResp{
        Uid: rpcResp.Uid,
    }, errmsg.Success
}
```

### 4.2 RPC 层逻辑

注意：`username` 在数据库中设置了唯一索引 unique，防止并发时出现问题。

```Go
func (l *RegisterLogic) Register(in *pb.CreateUserReq) (*pb.CreateUserResp, error) {
    //用户名查重
    
    _, err := l.svcCtx.UserModel.FindOneByUserName(l.ctx, in.Username)
    
    //处理数据库错误
    if err == nil {
        //...
    }
    
    //处理用户已存在的逻辑
    if err != model.ErrorNotFound {
        //...
    }

    // 密码加密
    truePassword, err := cryptx.PasswordEncrypt(in.Password)
    if err != nil {
        //...
    }

    // 生成 UID
    var uid int64
    uid, err = snowflake.GetID()
    if err != nil {
        //...
    }

    // 数据填入数据库
    newUser := model.User{
        Uid:       uid,
        Username:  in.Username,
        Password:  truePassword,
        Email:     in.Email,
        Score:     0,
        Status:    0,
        ExtraInfo: in.ExtraInfo,
    }
    //插入数据库
    err = l.svcCtx.UserModel.Insert(l.ctx, &newUser)
    if err != nil {
        //...
    }
    
    return &pb.CreateUserResp{
        Uid: newUser.Uid,
    }, nil
}
```

### 4.3 Model 层逻辑

Model 层涉及到数据库操作，为了提高开发效率，使用的是 **GORM** 框架而不是 sqlx。

- **数据库初始化与 Migration：**

```Go
func InitDB(conf DBConf) *gorm.DB {
    dsn := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%s sslmode=%s TimeZone=Asia/Shanghai",
        conf.Host,
        conf.User,
        conf.Password,
        conf.DBName,
        conf.Port,
        conf.Mode,
    )
    
    //数据库连接
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalln("数据库连接失败")
    }
    
    //数据库自动迁移
    err = db.AutoMigrate(&User{})
    if err != nil {
        log.Fatalln("数据表迁移失败")
    }
    return db
}
```

- **CRUD 操作：**

```Go
// 根据用户名 username 查找用户
func (m *UserModel) FindOneByUserName(ctx context.Context, username string) (*User, error) {
    var user User
    err := m.conn.WithContext(ctx).Where("username = ?", username).First(&user).Error
    if err == nil {
        return &user, nil
    }
    if err == gorm.ErrRecordNotFound {
        return nil, ErrorNotFound
    }
    return nil, err
}

// 插入 user 数据
func (m *UserModel) Insert(ctx context.Context, user *User) error {
    err := m.conn.WithContext(ctx).Create(user).Error
    return err
}
//...
```

## 5. 关键技术点与避坑细节

### 5.1 ID 的选择：雪花算法

在本服务一开始，本人使用的是数据库自增 ID，但是数据库自增的 ID 存在泄露数据库总量这个敏感信息的风险。于是将所有的 ID 全部改为了 `uid`，并且使用了统一的 **雪花算法 (Snowflake)** 进行生成，保证了安全性。后续只需要利用 common 下的 `GetID()` 函数就可以得到全局唯一的ID。

```Go
//由于篇幅限制，其余的代码就不贴了，有关Snowflake算法的内容读者可以自行搜索
// GetID 生成全局唯一ID
func GetID() (int64, error) {
    Init()
    if sfErr != nil {
        return 0, sfErr
    }
    if sf == nil {
        return 0, fmt.Errorf("snowflake node is not initialized")
    }
    return sf.Generate().Int64(), nil
}
```

### 5.2 密码安全：Bcrypt 加密

密码作为极端敏感的信息，必然不能将原始值放入数据库，而是需要进行哈希加密。本人采用的是 `Go` 语言中 `bcrypt` 库进行加密：

```Go
import (
    "golang.org/x/crypto/bcrypt"
)

//生成加密后的密码
func PasswordEncrypt(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return "", err
    }
    return string(bytes), nil
}

//比对密码,用登录时判断密码是否正确
func CheckPassword(dbHash, inputPassword string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(dbHash), []byte(inputPassword))
    return err == nil
}
```

- `PasswordEncrypt` 是注册或修改密码时调用的逻辑。`bcrypt` 算法非常智能，它能够自动生成随机 **盐 (Salt)** 并将其藏在生成的字符串里。这意味着即使是两个**相同的原始密码** "123456"，**加密后的字符串仍然是不一样**的。
- `bcrypt` 故意设计得很慢（计算成本 `bcrypt.DefaultCost` 为 10），加密一次可能需要几十毫秒，能有效防止黑客暴力破解。

### 5.3 鉴权机制：JWT

本服务中使用 **JWT (JSON Web Token)** 进行鉴权。

#### **生成 Token：**

在 `common` 包中实现生成逻辑：

```Go
//参数包含签发时间、过期时间、用户Id等信息
func GetToken(secretKey string, iat, seconds int64, userId int64) (string, error) {
    claims := make(jwt.MapClaims)
    claims["exp"] = iat + seconds
    claims["iat"] = iat
    claims["userId"] = userId
    token := jwt.New(jwt.SigningMethodHS256)
    token.Claims = claims
    return token.SignedString([]byte(secretKey))
}
```

登录时调用：

```Go
now := time.Now().Unix() //获得签发时间
accessSecret := l.svcCtx.Config.UserAuth.AccessSecret //获得密钥
accessExpire := l.svcCtx.Config.UserAuth.AccessExpire //获得过期时间
token, err := jwt.GetToken(accessSecret, now, accessExpire, int64(rpcResp.Uid)) //生成 token
```

#### **解析 Token (踩坑点)：**

除了注册外，其他逻辑都需要登录后才能进行。我们可以通过 JWT 获取当前登录用户的 `uid`。

因为 `GetToken` 代码中我们把 `userId` 作为一个关键字放进去了，取出 uid 信息的代码如下：

```Go
userId, ok := l.ctx.Value("userId").(json.Number)
uid, err := userId.Int64()
```

**注意**：

1. `l.ctx.Value("userId")` 中的 key 必须与 claims 中的 key 对应。
2. 在 Go-Zero 的 JWT 中间件中，**解析出来的数字默认是** **`json.Number`** **类型**，而不是 `int64` 或 `float64`。这是为了防止大整数（如雪花 ID）在 JSON 传输中丢失精度。后续想要得到 `int64` 类型的 `uid` 的话，必须通过 `.Int64()` 进行转换。

## 6. 服务启动与测试

### 6.1 启动服务

首先启动基础设施：

```Bash
docker-compose up -d
```

然后分别启动 RPC 和 API 服务（以 user 为例）：

```Bash
# 启动 RPC
cd "Address\Sea-TryGo\service\user\user\rpc"
go run user.go -f etc/user.yaml #对应入口.go文件以及etc下的.yaml配置文件
# 启动 RPC
cd "Address\Sea-TryGo\service\user\user\rpc"
go run user.go -f etc/user.yaml
```

### 6.2 Apifox 接口测试

- **POST 类型接口测试（以** **`user`** **部分的** **`register`** **功能为例）：** 
- 我们需要输入对应的网址 `http://127.0.0.1:8888/usercenter/v1/user/register`。因为是 `POST` 类型，我们需要在页面中选择 **Body** 选项卡，然后切换到 **JSON** 格式并在里面输入参数，最后点击发送即可得到结果。

![img](https://raw.githubusercontent.com/unicornshjl/MyPic/img/pic1.png)

- **GET 类型接口测试（以** **`admin`** **部分的** **`getuser`** **功能为例）：** 
- 由于该接口是 `GET` 类型，传参方式有所不同，我们需要到 **Params** 栏去填写对应的查询参数。

![img](https://raw.githubusercontent.com/unicornshjl/MyPic/img/pic2.png)

- **JWT 鉴权配置：** 
- 由于系统鉴权方式采用的是 JWT，所以在涉及到**登录后才能进行的操作**时，我们需要进入接口的 **Auth** 栏，将鉴权方式选择为 **Bearer Token**，然后填入之前登录时获取到的 Token，即可正常发起请求。

![img](https://raw.githubusercontent.com/unicornshjl/MyPic/img/pic3.png)

**总结**：以上就是 user 部分开发的技术分享，涵盖了 API、RPC、Model 三层架构的协作，以及 ID 处理、密码加密、JWT 鉴权、日志系统等关键细节的实现与分析。