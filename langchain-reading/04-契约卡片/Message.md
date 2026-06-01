# 契约卡片：Message 类型体系

> 第 4 周 | P1 优先级

## 文件路径

- 入口：`langchain_core/messages/__init__.py`

## 类型清单

| 类型 | 关键字段 | 用途 |
|------|----------|------|
| HumanMessage | | |
| AIMessage | `content`, `tool_calls` | |
| SystemMessage | | |
| ToolMessage | | |
| FunctionMessage | | legacy? |

## 继承关系

```
BaseMessage
├── ...
```

## 用户扩展点

> Message 通常是直接使用，较少子类化。记录特殊扩展场景：


## 输入 / 输出在调用链中的形变

| 步骤 | Message 类型变化 |
|------|------------------|
| 用户输入 | |
| Model 返回 | |
| Tool 执行后 | |

## 对应测试文件


## 与 Agent 状态的关系

> messages 列表如何随多轮对话增长：


## 过关自检

- [ ] 能说明 AIMessage.tool_calls 与 ToolMessage 的对应关系
