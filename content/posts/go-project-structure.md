+++
title = "go项目组织实践"
date = 2020-04-03T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

### 前言

为Go项目设计一个合适的目录结构是一个令人头痛的事情，也是初学者入坑不可避免的一段弯路，一个好的项目结构不仅会使项目组织看起来更为清晰，解耦功能代码，而且也能带来开发效率的提升，尤其是在多人协作的项目中，可以降低许多时间成本。但是有没有一种完美的项目结构能适应多种不同的场景呢，答案应该是没有的，在不同的场景下选择最为合适的组织方式才是更好的选择，尽管如此，大部分情况下各种项目组织方式仍然有优劣之分，在参考了网上不少关于项目组合的分享以及自己的一些亲身实践，带着思考写下自己的一些见解，欢迎讨论。

### 关于包组织

假如现在开始编写一个Go的微服务，第一步要怎么编写代码呢？估计有不少人会开局一个main.go接着一把梭，这种方法在mini项目或者刚开始的时候比较高效，但随着项目功能的增长，文件内容会变得越来越臃肿，这时候就要开始思考怎么去拆分功能到其他地方了，在[Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)这篇文章中总结了以下几种常见的项目组织方式：

**1.Monolithic package**

简而言之这种方式是把所有的文件通通扔到一个包下面，这样解决了循环依赖的问题，但是当代码行数开始变多时，这种方式维护起来将会非常困难。

```shell
.
├── dao.go
├── repository.go
├── service.go
├── router.go
├── model.go
└── main.go
```

**2.Rails-style layout**

这种布局方式跟Rails的代码组织比较相似，相信许多从Python/Ruby转过来人都写过类似这种的项目结构，这种方式同样存在问题：首先是命名方面，你会得到类似`service.UserService`这种重复包名的糟糕命名，而且这种组织方式也存在双向依赖的风险。

```shell
.
├── repositories
│   └── user.go
├── models
│   └── user.go
├── services
│   └── user.go
├── models
│   └── user.go
└── main.go
```

**3.Group by module**

这种方式使用**模块**作为组织方式，而不是功能，这种做法的问题是也会得到`user.User`这种糟糕命名以及无法消除循环依赖问题。

```shell
.
├── user
│   ├── repository.go
│   ├── dao.go
│   ├── model.go
│   └── service.go
└── main.go
```

在这篇文章中，提出了几个核心的观点：

1. Main包存放领域模型和实现抽象
2. 通过以依赖作为组织方式管理子包
3. 使用一个共享的mock子包
4. Main包将关系依赖联系到一起

最终，你会得到一个像下面那样的目录结构，`User`对象以及抽象接口的定义将会放在根目录下，而具体的实现将放在各自的子包中，在这个例子中，假如接口`UserService`有一个`FindUserByID`方法，这个接口会分别在`mysql`和`redis`包下实现各自的逻辑，而主入口则负责注入`UserService`的实现依赖。

```shell
.
├── mysql
│   └── user.go
├── redis
│   └── user.go
└── user.go
└── main.go
```

这篇文章虽然对前面几种项目风格提出了批判，但并不能说明前面几种方式是anti-pattern，实际中选择最适合项目的组织方式才是妥当的做法，但我觉得有一点是通用的，就是你写出来的实现应该是易于测试的，不然则需要考虑对代码进行重构。

### 关于分层

为了方便对应用功能进行扩展，项目中一般会采取分层把代码按功能维度分成多层，例如最为广泛的项目中一般会采取控制器层（Controller Layer）->逻辑层（Logic Layer）->存储层（Repository/DAO Layer）分层，其中：

- 控制器层：负责接收协议层（http/grpc）的输入，转发/解析/校验请求，有的框架也会把这层扩展为更为具体Transport+Endpoint层。
- 逻辑层：对请求数据处理，执行具体的逻辑业务。
- 存储层：访问底层数据，与领域模型交互返回。

在这个模型下通常还能再简化到只有接口层和存储层，直接通过DataMapper或ORM等方式访问数据（类似PHP/Rails），而具体逻辑则转移为公共组件而不再作为一个层去使用，也有一些应用会使用DDD来解耦代码，这种情况下分层的聚焦点便从功能转移到了领域模型本身以及限界上下文，更偏重于领域设计。

对此我一直觉得分层应该是在满足功能的情况下越简单越好的，所以偏向在一个微服务中不去使用太多分层，只保留最基本的接口层和存储层，将逻辑转移到领域对象本身以及外部包，这样的好处是能更加清晰地了解整个代码逻辑。

### 关于项目结构

关于项目结构其实各人有各自的喜好，我在看过[golang-standards/project-layout](https://github.com/golang-standards/project-layout)以及亲身经历的一些项目，总结了这套比较适合自身的项目结构：

```shell
.
├── api
├── cmd
│   └── main.go
├── gen
│   └── go
├── deployments
├── integration_test
├── scripts
├── config
├── internal
│   ├── mock
│   ├── handler
│   ├── model
│   ├── dao
│   └── router
├── pkg
├── vendor
├── README.md
├── Makefile
├── .gitlab-ci.yml
├── go.mod
└── go.sum
```

其中它们的作用如下：

- `api`：存放项目相关的api/proto定义。
- `cmd`：程序的主入口，也就是main.go所在的地方，如果项目有多个应用的话则可以再次划分多个应用入口目录。
- `gen`：存放proto文件生成的目标代码。
- `deployments`：存放项目部署的模板和配置文件。
- `integration_test`：存放项目的集成测试文件（包括sql，docker-compose依赖等）。
- `scripts`：存放项目build/lint/githooks等脚本，提供给Makefile使用。
- `config`：存放项目配置定义以及注册方法（配置中心，db，redis等）。
- `internal`：存放所有的内部实现（不对外暴露）的代码模块。
- `pkg`：存放可被外部项目引入的组件和模块。
- `vendor`：项目依赖管理是一个令人头痛的问题，对我来说把vendor提交到仓库是一个可以接受的方案。
- `README.md`：项目的说明文件。
- `Makefile`：用来构建和启动Go程序。
- `.gitlab-ci.yml`：gitlab的CI定义文件，在其中定义各种测试任务（lint/build/unit_test）。
- `go.mod`：go module可能是目前最好的依赖管理工具。

Reference:

- [golang-standards/project-layout](https://github.com/golang-standards/project-layout)
- [Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)
- [service-pattern-go](https://github.com/irahardianto/service-pattern-go)
