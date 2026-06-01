# 契约卡片：BaseChatModel

> 第 4 周 | P0 优先级

## 文件路径

- 基类：
- 父类 BaseLanguageModel：

## 继承关系

```
BaseLanguageModel
└── BaseChatModel
    └── ChatOpenAI（扩展层）
```

## 用户必须实现

| 方法 | 签名 | 说明 |
|------|------|------|
| | | |

## 框架默认提供

| 方法 | 说明 |
|------|------|
| `invoke` | |
| `stream` | |
| `_generate` 调用链 | |

## 输入 / 输出类型

| 场景 | 输入 | 输出 |
|------|------|------|
| `invoke("hi")` | | |
| `invoke([HumanMessage(...)])` | | |

## 最小合法扩展示例（伪代码）

```python
class MyChatModel(BaseChatModel):
    ...
```

## 对应测试文件

- [ ] `libs/core/tests/unit_tests/language_models/`
- [ ] `libs/standard-tests/langchain_tests/unit_tests/chat_models.py`

## 官方集成如何实现（简述）

（OpenAI：扩展层填写）

## §测试规格摘要

1.
2.

## 过关自检

- [ ] 能说明 str 如何变成 AIMessage
- [ ] 能列出对接新 LLM API 最少要实现的方法
