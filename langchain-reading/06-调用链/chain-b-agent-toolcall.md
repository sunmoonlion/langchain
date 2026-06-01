# 调用链 B：Agent + Tool Call

> 第 7 周交付物 | playground: `playground/chain_b.py`

---

## 复现脚本

- **路径**：
- **依赖**：
- **运行命令**：

```bash
cd ~/langchain-reading/playground && python chain_b.py
```

---

## 调用链（文件:行号）

| 步 | 文件:行 | 函数 | 输入类型 | 输出类型 | 备注 |
|----|---------|------|----------|----------|------|
| 1 | | agent.invoke | `{"messages": [...]}` | | |
| 2 | | | | | |
| 3 | | ToolNode | | | |
| 4 | | | | ToolMessage | |
| 5 | | 再次 model | | AIMessage（无 tool_calls）| 终止 |

---

## 状态流转图

```
messages: [HumanMessage]
    → [..., AIMessage(tool_calls=[...])]
    → [..., ToolMessage]
    → [..., AIMessage]  # 最终回复
```

---

## 关键分叉

| 问题 | 答案 | 代码位置 |
|------|------|----------|
| tool_calls 如何产生？ | | |
| 如何决定进入 ToolNode？ | | |
| 何时终止循环？ | | |

---

## Stream 分支（选读）

- `stream` vs `invoke` 差异：
- event 流：`test_runnable_events_v*.py` 关联：

---

## 与链 A 的公共框架层

| 公共层 | 链 A | 链 B |
|--------|------|------|
| Runnable.invoke | ✅ | |
| BaseChatModel | ✅ | ✅ |
| Message 体系 | ✅ | ✅ |
| ToolNode | | ✅ |

---

## 过关自检

- [ ] 能画出 messages 列表增长过程
- [ ] 能说明多轮 agent 循环终止条件
