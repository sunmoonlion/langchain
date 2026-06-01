# 契约卡片：BaseTool

> 第 4 周 | P0 优先级

## 文件路径

- 基类：
- `@tool` 装饰器：

## 继承关系

```
BaseTool
├── StructuredTool
└── ...
```

## 用户必须实现

| 方法 / 属性 | 说明 |
|-------------|------|
| `name` | |
| `description` | |
| `_run` / `_arun` | |

## 框架默认提供

| 方法 | 说明 |
|------|------|
| `invoke` | |
| schema 生成 | |

## 输入 / 输出类型

| 场景 | 输入 | 输出 |
|------|------|------|
| tool.invoke({...}) | | |

## 最小合法扩展示例（伪代码）

```python
@tool
def my_tool(x: str) -> str:
    ...
```

## 对应测试文件

- [ ] `libs/core/tests/unit_tests/...`

## 官方集成如何实现（简述）


## §@tool 装饰器入口逻辑

> 从 `tools/convert.py` 摘录关键步骤：

1.
2.

## 过关自检

- [ ] 能写出最小 Tool 伪代码
- [ ] 能说明 Tool 如何被 Agent 调用
