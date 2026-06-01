# 调用链 A：model.invoke("hi")

> 第 6 周交付物 | playground: `playground/chain_a.py`

---

## 复现脚本

- **路径**：
- **依赖**：（FakeListChatModel / mock，无需 API key）
- **运行命令**：

```bash
cd ~/langchain-reading/playground && python chain_a.py
```

---

## 调用链（文件:行号）

| 步 | 文件:行 | 函数 | 输入类型 | 输出类型 | 备注 |
|----|---------|------|----------|----------|------|
| 1 | | | `str` | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | `AIMessage` | |

---

## 数据形变表

```
str "hi"
  → ...
  → AIMessage(content="...")
```

---

## LCEL 分支（Day 41）

### 脚本：`playground/chain_a_lcel.py`

```
prompt | model | StrOutputParser
```

| 步 | 说明 |
|----|------|
| 1 | RunnableSequence.invoke 入口 |
| 2 | | |
| 3 | 最终输出类型：`str` |

---

## Callback / Config 传递

| 触发点 | 文件 | 说明 |
|--------|------|------|
| | | |

---

## 跟栈记录

| 日期 | 迷路次数 | 卡点 |
|------|----------|------|
| | | |

---

## 过关自检

- [ ] 无文档从 invoke 跟到返回，迷路 ≤ 3 次
- [ ] 数据形变表完整
