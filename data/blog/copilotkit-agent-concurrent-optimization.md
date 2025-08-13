---
title: 'CopilotKit Agent 并发优化完整总结'
date: '2025-08-13'
tags: ['CopilotKit', 'LangGraph', '并发优化', 'Agent', '线程安全', 'Python', 'FastAPI']
draft: false
summary: '记录了医学文章生成AI agent在处理并发请求时出现状态冲突的完整解决方案，包括线程安全的Agent工厂、动态代理机制和测试验证。'
---

## 🎯 问题背景

原始问题：医学文章生成AI agent在处理并发请求时出现错误，两个同时执行的请求会导致第一个请求失败。

核心错误信息：
```python
TypeError: 'NoneType' object is not a mapping
at self.messages_in_process[run_id] = {
```

## 🔍 根本原因分析

1. **共享状态污染**：多个并发请求共享同一个 `LangGraphAGUIAgent` 实例
2. **线程竞争条件**：`messages_in_process` 属性在并发场景下被意外重置为 `None`
3. **会话隔离不完善**：虽然有 `SessionManager`，但 agent 层面的状态隔离不够完善
4. **第三方包限制**：`ag_ui_langgraph` 包中的 agent 类不是为并发设计的

## ✅ 解决方案实施

### 1. 创建线程安全的 Agent 工厂 (agent_factory.py)

```python
class SafeLangGraphAGUIAgent(LangGraphAGUIAgent):
    """线程安全的 Agent 包装类"""

    def __init__(self, name: str, description: str, graph, session_id: Optional[str] = None):
        super().__init__(name=name, description=description, graph=graph)
        self.session_id = session_id or str(uuid.uuid4())
        self._lock = RLock()

        # 确保 messages_in_process 永不为 None
        if not hasattr(self, 'messages_in_process') or self.messages_in_process is None:
            self.messages_in_process = {}

    def __getattribute__(self, name):
        """拦截 messages_in_process 访问，确保其永不为 None"""
        if name == 'messages_in_process':
            value = super().__getattribute__(name)
            if value is None:
                with getattr(self, '_lock', RLock()):
                    super().__setattr__(name, {})
                    value = {}
            return value
        return super().__getattribute__(name)
```

**关键特性**：
- 防御性初始化 `messages_in_process`
- 线程锁保护关键代码段
- 属性访问拦截，确保状态一致性

### 2. 创建动态 Agent 代理 (dynamic_agent.py)

```python
class DynamicAgentProxy:
    """动态 Agent 代理，为每个会话创建独立实例"""

    def _get_or_create_agent(self, session_id: Optional[str] = None) -> LangGraphAGUIAgent:
        """为每个会话获取或创建独立的 agent 实例"""
        if not session_id:
            session_id = str(uuid.uuid4())

        with self._lock:
            if session_id not in self._active_agents:
                self._active_agents[session_id] = create_safe_agent(
                    name=f"{self.base_name}_{session_id[:8]}",
                    description=self.description,
                    session_id=session_id
                )

            return self._active_agents[session_id]
```

**核心机制**：
- 基于 `threadId/runId` 进行会话隔离
- 每个请求获得独立的 agent 实例
- 自动清理过期的 agent 实例

### 3. 修改服务端配置 (demo.py)

**原代码**：
```python
add_langgraph_fastapi_endpoint(
    app=app,
    agent=LangGraphAGUIAgent(
        name="research_agent",
        description="Research agent.",
        graph=graph
    ),
    path="/copilotkit/agents/research_agent"
)
```

**修改后**：
```python
add_langgraph_fastapi_endpoint(
    app=app,
    agent=create_dynamic_agent(
        name="research_agent",
        description="Research agent with concurrent request isolation."
    ),
    path="/copilotkit/agents/research_agent"
)
```

### 4. 方法签名兼容性修复

处理了 `LangGraphAgent.run()` 方法参数不匹配的问题：

```python
async def run(self, input_data: Dict[str, Any], config: Optional[Dict[str, Any]] = None):
    try:
        # 尝试带 config 参数调用
        if config is not None:
            async for event in super().run(input_data, config):
                yield event
        else:
            async for event in super().run(input_data):
                yield event
    except TypeError as e:
        if "takes 2 positional arguments but 3 were given" in str(e):
            # 降级到单参数调用
            async for event in super().run(input_data):
                yield event
        else:
            raise
```

## 🏗️ 技术架构改进

### 修复前架构：
```
前端请求 → FastAPI → 共享的 LangGraphAGUIAgent 实例 → 状态冲突
```

### 修复后架构：
```
前端请求 → FastAPI → DynamicAgentProxy → 会话隔离的 Agent 实例 → 独立状态
```

## 🧪 创建的测试工具

### 并发测试脚本 (test_concurrent_fix.py)

```python
class ConcurrentTester:
    """专业的并发测试工具"""

    async def run_concurrent_test(self, queries: List[str], concurrent_count: int = 2):
        """运行不同并发级别的测试"""
        # 支持 2并发、3并发、5并发等多种场景
        # 详细的错误统计和性能分析
```

## 💻 前端兼容性分析

**检查结果**：前端代码与修复后的 agent 完全兼容，无需修改。

**前端架构**：
- 使用 `LangGraphHttpAgent` 通过 HTTP 调用后端 agent
- CopilotKit 自动处理会话管理
- 状态通过 `useCoAgent` 正确绑定
- 已有的 cleanup 机制支持资源清理

## 📊 解决效果

### ❌ 修复前的问题：
- 并发请求导致状态冲突
- `messages_in_process` 为 `None` 错误
- 第一个请求失败，第二个请求成功

### ✅ 修复后的效果：
- ✅ 支持多个并发请求无冲突
- ✅ 每个请求有独立的状态空间
- ✅ 防御性错误处理和状态管理
- ✅ 线程安全的 agent 实例管理
- ✅ 自动会话清理和内存管理

## 🔧 关键技术要点

1. **会话隔离**：基于 `threadId` 创建独立的 agent 实例
2. **防御性编程**：确保关键属性永不为 `None`
3. **线程安全**：使用 `RLock` 保护共享状态
4. **资源管理**：自动清理过期的 agent 实例
5. **兼容性**：处理第三方库的方法签名变化
6. **测试完备性**：专业的并发测试工具

## 🚀 部署建议

1. **重启服务**：修改后需要重启后端服务以应用新的 agent 代理
2. **监控指标**：观察并发请求的成功率和响应时间
3. **内存使用**：监控 agent 实例缓存的内存占用
4. **日志记录**：关注会话创建和清理的日志

## 📝 总结

这次优化彻底解决了 **CopilotKit + LangGraph** 的并发问题，使系统能够稳定处理多用户并发访问场景。

通过实施线程安全的 Agent 工厂、动态代理机制和完善的测试验证，我们不仅解决了当前的并发冲突问题，还为系统的可扩展性和稳定性奠定了坚实的基础。

这个解决方案适用于所有基于 CopilotKit 和 LangGraph 构建的 AI 应用，特别是需要处理多用户并发场景的生产环境。
