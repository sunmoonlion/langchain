# 集成对照表：langchain-openai

> 第 8 周交付物 | 对照 `04-契约卡片/BaseChatModel.md`

---

## 包边界

- **路径**：`/home/zym/langchain/libs/partners/openai/`
- **公开 API**（`langchain_openai/__init__.py`）：
- **依赖**：仅 core，不依赖 langchain_v1 ✅ / ❌

---

## ChatOpenAI 逐方法对照

| 方法 / 属性 | BaseChatModel 契约 | ChatOpenAI 实现 | OpenAI 特有 | 备注 |
|-------------|-------------------|-----------------|-------------|------|
| | 必须/默认 | | | |
| | | | | |

---

## 职责边界

| 层次 | 负责什么 | 不负责什么 |
|------|----------|------------|
| langchain_core | | |
| langchain-openai | | |
| 用户代码 | | |

---

## HTTP / API 层（扩展层才读）

- 请求构造位置：
- 响应 → AIMessage 映射位置：
- streaming 处理：

---

## 第二 Partner 对比（Ollama）

| 维度 | OpenAI | Ollama | 共同模式 |
|------|--------|--------|----------|
| 继承基类 | | | |
| 必须实现方法 | | | |
| 特有逻辑 | | | |

---

## 最小集成伪代码

> 「假如对接新 LLM API X，最少步骤：」

```python
# 1. 继承 BaseChatModel
# 2. 实现 ...
# 3. 注册到 init_chat_model（如需要）
```

### 必须实现的 3–5 个方法

1.
2.
3.

---

## 过关自检

- [ ] 能说明 Core 与 langchain-openai 的边界
- [ ] 能写出最小新 LLM 集成步骤
