# LangChain + LangGraph 新版中的 GoF 23 种设计模式

> 本文分析新版 LangChain 及其实际执行内核 LangGraph，不分析
> `langchain-classic`，也不混入旧版 LangGraph。

## 0. 阅读边界

本次阅读基于：

- 仓库提交：`f766ce91bf3445521b4f4727496f405bc44a02a1`
- `libs/langchain_v1`：发布包 `langchain==1.3.2`
- `libs/core`：发布包 `langchain-core==1.4.0`
- `libs/partners/openai`：官方 OpenAI 集成样本
- `libs/partners/anthropic`：官方 Anthropic 集成样本
- LangGraph 仓库标签 `1.2.2`
  - 提交：`add269632bb32c57f3252b7a7006c8115b579fb4`
  - 发布日期：2026-05-26
- `langgraph-prebuilt==1.1.0`
- `langgraph-checkpoint==4.1.0`
  - 对应标签：`checkpoint==4.1.0`
  - 提交：`3614e88c5`

版本选择以 `libs/langchain_v1/uv.lock` 的实际解析结果为准：

```text
langchain             1.3.2
langchain-core        1.4.0
langgraph             1.2.2
langgraph-prebuilt    1.1.0
langgraph-checkpoint  4.1.0
```

注意：LangGraph 仓库当前 `main` 已是 `1.2.5`，本文没有用 `main` 代替
LangChain 锁定的 `1.2.2`。`langgraph==1.2.2` 标签里的 checkpoint 已经升到
`4.1.1`，所以 checkpoint 部分单独按 `checkpoint==4.1.0` 标签阅读。

明确排除：

- `libs/langchain`
- 它发布的是 `langchain-classic==1.0.7`
- 旧式 Chain、旧 Agent、旧 Retrieval Chain 不作为本文证据

这一区分非常重要。新版 `langchain` 已经不是一个塞满各种具体 Chain
实现的大包，而是一个很薄的 Agent 组装层：

```text
langchain_v1
├── agents
│   ├── factory.py
│   ├── middleware/
│   └── structured_output.py
├── chat_models/
├── embeddings/
├── messages/
├── tools/
└── rate_limiters.py
```

新版 LangChain 依赖两个更基础的部分：

```text
langchain-core：统一组件契约、Runnable、消息、模型、工具、回调
LangGraph：图执行、状态流转、持久化、暂停与恢复
```

因此，完整阅读新版 LangChain 时，应当把它理解成：

```text
LangChain 新版
= langchain-core 的组件协议
+ langchain_v1 的 Agent 组装
+ LangGraph 1.2.2 的图构建与 Pregel 运行时
+ checkpoint 4.1.0 的状态快照协议
+ prebuilt 1.1.0 的 ToolNode
+ 各 provider integration 的协议适配
```

## 1. 先看真实执行主线

如果先背设计模式名称，很容易把类名和模式强行配对。应先抓住一条最小
执行链：

```python
agent = create_agent(
    model="openai:gpt-...",
    tools=[search],
    middleware=[retry, fallback],
    response_format=Answer,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "问题"}]}
)
```

源码中的实际过程是：

```text
create_agent
  |
  +-- init_chat_model
  |     `-- 根据 provider 注册表创建或延迟创建 BaseChatModel
  |
  +-- 普通函数/字典/类型 -> BaseTool
  |
  +-- response_format -> ToolStrategy / ProviderStrategy / AutoStrategy
  |
  +-- AgentMiddleware
  |     +-- before_agent / before_model 编成图节点
  |     +-- after_model / after_agent 编成图节点
  |     +-- wrap_model_call 编成调用包装链
  |     `-- wrap_tool_call 编成调用包装链
  |
  +-- StateGraph
  |     +-- model node
  |     +-- tools node
  |     +-- middleware nodes
  |     `-- conditional edges
  |
  `-- compile() -> CompiledStateGraph
          |
          `-- Pregel.invoke / stream / ainvoke
                |
                +-- SyncPregelLoop / AsyncPregelLoop
                +-- PregelRunner
                +-- channels + reducers
                +-- checkpoint + pending writes
                `-- interrupt / resume / replay
```

从 Agent 视角看，一次典型循环为：

```text
输入消息
  -> before_agent
  -> before_model
  -> model
  -> after_model
  -> 模型是否产生 tool_calls？
       |
       +-- 否 -> after_agent -> END
       |
       `-- 是 -> tools
                  -> ToolMessage / Command
                  -> before_model
                  -> model
                  -> ...
```

但 LangGraph 不是沿着边立即递归调用下一个节点。真正执行采用
Bulk Synchronous Parallel/Pregel 超步：

```text
step N 开始
  -> 从 checkpoint/channel versions 计算本步任务
  -> 所有任务读取同一个不可变 channel 快照
  -> PregelRunner 并发运行任务
  -> 每个任务只产生 pending writes
  -> 屏障：apply_writes
  -> channel.update(values) 执行 reducer
  -> 创建 checkpoint
  -> step N+1 才能看到这些更新
```

源码中的核心调用链：

```text
CompiledStateGraph.stream
  -> SyncPregelLoop.__enter__
  -> PregelLoop.tick
       -> prepare_next_tasks
  -> PregelRunner.tick
       -> run_with_retry
       -> task.proc.invoke
       -> commit task writes
  -> PregelLoop.after_tick
       -> apply_writes
       -> channel.update
       -> create_checkpoint / put
  -> 下一超步或结束
```

这条完整主线决定了新版最重要的模式不是某个 provider 的实现细节，而是：

1. `Runnable` 的统一组件协议
2. `RunnableSequence`、`RunnableParallel` 的组合
3. middleware 和 binding 的包装
4. model、tool、structured output 的策略替换
5. `BaseChatModel`、`BaseTool` 的模板方法
6. callback 的事件观察
7. `_ConfigurableModel` 的延迟代理
8. `StateGraph` 的 Builder
9. `Command/Send` 的控制命令
10. checkpoint 的 Memento
11. Channel 的 reducer Strategy
12. PregelLoop/PregelRunner 的集中调度

## 2. 判定标准

本文把 23 种模式分成四档：

| 等级 | 含义 |
|---|---|
| 明确使用 | 角色、协作关系和运行行为都与 GoF 模式高度一致 |
| 变体使用 | 解决的是同类问题，但使用 Python 函数、类型或外部运行时实现 |
| 仅相似 | 局部形状相似，不足以认定为该模式 |
| 未发现 | 新版主干中没有可靠证据，不强行套用 |

总览：

| 类别 | 模式 | 判定 | 新版中的主要位置 |
|---|---|---|---|
| 创建型 | Abstract Factory | 未发现 | provider 注册表不是产品族工厂 |
| 创建型 | Builder | 明确使用 | `StateGraph`、`NodeBuilder`、LCEL |
| 创建型 | Factory Method | 仅相似 | `init_chat_model` 是简单/注册表工厂 |
| 创建型 | Prototype | 变体使用 | Channel `copy/from_checkpoint`、request override |
| 创建型 | Singleton | 未发现 | 注册表和缓存不等于单例 |
| 结构型 | Adapter | 明确使用 | provider、tool、Runnable 转换 |
| 结构型 | Bridge | 未发现 | provider 主要靠继承和适配 |
| 结构型 | Composite | 明确使用 | `RunnableSequence/Parallel` |
| 结构型 | Decorator | 明确使用 | `RunnableBinding`、middleware wrapper |
| 结构型 | Facade | 明确使用 | `create_agent`、`init_chat_model` |
| 结构型 | Flyweight | 未发现 | creator 缓存不是享元对象池 |
| 结构型 | Proxy | 明确使用 | `_ConfigurableModel`、`RemoteGraph` |
| 行为型 | Chain of Responsibility | 变体使用 | fallbacks、middleware 调用链 |
| 行为型 | Command | 明确使用 | `Command`、`Send`、`ToolCall` |
| 行为型 | Interpreter | 仅相似 | prompt/schema 的解释转换 |
| 行为型 | Iterator | 明确使用 | `stream/astream`、消息块 |
| 行为型 | Mediator | 变体使用 | PregelLoop/PregelRunner/channel |
| 行为型 | Memento | 明确使用 | Checkpoint、Saver、StateSnapshot |
| 行为型 | Observer | 明确使用 | callback manager/handler |
| 行为型 | State | 仅相似 | Agent 是状态图，不是经典 State 对象 |
| 行为型 | Strategy | 明确使用 | structured output、provider、policy |
| 行为型 | Template Method | 明确使用 | model、tool、prompt、parser |
| 行为型 | Visitor | 未发现 | 没有双分派访问体系 |

---

# 一、创建型模式

## 3. Abstract Factory：未发现

### 经典意图

抽象工厂创建一组相互关联的产品，客户端只依赖抽象工厂，不依赖具体产品。

例如：

```text
GUIFactory
├── create_button()
└── create_menu()

WindowsFactory -> WindowsButton + WindowsMenu
MacFactory     -> MacButton + MacMenu
```

### LangChain 为什么不是

`init_chat_model()` 确实可以按 provider 创建模型：

```text
"openai"    -> ChatOpenAI
"anthropic" -> ChatAnthropic
```

但它只创建一个模型对象，并没有：

```text
OpenAIFactory
├── create_chat_model()
├── create_embeddings()
└── create_tool_runtime()
```

`init_embeddings()` 也有自己的注册表，二者没有组成一个“产品族工厂”。

### 容易误判的地方

“能创建不同实现”不等于 Abstract Factory。这里只有注册表驱动的简单工厂，
缺少“一个具体工厂生产一整组兼容产品”这一关键结构。

**结论：新版主干未发现 Abstract Factory。**

## 4. Builder：明确使用

### 经典意图

Builder 把复杂对象的构造过程与最终表示分开，使同一构造过程能产生不同产品。

### 证据一：LCEL 逐步构造 Runnable 图

`Runnable` 重载 `|`，把左、右两侧逐步组合成 `RunnableSequence`：

```python
chain = prompt | model | parser
```

最终产品不是立即执行的结果，而是一个可执行对象：

```text
RunnableSequence(
    first=prompt,
    middle=[model],
    last=parser,
)
```

字典还会被转换为 `RunnableParallel`：

```python
chain = {
    "answer": answer_chain,
    "sources": source_chain,
} | formatter
```

这里的“构造语言”是运算符和字典，而不是传统的 `Director + Builder` 类。

源码：

- `libs/core/langchain_core/runnables/base.py`
- `RunnableSequence`：约 2995 行
- `RunnableParallel`：约 3743 行
- `coerce_to_runnable`：约 6489 行

### 证据二：`StateGraph` 是显式 Builder

LangGraph 源码直接声明：

```text
StateGraph is a builder class and cannot be used directly for execution.
```

它维护尚未编译的：

- nodes
- edges
- branches
- state/input/output schema
- channels 与 managed values
- retry/cache/error handler/timeout 默认策略

用户逐步调用：

```python
builder = StateGraph(State)
builder.add_node("model", model_node)
builder.add_node("tools", tool_node)
builder.add_edge(START, "model")
builder.add_conditional_edges("model", route)
graph = builder.compile()
```

`compile()` 才产生最终产品 `CompiledStateGraph`。编译过程会：

1. 校验节点和边
2. 把 schema 字段解析为 `BaseChannel`
3. 把节点包装成 `PregelNode`
4. 把边编译成 trigger/channel write
5. 把等待边编译成 barrier channel
6. 把条件边编译成 `BranchSpec`
7. 注入 checkpointer、store、cache 和 interrupt 配置

源码：

- `/home/zym/langgraph/libs/langgraph/langgraph/graph/state.py`
- `StateGraph`：约 130 行
- `compile`：约 1164 行
- `CompiledStateGraph`：约 1391 行
- `NodeBuilder`：`pregel/main.py` 约 204 行

这已经具备 Builder 的完整角色：

| Builder 角色 | LangGraph |
|---|---|
| Builder | `StateGraph` |
| 构造步骤 | `add_node/add_edge/add_conditional_edges` |
| Product | `CompiledStateGraph` |
| 构造完成 | `compile()` |

### 证据三：`create_agent()` 作为高层 Director/Facade

`create_agent()` 依次完成：

1. 初始化模型
2. 转换工具
3. 解析结构化输出策略
4. 收集中间件 hook
5. 构造 `StateGraph`
6. 添加 model、tools、中间件节点
7. 添加条件边
8. `compile()` 返回 `CompiledStateGraph`

源码：

- `libs/langchain_v1/langchain/agents/factory.py:696`
- 创建图约在 1049 行
- 添加节点约在 1385 行以后
- 编译约在 1670 行

### 判定

LCEL 本身是函数式 Builder 变体；LangGraph 的 `StateGraph -> compile()` 则是
非常明确的 Builder。`create_agent()` 在更高一层决定如何使用这个 Builder。

**结论：完整读取 LangGraph 后，Builder 应从“变体”提升为“明确使用”。**

## 5. Factory Method：仅相似

### 经典意图

父类定义创建产品的方法，子类重写该方法决定创建哪一种具体产品。

### 新版中的工厂函数

`init_chat_model()` 使用 `_BUILTIN_PROVIDERS` 注册表：

```text
provider
  -> package
  -> class name
  -> creator function
  -> concrete BaseChatModel
```

源码：

- `libs/langchain_v1/langchain/chat_models/base.py:38`
- `init_chat_model`：约 175 行
- `_init_chat_model_helper`：约 510 行

类似入口还有：

- `init_embeddings()`
- `tool()` 装饰器/函数
- `create_agent()`

### 为什么不是经典 Factory Method

它们没有“Creator 父类 + 子类覆盖 factory_method()”结构，而是普通函数根据参数、
注册表和惰性 import 选择类。

更准确的名字是：

```text
Simple Factory
+ Registry Factory
+ Lazy Import
```

**结论：有大量工厂函数，但没有可靠的经典 Factory Method。**

## 6. Prototype：变体使用

### 证据

`ModelRequest` 是 dataclass，middleware 可以通过：

```python
new_request = request.override(
    model=fallback_model,
    system_prompt="新的提示",
)
```

创建一个修改部分字段的新请求。`override()` 内部使用
`dataclasses.replace(self, **overrides)`。

源码：

- `libs/langchain_v1/langchain/agents/middleware/types.py:90`
- `ModelRequest.override`：约 205 行

这让 middleware 可以保持原请求不变：

```text
原 request
  -> copy-with model=A
  -> 失败
  -> copy-with model=B
```

### LangGraph Channel 的复制协议

`BaseChannel` 定义：

```python
def copy(self) -> Self:
    return self.from_checkpoint(self.checkpoint())

def from_checkpoint(self, checkpoint) -> Self:
    ...
```

`LastValue`、`Topic`、`BinaryOperatorAggregate`、`NamedBarrierValue`、
`EphemeralValue` 等都创建同类型的新 channel，并复制当前内部状态。

LangGraph 每次运行和恢复不能直接共享 Builder 中的可变 channel 原型，而要从：

- 当前 channel
- checkpoint blob

构造独立的运行时副本。

源码：

- `/home/zym/langgraph/libs/langgraph/langgraph/channels/base.py`
- `/home/zym/langgraph/libs/langgraph/langgraph/channels/*.py`

### 为什么仍称为变体

Channel 的复制首先服务于运行隔离和 checkpoint 恢复，Prototype 与 Memento 在这里
重叠；它并没有提供一个面向用户的通用 `clone()` 产品创建体系。

**结论：`ModelRequest.override()` 只是相似；Channel 的 `copy/from_checkpoint`
构成可靠的 Prototype 变体。**

## 7. Singleton：未发现

`_BUILTIN_PROVIDERS` 是模块常量，`_get_chat_model_creator()` 使用
`lru_cache` 缓存 creator，但它们都不是：

```text
一个类
+ 一个全局实例
+ 受控构造入口
```

每次初始化仍可产生不同模型实例。callback manager、agent、tool 也没有被限制为单例。

**结论：新版主干未使用 Singleton；缓存和模块全局变量不能直接算单例。**

---

# 二、结构型模式

## 8. Adapter：明确使用

Adapter 是 LangChain 最核心的模式之一。

### 场景一：provider SDK 适配为 `BaseChatModel`

统一目标接口：

```python
model.invoke(messages)
model.stream(messages)
model.bind_tools(tools)
model.with_structured_output(schema)
```

OpenAI 的具体适配器：

```text
OpenAI request/response/stream
  <-> ChatOpenAI
  <-> BaseChatModel / AIMessage / ChatGenerationChunk
```

Anthropic 的具体适配器：

```text
Anthropic messages/content blocks
  <-> ChatAnthropic
  <-> BaseChatModel / AIMessage / ChatGenerationChunk
```

源码：

- `libs/partners/openai/langchain_openai/chat_models/base.py`
  - `ChatOpenAI`：约 2534 行
  - `_stream`：约 1549 行
  - `_generate`：约 1624 行
  - `bind_tools`：约 2131 行
- `libs/partners/anthropic/langchain_anthropic/chat_models.py`
  - `ChatAnthropic`：约 877 行
  - `_stream`：约 1432 行
  - `_generate`：约 1751 行
  - `bind_tools`：约 1808 行

两个 provider 的请求字段、tool schema、流式事件完全不同，但上层 Agent 只面对
`BaseChatModel`。

### 场景二：普通 Python 对象适配为 Tool

用户可以传：

```python
def search(query: str) -> str:
    ...
```

`tool()`、`StructuredTool.from_function()` 和 schema 推断会把它转换成 `BaseTool`。
Agent 之后只调用统一的：

```python
tool.invoke(tool_call)
```

### 场景三：不同对象适配为 Runnable

`coerce_to_runnable()` 的转换规则：

```text
Runnable  -> 原样返回
callable  -> RunnableLambda
generator -> RunnableGenerator
dict      -> RunnableParallel
```

这使 LCEL 可以接受普通函数和字典。

### 设计价值

Adapter 解决的是“外部世界接口不同”，不是“运行时选择哪个算法”。provider
类同时常被当成 Strategy 使用，但它首先必须完成协议适配。

**结论：Adapter 是新版 LangChain 跨 provider、跨对象类型统一调用的基础。**

## 9. Bridge：未发现

Bridge 要求把两个可独立变化的维度拆成组合关系：

```text
Abstraction ----has-a---- Implementor
```

容易把 `BaseChatModel + ChatOpenAI` 看作 Bridge，但它们主要是继承关系：

```text
ChatOpenAI is-a BaseChatModel
```

并没有一个独立的 `ChatModelAbstraction` 持有可替换的 `ProviderImplementor`。

`RunnableBinding` 虽然持有另一个 Runnable，但它的目的属于 Decorator，不是把
抽象维度和实现维度分离。

**结论：没有足够证据认定 Bridge。**

## 10. Composite：明确使用

### 角色对应

| Composite 角色 | LangChain |
|---|---|
| Component | `Runnable` |
| Leaf | model、prompt、parser、tool、`RunnableLambda` |
| Composite | `RunnableSequence`、`RunnableParallel` |
| Operation | `invoke/ainvoke/batch/stream` |

### 顺序组合

```python
chain = prompt | model | parser
result = chain.invoke(input)
```

`RunnableSequence.invoke()` 依次调用内部步骤，并为每一步派生 callback config。

```text
input
  -> prompt.invoke
  -> model.invoke
  -> parser.invoke
  -> output
```

### 并行组合

```python
parallel = RunnableParallel(
    answer=answer_chain,
    sources=source_chain,
)
```

它自己仍然是 Runnable，因此可以继续组合：

```python
full = parallel | formatter
```

树中的叶子和组合对象拥有同一接口，这正是 Composite 的核心。

源码：

- `libs/core/langchain_core/runnables/base.py:2995`
- `libs/core/langchain_core/runnables/base.py:3743`

### 设计价值

用户不必判断当前对象是模型、函数、顺序链还是并行图：

```python
anything_runnable.invoke(input)
```

**结论：Runnable 体系是标准且非常清晰的 Composite。**

## 11. Decorator：明确使用

新版有两套非常明确的 Decorator。

### 第一套：`RunnableBinding`

`RunnableBindingBase` 持有：

```python
bound: Runnable
kwargs: Mapping[str, Any]
config: RunnableConfig
```

它对外仍然暴露 Runnable 接口，把调用委托给 `bound`，同时叠加参数、配置、
listener、类型信息等。

```python
configured = model.bind(temperature=0)
retried = configured.with_retry(...)
fallback = retried.with_fallbacks([...])
```

每一层仍然是 Runnable：

```text
RunnableWithFallbacks
  -> RunnableRetry
       -> RunnableBinding
            -> ChatModel
```

源码甚至直接把 `RunnableBinding` 描述为 runnable decorator：

- `libs/core/langchain_core/runnables/base.py:5716`
- `libs/core/langchain_core/runnables/base.py:6245`

### 第二套：Agent middleware 包装链

`wrap_model_call(request, handler)` 可以：

- 修改 request 后调用 handler
- 调用 handler 多次以重试
- 完全不调用 handler，直接短路
- 修改 response
- 捕获异常后切换模型

例如：

```python
class RetryMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        for attempt in range(3):
            try:
                return handler(request)
            except TemporaryError:
                if attempt == 2:
                    raise
```

多个 wrapper 按注册顺序组成嵌套：

```text
auth(
  cache(
    retry(
      actual_model_call
    )
  )
)
```

进入顺序从外向内，返回顺序从内向外。

源码：

- model wrapper 组链：`agents/factory.py:220`
- tool wrapper 组链：`agents/factory.py:584`
- middleware 契约：`agents/middleware/types.py:383`

### 不要混淆

`before_model`、`after_model` 被编译成图节点，更接近 Pipeline hook；
`wrap_model_call`、`wrap_tool_call` 才是结构上严格的 Decorator。

**结论：Decorator 是新版扩展行为而不修改核心对象的主要手段。**

## 12. Facade：明确使用

### `create_agent()` 是 Agent 子系统门面

用户只传：

```python
create_agent(model, tools, middleware, response_format)
```

门面后面隐藏了：

- provider 模型初始化
- tool schema 转换
- structured output 策略选择
- middleware 顺序处理
- state schema 合并
- `ToolNode` 创建
- `StateGraph` 节点和边创建
- checkpointer/store/cache/interrupt 参数
- 图编译

用户不需要亲自操作这些对象。

### `init_chat_model()` 是 provider 子系统门面

它隐藏：

- provider 名称推断
- 可选依赖导入
- provider 类选择
- 固定模型和可配置模型差异
- 延迟创建

### Facade 与 Factory 的区别

`init_chat_model()` 有创建功能，但它的门面价值是统一多个 provider 的复杂初始化。
`create_agent()` 更明显：它不是简单 new 一个对象，而是协调多个子系统。

**结论：`create_agent()` 是新版最重要的功能型 Facade。**

## 13. Flyweight：未发现

`_get_chat_model_creator()` 会缓存 provider creator，schema 也可能被缓存，但没有：

- 大量细粒度共享对象
- 内部状态/外部状态分离
- 通过工厂复用相同对象实例

缓存“如何创建模型的函数”与共享“模型对象本身”不是一回事。

**结论：新版主干未发现 Flyweight。**

## 14. Proxy：明确使用

### `_ConfigurableModel` 是虚拟代理

`init_chat_model()` 在允许运行时配置时，不一定立即创建 provider 模型，而是返回
`_ConfigurableModel`：

```text
调用前：
_ConfigurableModel(
    default_config=...,
    queued_declarative_operations=...
)

调用时：
config
  -> 计算 model/provider 参数
  -> 创建具体 ChatOpenAI/ChatAnthropic
  -> 重放 bind_tools/with_structured_output
  -> 委托 invoke/stream/batch
```

源码：

- `_ConfigurableModel`：`chat_models/base.py:635`
- `__getattr__`：约 659 行
- `_model`：约 689 行

### 代理行为

用户可以在具体模型尚未创建时写：

```python
model = init_chat_model(configurable_fields="any")
model = model.bind_tools(tools)
```

`__getattr__` 捕获允许延迟执行的声明式操作，把它们排队；真正调用时才创建目标对象。

### 为什么是 Proxy

它和真实模型拥有相同 Runnable 使用方式，并控制对真实模型的访问时机。它不是单纯
Adapter，因为接口没有被转换；也不只是 Decorator，因为关键目的是延迟解析和创建
真实对象。

**结论：这是典型 Virtual Proxy，并附带运行时选择能力。**

### `RemoteGraph` 是远程代理

LangGraph 的 `RemoteGraph` 实现 `PregelProtocol`，对外提供与本地图相近的：

```text
invoke / ainvoke
stream / astream
get_state / get_state_history
update_state
get_graph
```

内部则把操作转换成 LangGraph Server API 请求，再把远程：

- stream chunk
- checkpoint
- interrupt
- `Command.PARENT`
- error

转换回本地 `StateSnapshot`、`Interrupt`、`GraphInterrupt` 等协议对象。

源码：

- `/home/zym/langgraph/libs/langgraph/langgraph/pregel/remote.py`
- `RemoteGraph`：约 112 行

这符合 Remote Proxy：

```text
客户端像调用本地图一样调用 RemoteGraph
  -> RemoteGraph 控制网络访问
  -> 远端图真正执行
```

**补充结论：新版同时存在 Virtual Proxy 与 Remote Proxy。**

---

# 三、行为型模式

## 15. Chain of Responsibility：变体使用

### 场景一：`RunnableWithFallbacks`

它按顺序保存：

```text
primary
fallback_1
fallback_2
...
```

执行逻辑：

```python
for runnable in self.runnables:
    try:
        return runnable.invoke(input)
    except exceptions_to_handle:
        continue
raise first_error
```

源码：

- `libs/core/langchain_core/runnables/fallbacks.py:36`
- `invoke`：约 166 行
- `ainvoke`：约 216 行

请求沿候选处理器传播，直到一个成功处理，符合 CoR 的问题结构。

### 场景二：`ModelFallbackMiddleware`

middleware 先调用默认模型，失败后用：

```python
request.override(model=fallback_model)
```

把同一请求交给下一个候选模型。

### 为什么判为变体

经典 CoR 常让每个 handler 持有 `next` 并决定是否转发。这里是一个协调对象遍历
handler 列表，更接近集中式 fallback chain。

另外，middleware wrapper 的整体结构首先是 Decorator；只有“某层决定调用或跳过
下一个 handler”这一行为具有 CoR 特征。

**结论：fallback 是可靠的 CoR 变体，不应把所有 middleware 都统称责任链。**

## 16. Command：明确使用

### ToolCall 作为请求对象

模型不直接执行 Python 函数，而是产生结构化 `ToolCall`：

```python
{
    "name": "search",
    "args": {"query": "LangChain"},
    "id": "call_123",
}
```

`ToolNode` 根据名字找到 receiver，随后调用对应 `BaseTool`。

角色近似：

| Command 角色 | 新版对象 |
|---|---|
| Command 数据 | `ToolCall` |
| Invoker | Agent graph / `ToolNode` |
| Receiver | `BaseTool` 实现 |
| Result | `ToolMessage` |

### LangGraph `Command` 是一等控制对象

`Command` 是冻结 dataclass，可以同时表达：

- `update`：要写入 state/channel 的更新
- `goto`：节点名、多个节点或 `Send`
- `resume`：恢复某个 interrupt
- `graph`：作用于当前图或 `Command.PARENT`

节点、工具和 middleware 都可以返回它。`CompiledStateGraph.attach_node()` 把
`Command.update` 转成 channel writes，把 `goto` 转成控制分支；PregelLoop 又会把
输入的 `Command(resume=...)` 转成 `RESUME` writes。

```python
return Command(
    update={"messages": [tool_message]},
    goto="model",
)
```

### `Send` 是参数化命令/消息

`Send(node, arg)` 可以在运行时动态创建多个目标任务：

```python
return [
    Send("generate_joke", {"subject": subject})
    for subject in state["subjects"]
]
```

这使同一个节点以不同输入并行执行，是 map-reduce 和多 Agent handoff 的基础。

### ToolNode 是 Invoker

`ToolNode`：

1. 解析 `ToolCall`
2. 按名称查找 `BaseTool`
3. 注入 state/store/runtime
4. 并行执行 receiver
5. 规范化为 `ToolMessage` 或 `Command`
6. 校验跨图命令和 tool_call_id

源码：

- `/home/zym/langgraph/libs/langgraph/langgraph/types.py`
- `/home/zym/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py`
- `/home/zym/langgraph/libs/langgraph/langgraph/graph/state.py`

虽然 `Command` 本身是数据对象而不是 `execute()` 多态类，但框架已经明确区分
Command、Invoker、Receiver 和执行结果。

**结论：结合 LangGraph 后，Command 应判为明确使用，而不是只算近似。**

## 17. Interpreter：仅相似

### 相似点

Prompt 模板会解释：

- f-string 模板
- mustache 模板
- jinja2 模板

Structured output 也会把：

- Pydantic model
- dataclass
- TypedDict
- JSON Schema

解释为 provider schema、tool schema 和响应解析器。

`_SchemaSpec` 正是在统一这些 schema 表达。

### 缺少的结构

经典 Interpreter 通常有：

```text
AbstractExpression
├── TerminalExpression
└── NonTerminalExpression

expression.interpret(context)
```

LangChain 没有用一棵自有表达式对象树解释完整语言。它主要调用 Python 格式化、
模板库、Pydantic 和 JSON Schema 工具。

**结论：有 DSL 解释行为，但不足以认定经典 Interpreter。**

## 18. Iterator：明确使用

### 同步流

```python
for chunk in model.stream(messages):
    consume(chunk)
```

### 异步流

```python
async for chunk in model.astream(messages):
    consume(chunk)
```

调用方只依赖迭代协议，不需要知道 provider 是：

- HTTP chunked stream
- SSE event
- SDK context manager
- 非流式结果模拟出的单块流

OpenAI `_stream()` 把 SDK chunk 逐个转换成 `ChatGenerationChunk` 后 `yield`；
Anthropic 也把自己的事件转换为统一 chunk。

Runnable 的 `stream/astream/transform/atransform` 进一步让流能够穿过组合链。

LangGraph 在此之上提供多种 step stream：

```text
values
updates
messages
custom
checkpoints
tasks
debug
```

`Pregel.stream()` 和 `Pregel.astream()` 驱动执行循环，同时把 task 完成、channel
更新、checkpoint 和 token 事件逐步暴露。`RemoteGraph.stream()` 又把远程 SSE
事件适配为同一迭代协议。

### 设计价值

Iterator 把“如何取得下一个 provider 事件”封装起来，调用方只处理统一消息块。

**结论：Python 原生同步/异步 Iterator 是 LangChain 流式执行的核心协议。**

## 19. Mediator：变体使用

### 节点不直接互相调用

编译后的节点主要面对：

- 输入 state
- `Runtime`
- channel read/write
- `Command/Send`

节点 A 不需要持有节点 B，也不负责调度 B。

### Pregel runtime 集中协调

```text
PregelLoop
  -> 根据 channel versions 决定哪些节点可运行
  -> 处理 input、resume、interrupt、checkpoint

PregelRunner
  -> 并发提交 task
  -> retry/timeout/error handler
  -> commit pending writes

apply_writes
  -> 在超步屏障统一合并结果
  -> 更新 channel version
```

这个运行时把节点之间原本可能形成的直接依赖集中到 channel 和调度器。

### 为什么判为变体

Pregel 首先是数据流/BSP 执行引擎，而不是为简化一组 GUI colleague 关系而设计的
经典 Mediator。`PregelLoop`、`PregelRunner`、channel 还共同分担了协调职责，不是
单一 Mediator 对象。

**结论：完整系统存在 Mediator 的结构效果，但更准确地称 Pregel 调度器变体。**

## 20. Memento：明确使用

### 角色对应

| Memento 角色 | LangGraph |
|---|---|
| Originator | Pregel graph/channel state |
| Memento | `Checkpoint` |
| Caretaker | `BaseCheckpointSaver` 实现 |
| 可读视图 | `StateSnapshot` |

`Checkpoint` 保存：

- `channel_values`
- `channel_versions`
- 每个节点的 `versions_seen`
- `updated_channels`
- checkpoint id 和时间

`CheckpointTuple` 再关联：

- metadata
- parent checkpoint config
- pending writes

`BaseCheckpointSaver` 定义：

```text
get_tuple / list
put / put_writes
aget_tuple / alist
aput / aput_writes
```

内存、SQLite、Postgres 等 saver 可以替换。

### 恢复不是简单反序列化

恢复时 LangGraph 会：

1. 从 checkpoint 重建各个 channel
2. 按 channel version 与 `versions_seen` 计算下一批 task
3. 重新附加已经成功任务的 pending writes
4. 跳过不必重复执行的成功任务
5. 对 interrupt 写入 resume value
6. 从节点开头重放被中断任务

### Time travel 与 fork

指定历史 checkpoint 执行时，运行时会建立新的 fork checkpoint。后续写入形成新
分支，而不是篡改原历史。

### DeltaChannel

`DeltaChannel` 并非每步保存完整值，而是定期保存 `_DeltaSnapshot`，其余状态通过
祖先 pending writes 重放。这仍属于 Memento，只是使用“快照 + 增量日志”优化。

源码：

- `/home/zym/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py`
- `/home/zym/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py`
- `/home/zym/langgraph/libs/langgraph/langgraph/pregel/_loop.py`
- `/home/zym/langgraph/libs/langgraph/langgraph/pregel/_checkpoint.py`
- `/home/zym/langgraph/libs/langgraph/langgraph/channels/delta.py`

**结论：在 LangChain + LangGraph 整体中，Memento 是最明确、最重要的模式之一。**

## 21. Observer：明确使用

### 角色对应

| Observer 角色 | LangChain |
|---|---|
| Subject/Dispatcher | callback manager |
| Observer | `BaseCallbackHandler` |
| Event | start、token、end、error、retry 等 |

`CallbackManager` 在运行过程中广播：

```text
on_chain_start
on_chat_model_start
on_llm_new_token
on_tool_start
on_tool_end
on_chain_error
...
```

`handle_event()` 遍历 handlers，调用对应事件方法；`ahandle_event()` 负责异步路径。

源码：

- `libs/core/langchain_core/callbacks/base.py:496`
- `libs/core/langchain_core/callbacks/manager.py:255`
- `libs/core/langchain_core/callbacks/manager.py:419`
- `CallbackManager`：约 1343 行

### 运行例子

当 OpenAI stream 产生一个 chunk：

```text
OpenAI SDK event
  -> 转换 ChatGenerationChunk
  -> run_manager.on_llm_new_token(...)
  -> callback manager
  -> 每个 handler.on_llm_new_token(...)
```

模型不需要知道日志、追踪、UI 或监控系统的具体类型。

### 与 middleware 的区别

- Observer 通常旁观事件，不决定核心调用是否继续
- middleware 可以修改请求、短路、重试和替换响应

**结论：callback 子系统是标准 Observer。**

## 22. State：仅相似

新版 Agent 明确拥有 `AgentState`：

```text
messages
jump_to
structured_response
自定义 state_schema 字段
```

图也会根据当前状态决定：

- 去 tools
- 回 model
- 结束
- 跳到 middleware 节点

### 为什么不是经典 State

经典 State 模式让 Context 持有一个 State 对象：

```text
Context.state.handle()
  -> ConcreteStateA / ConcreteStateB
```

新版 Agent 的状态主要是 TypedDict 数据；行为位于图节点和路由函数中，没有
`ThinkingState`、`ToolCallingState`、`FinishedState` 这样的多态状态对象。

它更准确地说是：

```text
State Machine / State Graph
```

而不是 GoF State 对象模式。

**结论：状态流转非常重要，但不能因此直接判定使用了经典 State。**

## 23. Strategy：明确使用

Strategy 是新版另一条主轴。

### 场景一：结构化输出策略

`response_format` 可表示：

```text
ToolStrategy
ProviderStrategy
AutoStrategy
```

职责分别是：

- `ToolStrategy`：把目标 schema 暴露成工具，让模型通过 tool call 返回
- `ProviderStrategy`：使用 provider 原生 structured output
- `AutoStrategy`：根据模型能力自动选择前两者

源码：

- `structured_output.py:195`
- `structured_output.py:261`
- `structured_output.py:447`

上层目标不变：

```text
获得符合 schema 的 structured_response
```

实现算法可以替换，这就是 Strategy。

### 场景二：模型实现策略

Agent 只依赖 `BaseChatModel`：

```python
create_agent(model=chat_openai, ...)
create_agent(model=chat_anthropic, ...)
```

在“完成一次标准聊天模型调用”这一上下文中，provider 模型是可互换策略。同时它们
也承担 Adapter 职责：

```text
Adapter：把 provider API 翻译成统一协议
Strategy：上层运行时可替换使用哪个实现
```

### 场景三：执行策略

middleware 的 `_execution.py` 定义：

```text
BaseExecutionPolicy
├── HostExecutionPolicy
├── CodexSandboxExecutionPolicy
└── DockerExecutionPolicy
```

同一执行请求可以选择宿主机、Codex sandbox 或 Docker。

### 场景四：上下文编辑策略

`ContextEdit` Protocol 定义 `apply()`，`ContextEditingMiddleware` 按配置执行具体
编辑策略，例如 `ClearToolUsesEdit`。

源码：

- `agents/middleware/_execution.py:57`
- `agents/middleware/context_editing.py:44`

### 场景五：Channel/reducer 策略

State schema 的每个字段会被解析成不同 channel：

```text
LastValue               单写者最后值
BinaryOperatorAggregate 用 reducer 合并并发写入
Topic                   发布订阅集合
EphemeralValue          只保留相邻超步
NamedBarrierValue       等齐多个上游
DeltaChannel            快照加增量重放
```

Pregel 的 `apply_writes()` 不需要知道具体合并算法，只调用：

```python
channel.update(values)
```

### 场景六：Checkpoint Saver 策略

运行时依赖 `BaseCheckpointSaver`，具体可以使用内存、SQLite、Postgres 或自定义
saver。`durability` 还可以选择 `sync/async/exit` 持久化策略。

**结论：凡是“目标稳定、算法或 provider 可换”的位置，新版普遍使用 Strategy。**

## 24. Template Method：明确使用

### `BaseChatModel`

子类最少实现：

```python
def _generate(...) -> ChatResult:
    ...

@property
def _llm_type(self) -> str:
    ...
```

可选实现 `_stream()`、`_agenerate()`、`_astream()`。

公共 `invoke()`/`stream()` 负责固定骨架：

```text
输入标准化
  -> config/callback/cache/rate limit
  -> 子类 _generate/_stream
  -> 标准输出转换
  -> callback end/error
```

OpenAI 和 Anthropic 只填 provider 特有步骤。

源码：

- `libs/core/langchain_core/language_models/chat_models.py:270`
- `invoke`：约 461 行
- `stream`：约 713 行
- 抽象 `_generate`：约 2181 行

### `BaseTool`

公共 `run()` 固定：

```text
callback start
  -> 输入解析与 schema 校验
  -> 注入 config/run manager
  -> 子类 _run()
  -> 格式化 content/artifact
  -> callback end
  -> 异常处理
```

具体 Tool 主要实现 `_run()`。

源码：

- `libs/core/langchain_core/tools/base.py:405`
- `_run`：约 778 行
- `run`：约 878 行

### Prompt 和 Parser

`BasePromptTemplate` 固定 Runnable 调用和错误处理，把具体格式化留给子类；
`BaseOutputParser` 固定 generation 到解析流程，把 `parse()` 留给具体 parser。

### 设计价值

Template Method 保证 provider 和用户扩展只实现最小变化点，而 tracing、callback、
异步桥接和错误处理保持一致。

**结论：这是 LangChain 扩展契约最重要的模式。**

## 25. Visitor：未发现

Visitor 要求：

```text
element.accept(visitor)
visitor.visit_concrete_element(element)
```

并依赖双分派，使新增操作不必修改元素类。

新版虽然会遍历：

- messages
- content blocks
- Runnable 图
- schemas

但主要使用 `isinstance`、普通函数、序列化方法和 provider converter，没有统一的
`accept(visitor)` 协议。

**结论：新版主干未发现 Visitor。**

---

# 四、不要被 GoF 名称遮住的真实架构

## 26. 新版最重要的八个模式

若只保留最值得学习的八个：

### 1. Composite：Runnable 是统一组件

```text
叶子可以执行
组合也可以执行
组合还能继续被组合
```

这是 LCEL 能成立的基础。

### 2. Decorator：在调用外层叠加能力

```text
model
  -> binding
  -> retry
  -> fallback
  -> middleware
```

无需修改 provider 类。

### 3. Strategy：替换 provider 和算法

```text
模型可换
structured output 方法可换
执行环境可换
上下文编辑算法可换
```

### 4. Template Method：固定框架骨架

```text
框架处理共性
子类实现 _generate/_run 等变化点
```

### 5. Observer：运行过程对外广播

```text
业务执行
  -> callback event
  -> tracing/logging/UI
```

### 6. Builder：声明图，编译为运行单元

```text
StateGraph
  -> nodes/edges/branches/schema
  -> compile
  -> CompiledStateGraph
```

### 7. Command：用数据控制状态与路由

```text
Command(update, goto, resume, graph)
Send(node, arg)
```

### 8. Memento：持久化超步边界

```text
channel state
  -> Checkpoint
  -> Saver
  -> resume/replay/fork
```

Proxy 仍然明确存在，但理解整体执行层时，Builder、Command、Memento 比 Proxy
优先级更高：

```text
_ConfigurableModel -> 延迟真实模型
RemoteGraph        -> 代理远程图
```

## 27. 三个非 GoF、但更重要的模式

### Pipeline

`before_agent -> before_model -> model -> after_model -> tools` 是执行管道。
它比把全部 middleware 都叫责任链更准确。

### Registry

provider 名称到创建器的映射：

```text
provider string -> package/class/creator
```

它支撑简单工厂和延迟 import。Registry 不属于 GoF 23 种，但在现代框架里非常常见。

### Bulk Synchronous Parallel

LangGraph 的根本执行模型是 BSP/Pregel：

```text
并行读取当前快照
  -> 各自产生 writes
  -> 超步屏障统一归并
  -> 下一步可见
```

这比用某个 GoF 名称描述执行器更准确。它解释了：

- 为什么同一步的并发写入需要 reducer
- 为什么 `LastValue` 会拒绝多个更新
- 为什么等待边需要 barrier channel
- 为什么 checkpoint 自然落在超步边界
- 为什么失败恢复可以复用已成功任务的 pending writes

## 28. LangChain 与 LangGraph 的真实分工

旧式理解容易认为“LangChain 应该自己拥有 AgentExecutor、状态、记忆和循环”。
新版的真实边界已经改变：

```text
LangChain
  -> 定义 agent 的组件和组装方式

LangGraph
  -> 执行图、状态、checkpoint、中断、恢复
```

所以：

- `create_agent()` 是 Facade/Builder
- `AgentMiddleware` 是扩展协议
- `CompiledStateGraph` 是最终运行单元
- `PregelLoop/PregelRunner` 是执行调度核心
- Channel 决定状态归并语义
- Checkpoint/Saver 实现恢复、replay 和 time travel
- `ToolNode` 把模型 ToolCall 连接到 BaseTool 和 Command

只读 LangChain 可以理解“Agent 如何组装”，但不能完整回答“运行时实际发生什么”。
执行层必须继续读 LangGraph。

---

# 五、从模式回到六层阅读法

## 29. 意图层

新版作者希望用户：

```text
用 create_agent 快速得到可运行 Agent
用 middleware 扩展横切行为
用 provider package 接入模型
用 langchain-core 获得统一协议
用 StateGraph/Functional API 描述有状态工作流
用 checkpointer 获得 interrupt、resume 和 time travel
```

## 30. 边界层

```text
langchain_v1：Agent 门面和官方中间件
langchain-core：协议与可组合原语
partners：外部 API Adapter
langgraph：StateGraph、Pregel、channels、Command
langgraph-checkpoint：快照、pending writes、Saver
langgraph-prebuilt：ToolNode 等高层节点
```

## 31. 契约层

最重要的扩展契约：

| 扩展目标 | 最小契约 |
|---|---|
| 新 Chat Model | `BaseChatModel._generate`、`_llm_type` |
| 新 Tool | `BaseTool._run`，或直接使用 `@tool` |
| 新 Middleware | 继承 `AgentMiddleware`，实现一个或多个 hook |
| 新 Runnable | 实现 `invoke`，通常再补异步/流式能力 |
| 新输出解析器 | 实现 parser 的 `parse`/`parse_result` |
| 新 Channel | 继承 `BaseChannel`，实现 `update/get/from_checkpoint` |
| 新 Checkpointer | 实现 `BaseCheckpointSaver` 同步/异步存取契约 |
| 新 Store | 实现 `BaseStore.batch/abatch` |

这层最明显的是 Template Method 和 Adapter。

## 32. 组装层

```text
LCEL：组装通用 Runnable
create_agent：组装 Agent 图
init_chat_model：组装 provider model
middleware list：组装 hook 和 wrapper
StateGraph：组装节点、边、分支和 state schema
entrypoint/task：把普通函数编译为 Pregel 图和 task
```

这里主要是 Builder、Composite、Facade、Decorator。

## 33. 执行层

真正值得追的一条调用链：

```text
CompiledStateGraph.invoke
  -> Pregel.stream
  -> PregelLoop.tick
       -> prepare_next_tasks
       -> middleware/model/tools 中当前可运行节点
  -> PregelRunner.tick
       -> wrapper chain
       -> BaseChatModel.invoke / ToolNode
       -> provider _generate/_stream / BaseTool.run
       -> task pending writes
  -> PregelLoop.after_tick
       -> apply_writes
       -> channel reducer
       -> checkpoint
  -> 下一超步
  -> 无 task / END / interrupt
```

节点失败时，runner 按 retry policy 重试；成功 task 的 writes 会先持久化。interrupt
通过 `GraphInterrupt` 退出本轮，恢复时 `Command(resume=...)` 写入 resume value，
节点从头重放，但已完成 task 可复用 checkpoint 中的结果。

## 34. 扩展层

OpenAI 和 Anthropic 的实现说明了官方认可的扩展方式：

1. 继承 core 抽象类
2. 在 `_generate/_stream` 中适配 provider SDK
3. 输出统一 Message/Generation 类型
4. 用 `bind_tools` 转换统一 tool schema
5. 用标准测试验证模型契约

这不是把 provider SDK 直接暴露给 Agent，而是通过 Adapter 把差异压到集成包内部。

---

# 六、最终结论

## 35. 不应追求“23 种全部找到”

在新版 LangChain + LangGraph 中可靠存在的模式是：

```text
Adapter
Composite
Decorator
Facade
Proxy
Builder
Command
Iterator
Memento
Observer
Strategy
Template Method
```

以变体存在的是：

```text
Prototype
Chain of Responsibility
Mediator
```

只有局部相似的是：

```text
Factory Method
Interpreter
State
```

没有可靠证据的是：

```text
Abstract Factory
Singleton
Bridge
Flyweight
Visitor
```

## 36. 一句话理解新版 LangChain + LangGraph

新版 LangChain 用 `Runnable` Composite 统一组件，用 Template Method 和 Adapter
接入 provider，用 Decorator 与 Strategy 扩展调用行为，用 Observer 暴露运行事件，
再由 `create_agent()` 这个 Facade 驱动 `StateGraph` Builder，把它们编译成 Pregel
运行单元；LangGraph 用 Command 控制路由，用 Channel Strategy 归并并发写入，用
Memento 保存每个超步，从而实现 interrupt、resume、replay 与 time travel。

这比“LangChain 用了哪些设计模式”更接近源码本身：

```text
统一契约
  -> 适配外部实现
  -> 组合为可执行单元
  -> 包装横切能力
  -> 编译为 Pregel 图
  -> 超步并发执行
  -> 屏障归并 channel writes
  -> checkpoint 后进入下一步
```
