<p align="center">
<b><a href="README.md">English</a></b> | 中文
</p>

## Agentic API Spec Design: When API Callers Change from Application to AI Agent

在现在 AI Agent 开发和使用中，我们习惯于给 Agent 喂上下文 - Skill Tool MCP Prompt。

随着 agent tool 的持续进化，MCP 现在大家已经对其没有刚开始出来的时候那么热情了，它规范了API的调用方式，但同样的它让事情变得复杂，需要有支持其协议的客户端，调用的对端还需要对应的服务器协议实现，但如果只是包一堆 API 接口，那为什么不是直接写代码调用那些 API 呢，Skill 就更加合适的代替它了，重要的，它还有整个能让模型更容易理解的使用说明文档。

在 openclaw, 除了它的统一对外消息链接的 gateway 和定时任务机制，它拥有的 skill 的能力与 clawhub 肯定也是让他如此火爆的原因。Skill 的渐进式披露一定程度改善了模型的上下文，但对于复杂系统的skill来说，其过大 skill.md 内容还是会充满模型。同时，它是一个客户端方案，我们需要手工安装维护 skill ，且其版本要跟服务端API对齐，后端接口稍微改个路径或参数，Skill 得全量更新。在 Agent 的世界里，它调用 API 拿到数据后，知道下一步能干什么， 但他是很早之前就知道的， 并且知道了很多不想知道的， 而不是刚刚知道的。

基于上，在想是否能做到更细的渐进式披露，它不会有状态信息差异，不需要客户端安装，让模型能完成任务的前提下，知道的越少越好。于是就设计了下面这个API规范，看能否在服务端侧和Skill形成互补，让agent的规划，调用更顺畅，智能。

以上我们讨论的是那些面向服务API的Skill，本地技能等Skill忽略。


## Agentic API Design

> 这是一个面向 Agent 原生的 API 设计规范。它的核心逻辑是：不要让 Agent 去背全部文档，要让 API 自己会说话。


### 1. 核心响应结构

这里需要所有业务 API 响应包含三个核心字段：

```
{
  "data": {},    // 业务数据
  "error": {},   // 错误控制
  "relates": []  // 关联动作（导航信息）
}

```

* **data 代表"当前状态"**：返回的是当前的业务真实数据，让 Agent 了解当前的业务状态。
* **error 提供"反馈机制"**：通过稳定的错误码（如 `TASK_LOCKED`），让 Agent 能够执行预设的逻辑分支，而不是依赖模糊的文本描述。
* **relates 指示"后续可能性"**：关键设计点。不让 Agent 推测"获得 ID 后应该调用哪个接口"，而是在响应中直接告知可用的"下一步"API操作。

全貌例如下:

```
{
  "data": object,
  "error": {
    code: string,
    message: string
  },
  "relates": [
    {
      "method": "GET|POST|PUT|DELETE",
      "path": "/path/to/resource",
      "desc": "简短说明 API 意图和参数用途",
      "schema": "typechat schema define = { param: string; }"
    }
  ]
}
```

---

### 2. Relates：将“文档”注入“运行时”

传统的 API 是孤岛，而 Agent 需要的是一张图。在 `relates` 结构中定义了当前所调用的API其关联的其它 API 的完整的动作指南：

```
{
  "data": { "id": "task_123", "title": "Fix bug" },
  "relates": [
    {
      "method": "GET",
      "path": "/tasks/{id}",
      "desc": "Retrieve task details",
      "schema": "type GetTask = { id: string };"
    },
    {
      "method": "PUT",
      "path": "/tasks/{id}",
      "desc": "Update task info",
      "schema": "interface UpdateTask { title?: string; priority?: 'low' | 'medium' | 'high'; completed?: boolean; }"
    },
    {
      "method": "DELETE",
      "path": "/tasks/{id}",
      "desc": "Delete task permanently",
      "schema": "type DeleteTask = { id: string };"
    }
  ]
}

```

### 3. API Discovery：图的入口

为了解决 Agent 的"冷启动"问题，还需要有统一入口：`GET /api/llms.txt`。

该接口不返回业务数据，返回当前用户权限下可执行的**顶层操作API**和系统描述。这为 Agent 提供了系统全貌以执行规划（这里我们也需要考虑暴露的粒度）：

```
# Project Name

project description, how to use and soon.

## Entry APIs

### POST /tasks
desc: Create a new task with specified priority  
schema: interface CreateTask { title: string; priority: 'low' | 'medium' | 'high'; description?: string; }


### GET /tasks
desc: List all tasks with pagination  
schema: interface ListTasks { page?: number; limit?: number; status?: 'todo' | 'done'; }

```

---

**对比一下：**

* **传统做法**：Agent 查阅几千字的静态文档，寻找创建任务的方法。
* **这里做法**：Agent 请求任务列表后，响应体直接带上了“创建任务”的接口定义和参数枚举。Agent 只需要根据 `desc`（描述）匹配意图后规划调用。

这种设计本质上是把 **API 的发现逻辑从“静态 Prompt”转到了“动态响应”** 中。

### 4. 对比


| 维度      | 原始 API           | Skill 模式          | Agentic API       |
|---------|------------------|-------------------|--------------------------|
| 感知能力    | 塞入Prompt理解，调用试错  | 静态感知。文档告知所有能力。    | 动态感知。根据当前数据状态实时下发能力。            |
| Token 消耗 | 试错过程中的重复调用       | 随着功能增加线性增长，很容易撑爆。 | 只需一个 `llms.txt` 接口，后续按需下发。     |
| 解耦程度    | 不易用无耦合           | 后端改代码，Prompt 也要改。 | 后端改代码，Agent 自动根据 `relates` 适应。 |

---

### 5. 总结

在 agent 规划调用服务端 API 这个场景下，这套规范确实增加了后端的开发复杂度——需要额外维护 `relates` 的生成逻辑，但看在以后都是 agent 的份上，还是可以接受的。
