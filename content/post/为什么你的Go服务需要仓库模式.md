+++
date = '2026-04-26T23:00:15+08:00'
draft = false
title = '为什么你的Go服务需要仓库模式'
+++

在刚开始学习写 Go 服务的时候，把数据库的操作经常写在 handler 和 service 中。

起初，没觉得这种方式有什么问题而且效率还很高。随着业务和需求不断的增加，代码开始出现了各种问题：

- 单元测试如同水中月镜中花很难在项目中落地
- 业务逻辑和数据访问混在一起，被技术实现细节淹没
- 修改一个功能需要调整多个地方

出现这些问题的根本原因：业务逻辑和数据访问之间的边界不够清晰，导致数据访问这种变化在业务实现中随意扩散，系统也慢慢的走向了失控的边缘。

面对即将失控系统，有没有一种方式可以控制数据访问随意扩散，将这种变动限制在某一个范围内呢？

仓库模式应运而生。

### 仓库模式
仓库模式（Repository Pattern）是一种用于组织数据访问的设计方式。

它在业务逻辑和数据存储之间引入一层抽象，通过接口对外提供统一的数据操作能力，
而具体的数据存储实现被隐藏在接口之后。

在 Go 服务中，其结构与依赖关系如下：

                ┌────────────┐
                │  Handler   │
                └─────┬──────┘
                      │
                      ▼
                ┌────────────┐
                │  Service   │
                └─────┬──────┘
                      │  depends on
                      ▼
        ┌──────────────────────────────┐
        │   Repository (Interface)     │
        └───────────┬──────────────────┘
                    ▲
                    │ implements
     ┌──────────────┴──────────────┬──────────────┬──────────────┐
     │        MySQL Repo           │   Redis Repo │   API Repo   │
     └──────────────┬──────────────┴──────┬───────┴──────────────┘
                    │                     │
                    ▼                     ▼
               ┌────────┐           ┌────────┐
               │ MySQL  │           │ Redis  │
               └────────┘           └────────┘


从上到下各层有不同的职责划分：

- Handler 负责处理请求  
- Service 负责业务逻辑  
- Repository 接口定义数据访问能力  
- Repository 实现负责具体的数据存取（如 MySQL、Redis 或第三方 API）  

各层依赖关系：

- Service 依赖的是 Repository 接口，而不是具体实现  
- Repository 的具体实现依赖底层数据源  
- 整体依赖方向自上而下，具体实现被限制在最底层

可以看出，通过引入Repository (Interface) 层，业务领域和技术实现之间有了清晰的边界，数据访问被限制在了最底层。

在 Go 中，通过使用结构体和接口来实现这种模式。

### 反面例子
在刚开始写 Go 或者接手一些老代码，经常会把所有的代码都放在一个函数里，业务领域和技术实现堆在一起，缺乏结构设计。

``` go
func CreateUserHandler(c *gin.Context) {
	db, _ := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/testdb?parseTime=true")

	name := c.PostForm("name")

	if name == "" {
		c.String(400, "name required")
		return
	}

	_, err := db.Exec("INSERT INTO users(name) VALUES(?)", name)
	if err != nil {
		c.String(500, "db error")
		return
	}

	c.String(200, "user created")
}
```

这段代码，在 Handler 中实现了所有任务：参数接收，领域逻辑处理，数据访问。

### 仓库模式实现
在仓库模式中，把上面的代码分为不同层来设计：

Handler → Service → Repository(interface) → MySQL Repo → DB

#### 领域模型对象
```go
// 注意：领域模型对象只在各层之间传递，是业务中的数据，不属于任何一个层
type User struct {
	ID   int64
	Name string
}
```
#### MySQL Repo （数据访问）

```go
type userRepo struct {
	db *sql.DB
}

func NewUserRepo(db *sql.DB) UserRepository {
	return &userRepo{db: db}
}

func (r *userRepo) Create(ctx context.Context, user *User) error {
	res, err := r.db.ExecContext(ctx,
		"INSERT INTO users(name) VALUES(?)", user.Name)
	if err != nil {
		return err
	}

	id, err := res.LastInsertId()
	if err == nil {
		user.ID = id
	}

	return nil
}
```
这层只负责执行 SQL 语句，把领域对象写入数据库。

#### Repository (interface)

```go
type UserRepository interface {
	Create(ctx context.Context, user *User) error
}
```
这层是实现仓库模式关键。通过 Go 语言中的 interface 特性，在 Repo 和 Service 之间增加了一层，让 Service 层依赖一个抽象而不是具体的数据访问实现。

#### Service 层（业务逻辑）
```go
type UserService struct {
	repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}

func (s *UserService) CreateUser(ctx context.Context, name string) error {
	if name == "" {
		return errors.New("name required")
	}

	user := &User{Name: name}

	return s.repo.Create(ctx, user)
}
```
这层的边界边界变得更加清晰，职责更加单一。负责领域对象的参数校验和调用数据访问层保存数据。

#### Handler (Gin)

```go
type Handler struct {
	service *UserService
}

func NewHandler(service *UserService) *Handler {
	return &Handler{service: service}
}

func (h *Handler) CreateUser(c *gin.Context) {
	name := c.PostForm("name")

	err := h.service.CreateUser(c.Request.Context(), name)
	if err != nil {
		c.String(400, err.Error())
		return
	}

	c.String(200, "user created")
}
```
Handler 层也显得更清晰和简单了。它只接受来自网络层中参数，然后调用 Service 层处理，最后返回结果。

### 总结
仓库模式是一种有用代码组织方式，本质上是通过在业务领域层和数据访问层之间增加一个抽象层，控制变化。

让原本在业务逻辑和数据访问之间互相渗透的混乱访问方式更加清晰，便于测试。

因此，在项目中，开始发现：

- 测试难以落地
- 业务逻辑和 SQL 混杂
- 一个修改牵一发动全身

那么，也就意味着**是时候引入清晰的边界了**。

<!-- 同时，实现仓库模式需要提前做一些设计，从代码量来看，仓库模式要写更多的代码。因此在什么时候决定使用它也是一个需要考虑的问题。

有一个基本 

为什么你的 Go 服务需要仓库模式：从“代码混乱”到“变化可控”
-->


