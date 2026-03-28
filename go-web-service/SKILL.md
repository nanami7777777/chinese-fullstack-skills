---
name: go-web-service
description: Use this skill when building Go web services or APIs. It provides Chinese-language best practices for Go web development with Gin or Echo framework, GORM database access, project layout following golang-standards, error handling, middleware patterns, and deployment conventions for Chinese development teams.
---

# Go Web 服务中文开发规范

## 项目结构（标准布局）

```
├── cmd/
│   └── server/
│       └── main.go             # 启动入口
├── internal/                   # 私有代码（不可被外部导入）
│   ├── handler/                # HTTP 处理器（类似 controller）
│   ├── service/                # 业务逻辑层
│   ├── repository/             # 数据访问层
│   ├── model/                  # 数据模型（GORM model）
│   ├── middleware/             # 中间件
│   ├── config/                 # 配置加载
│   └── pkg/                    # 内部公共包
│       ├── response/           # 统一响应
│       ├── errors/             # 错误定义
│       └── logger/             # 日志
├── api/                        # API 文档（OpenAPI/Swagger）
├── configs/                    # 配置文件
│   ├── config.yaml
│   └── config.prod.yaml
├── migrations/                 # 数据库迁移
├── scripts/                    # 脚本
├── Dockerfile
├── Makefile
├── go.mod
└── go.sum
```

## 命名规范

### 包命名
- 全小写，单个单词：`handler`、`service`、`model`
- 不用下划线或驼峰：`userservice` ✗ → `service` ✓
- 包名不要和标准库冲突

### 文件命名
- 全小写 + 下划线：`user_handler.go`、`order_service.go`
- 测试文件：`user_handler_test.go`

### 变量和函数
- 导出用 PascalCase：`GetUserByID`、`UserService`
- 私有用 camelCase：`getUserByID`、`userRepo`
- 接口名用 er 后缀：`Reader`、`UserRepository`
- 错误变量用 Err 前缀：`ErrUserNotFound`、`ErrInvalidToken`

## 分层架构

```go
// internal/handler/user_handler.go — HTTP 处理器
type UserHandler struct {
    svc service.UserService
}

func NewUserHandler(svc service.UserService) *UserHandler {
    return &UserHandler{svc: svc}
}

func (h *UserHandler) GetList(c *gin.Context) {
    var query model.UserQuery
    if err := c.ShouldBindQuery(&query); err != nil {
        response.BadRequest(c, "参数错误")
        return
    }

    result, err := h.svc.GetList(c.Request.Context(), &query)
    if err != nil {
        response.Error(c, err)
        return
    }

    response.OK(c, result)
}
```

```go
// internal/service/user_service.go — 业务逻辑
type UserService interface {
    GetList(ctx context.Context, query *model.UserQuery) (*model.PageResult[model.User], error)
    Create(ctx context.Context, input *model.CreateUserInput) (*model.User, error)
}

type userService struct {
    repo repository.UserRepository
}

func NewUserService(repo repository.UserRepository) UserService {
    return &userService{repo: repo}
}

func (s *userService) GetList(ctx context.Context, query *model.UserQuery) (*model.PageResult[model.User], error) {
    return s.repo.FindPage(ctx, query)
}

func (s *userService) Create(ctx context.Context, input *model.CreateUserInput) (*model.User, error) {
    // 检查邮箱是否已存在
    existing, _ := s.repo.FindByEmail(ctx, input.Email)
    if existing != nil {
        return nil, errors.New(errors.CodeConflict, "该邮箱已被注册")
    }

    user := &model.User{
        Name:  input.Name,
        Email: input.Email,
    }
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, err
    }
    return user, nil
}
```

## 统一响应

```go
// internal/pkg/response/response.go
package response

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

func OK(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{
        Code:    0,
        Message: "成功",
        Data:    data,
    })
}

func BadRequest(c *gin.Context, msg string) {
    c.JSON(http.StatusBadRequest, Response{
        Code:    40000,
        Message: msg,
    })
}

func Error(c *gin.Context, err error) {
    // 根据错误类型返回不同状态码
    var appErr *errors.AppError
    if stderrors.As(err, &appErr) {
        c.JSON(appErr.HTTPStatus(), Response{
            Code:    appErr.Code,
            Message: appErr.Message,
        })
        return
    }

    c.JSON(http.StatusInternalServerError, Response{
        Code:    50000,
        Message: "服务器内部错误",
    })
}
```

## 错误处理

```go
// internal/pkg/errors/errors.go
package errors

const (
    CodeBadRequest   = 40000
    CodeUnauthorized = 40100
    CodeForbidden    = 40300
    CodeNotFound     = 40400
    CodeConflict     = 40900
    CodeInternal     = 50000
)

type AppError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func (e *AppError) Error() string { return e.Message }

func (e *AppError) HTTPStatus() int {
    switch {
    case e.Code >= 50000: return 500
    case e.Code >= 40900: return 409
    case e.Code >= 40400: return 404
    case e.Code >= 40300: return 403
    case e.Code >= 40100: return 401
    default: return 400
    }
}

func New(code int, msg string) *AppError {
    return &AppError{Code: code, Message: msg}
}
```

## 配置管理

```go
// internal/config/config.go
package config

import "github.com/spf13/viper"

type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    JWT      JWTConfig      `mapstructure:"jwt"`
    Redis    RedisConfig    `mapstructure:"redis"`
}

type ServerConfig struct {
    Port int    `mapstructure:"port"`
    Mode string `mapstructure:"mode"` // debug / release
}

type DatabaseConfig struct {
    DSN             string `mapstructure:"dsn"`
    MaxOpenConns    int    `mapstructure:"max_open_conns"`
    MaxIdleConns    int    `mapstructure:"max_idle_conns"`
    ConnMaxLifetime int    `mapstructure:"conn_max_lifetime"` // 秒
}

func Load(path string) (*Config, error) {
    viper.SetConfigFile(path)
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

## 中间件

```go
// JWT 认证中间件
func AuthMiddleware(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            response.Unauthorized(c, "请先登录")
            c.Abort()
            return
        }

        token = strings.TrimPrefix(token, "Bearer ")
        claims, err := jwt.ParseToken(token, secret)
        if err != nil {
            response.Unauthorized(c, "登录已过期")
            c.Abort()
            return
        }

        c.Set("userID", claims.UserID)
        c.Next()
    }
}

// 请求日志中间件
func LoggerMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        logger.Info("请求完成",
            "method", c.Request.Method,
            "path", c.Request.URL.Path,
            "status", c.Writer.Status(),
            "latency", time.Since(start),
            "ip", c.ClientIP(),
        )
    }
}
```

## Makefile

```makefile
.PHONY: run build test lint migrate

run:
	go run cmd/server/main.go

build:
	CGO_ENABLED=0 go build -o bin/server cmd/server/main.go

test:
	go test ./... -v -cover

lint:
	golangci-lint run

migrate:
	go run cmd/migrate/main.go

docker:
	docker build -t myapp .
```
