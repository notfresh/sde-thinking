# 如何阅读AI写的代码

## 可读性太差怎么办？

ai写的代码，确认传统方法去读已经走不通了。

现在 AI 写代码的普遍问题, 如果按照传统的方法去阅读，你除了痛苦，大概不会感觉到其他的东西。

这是我在Hermes项目中学会的。比如我曾经试图阅读Hermes agent的记忆压缩代码的时候，我得到的就是无穷无尽的沮丧，后来，我认为，这不是我的问题。

我将解释为什么。

我认为AI写的代码，必须使用AI来辅助阅读。

---

## AI 代码的特点

| 传统代码 | AI 代码 |
|----------|---------|
| 逐步构建，慢慢复杂 | 一次成型，超长函数 |
| 容易理解的结构 | 嵌套深，变量多 |
| 变量命名清晰 | 变量命名随意 |
| 注释清晰 | 注释有时还掩盖问题 |
| 边界情况少 | 边界情况塞满每个角落 |

**根源：** AI 为了"正确"，倾向于把所有边界情况都写进去，结果就是 2000 行的函数 + N 多嵌套 if。

例子如下：

**1. 嵌套太深**

```python
# 实际代码像这样
if a:
    if b:
        if c:
            if d:
                ...  # 7层嵌套
```

**2. 变量命名太通用**

```python
_existing_sp          # 什么 existing_sp?
_durable_cooldown    # durable 是什么?
_compaction_status   # compaction 和 compression 有什么区别?
```

**3. 没有分段落注释**

整块代码没有明显的视觉分隔符。

**4. 行数太长**

2000行在一个函数里，人眼扫不过来。

---

## 面对这种情况

**你不需要读懂每一行**，你只需要：

### 1. 找到入口和出口

```python
def compress_context(...) -> Tuple[list, str]:
    # ... 2000行 ...
    return compressed, new_system_prompt
```

知道**进去是什么，出来是什么**就够了。

### 2. 搜索"return"看所有退出点

```bash
grep -n "return" agent/conversation_compression.py | head -30
```

你会发现只有几种返回情况：
- 返回原始消息（跳过/失败）
- 返回压缩后消息（成功）

### 3. 找关键函数调用

```bash
grep -n "compress_fn\|commit_fence\|_release_lock" agent/conversation_compression.py
```

核心逻辑其实就几行。

---

## 现实建议

如果一段代码已经成型了，**硬读不划算**。

如果你需要改它，用**试验法**：

```
1. 猜这段代码在干什么
2. 小改一下
3. 运行测试
4. 看结果对不对
```


## 重构式阅读

原函数有2000行代码，如果为了优化和提高可读性，提高在生产环境的可维护性，那么大概率要重构。

我们可以最后通过重构来写。

以上下文压缩为例，hermes的上下文其实可以压缩成这样几个阶段：

```python
def compress_context(agent, messages, system_message, ...):
    """压缩对话上下文"""

    # === 阶段1: 准备 ===
    state = _prepare_compression(agent, messages)
    if state.should_skip():
        return state.original_messages()

    # === 阶段2: 获取锁 ===
    lock = _acquire_compression_lock(agent)
    if lock.is_contested():
        return state.original_messages()

    try:
        # === 阶段3: 执行压缩 ===
        result = _execute_compression(agent, messages, state)

        # === 阶段4: 验证结果 ===
        if not result.is_valid():
            return state.original_messages()

        # === 阶段5: 持久化 ===
        _persist_result(agent, result, in_place=True/False)

        # === 阶段6: 通知外部记忆插件 ===
        _notify_memory_manager(agent)
        _notify_context_engine(agent)

        return result.compressed_messages()

    finally:
        _release_lock(lock)
```

**每个阶段一个子函数**，一眼能看出流程。
