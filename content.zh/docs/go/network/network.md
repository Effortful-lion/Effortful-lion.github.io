---
title: "Network"
date: 2026-01-11T13:01:20+08:00
# bookComments: false
# bookSearchExclude: false
# bookPostThumbnail: thumbnail.*
---

## 1. 框架设计目标

设计一个简单、组件化的Actor框架，包含以下核心特性：

- **简洁的Actor模型**：遵循经典Actor模型，支持消息传递、封装状态和行为
- **组件化设计**：将框架拆分为独立、可复用的组件
- **轻量级实现**：最小化依赖，易于理解和扩展
- **高性能**：支持高效的消息处理和并发

## 2. 核心组件设计

### 2.1 Actor接口

```go
// Actor 定义Actor的核心接口
type Actor interface {
    // Receive 处理接收到的消息
    Receive(ctx Context)
}
```

### 2.2 Context接口

```go
// Context 提供Actor执行上下文
type Context interface {
    // Self 返回当前Actor的PID
    Self() *PID
    
    // Sender 返回发送当前消息的Actor的PID
    Sender() *PID
    
    // Message 返回当前处理的消息
    Message() interface{}
    
    // Send 发送消息给指定PID的Actor
    Send(pid *PID, message interface{})
    
    // Spawn 创建新的子Actor
    Spawn(props *Props) *PID
    
    // Stop 停止指定PID的Actor
    Stop(pid *PID)
}
```

### 2.3 PID (Process ID)

```go
// PID 表示Actor的唯一标识符
type PID struct {
    // Address 表示Actor所在的地址
    Address string
    
    // ID 表示Actor的唯一标识
    ID string
}
```

### 2.4 Props

```go
// Props 包含创建Actor的配置信息
type Props struct {
    // Producer 用于创建Actor实例的函数
    Producer func() Actor
    
    // MailboxProducer 用于创建邮箱的函数
    MailboxProducer func() Mailbox
    
    // Dispatcher 用于调度Actor的调度器
    Dispatcher Dispatcher
}
```

### 2.5 Mailbox

```go
// Mailbox 用于存储和处理消息
type Mailbox interface {
    // PostUserMessage 发送用户消息
    PostUserMessage(message interface{})
    
    // PostSystemMessage 发送系统消息
    PostSystemMessage(message interface{})
    
    // RegisterHandlers 注册消息处理器
    RegisterHandlers(invoker MessageInvoker, dispatcher Dispatcher)
    
    // Start 启动邮箱
    Start()
}
```

### 2.6 Dispatcher

```go
// Dispatcher 用于调度Actor的消息处理
type Dispatcher interface {
    // Schedule 调度任务执行
    Schedule(task func())
}
```

### 2.7 ActorSystem

```go
// ActorSystem 是Actor框架的运行时环境
type ActorSystem struct {
    // Root 根上下文，用于创建顶级Actor
    Root *RootContext
    
    // ProcessRegistry 用于注册和查找Actor
    ProcessRegistry *ProcessRegistry
    
    // Config 系统配置
    Config *Config
}
```

## 3. 组件化架构

框架采用组件化设计，各组件之间通过接口进行通信，实现松耦合：

```
+----------------+     +----------------+     +----------------+
|                |     |                |     |                |
|  ActorSystem   |     |   Process      |     |    Mailbox     |
|                |     |   Registry     |     |                |
+-------+--------+     +-------+--------+     +--------+-------+
        |                      |                       |
        v                      v                       v
+-------+--------+     +-------+--------+     +--------+-------+
|                |     |                |     |                |
|    Context     |     |     Actor      |     |   Dispatcher   |
|                |     |                |     |                |
+----------------+     +----------------+     +----------------+
```

## 4. 聊天系统设计

### 4.1 系统架构

聊天系统基于上述Actor框架实现，包含以下核心组件：

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  UserActor       |<--->|  RoomActor       |<--->|  RoomManager     |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
```
