# MCP 版本兼容性问题及解决方案

## 当前问题

`langchain-mcp-adapters` 与 `mcp` 包存在版本不兼容问题。

### 错误信息
```
ImportError: cannot import name 'RequestContext' from 'mcp.shared.context'
ImportError: cannot import name 'ElicitationFnT' from 'mcp.client.session'
```

### 版本信息
- `langchain-mcp-adapters`: 0.3.1 (最新)
- `mcp`: 2.0.0 (最新)
- 兼容性: ❌ **不兼容**

## 问题原因

`mcp 2.0.0` 进行了重大 API 重构，移除或重命名了一些类和函数：
- `RequestContext` 已被移除或重命名
- `ElicitationFnT` 已被移除或重命名
- `langchain-mcp-adapters 0.3.1` 尚未更新适配新的 API

## 解决方案

### 方案 1: 暂时禁用 MCP（推荐）✅

使用已有的搜索提供商，性能同样优秀：

**配置** (.env):
```bash
# 禁用 MCP
# MCP_SERVERS_JSON=...

# 启用其他搜索提供商
TAVILY_API_KEY=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
SERPAPI_API_KEY=b955fc12e66f61cc23bb378d14a650c718556cac754f469c93344186af4b6b56
```

**优势**:
- ✅ 无版本冲突
- ✅ Tavily 搜索质量高
- ✅ SerpAPI 支持多个搜索引擎（百度、Google、必应等）
- ✅ DuckDuckGo 作为免费后备
- ✅ 搜索缓存优化已启用（30分钟 TTL）

**搜索优先级**:
1. MCP (禁用)
2. **Tavily** ← 高质量结果
3. **SerpAPI** ← 多引擎支持
4. **DuckDuckGo** ← 免费后备

---

### 方案 2: 降级 mcp 包（不推荐）❌

尝试使用旧版本的 `mcp`：

```bash
pip install "mcp==1.0.0" --force-reinstall
```

**问题**:
- ❌ `mcp-types==1.0.0` 不存在
- ❌ 旧版本可能有其他依赖问题
- ❌ 缺少新功能和修复

---

### 方案 3: 等待官方修复（长期）⏳

追踪以下项目的更新：

1. **langchain-mcp-adapters**
   - GitHub: https://github.com/langchain-ai/langchain
   - 等待发布兼容 `mcp 2.0.0` 的新版本

2. **mcp**
   - GitHub: https://github.com/modelcontextprotocol/python-sdk
   - 查看是否有向后兼容性修复

**检查更新**:
```bash
pip list --outdated | grep -E "(mcp|langchain)"
```

---

### 方案 4: 自定义 MCP 集成（高级）🔧

如果必须使用知乎 MCP，可以绕过 `langchain-mcp-adapters`，直接使用 `mcp` 客户端：

**创建自定义适配器** (src/career_agent/mcp_direct.py):
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class DirectMCPClient:
    """直接使用 mcp 2.0.0 SDK 的客户端"""
    
    async def connect_zhihu(self):
        server_params = StdioServerParameters(
            command="node",
            args=["E:\\zhihu_mcp\\zhihu-mcp-server"]
        )
        
        async with stdio_client(server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                tools = await session.list_tools()
                # 使用工具...
```

**优势**:
- ✅ 直接使用最新 `mcp 2.0.0` API
- ✅ 无中间层依赖冲突

**劣势**:
- ❌ 需要自己实现工具调用逻辑
- ❌ 失去 LangChain 集成的便利性
- ❌ 维护成本高

---

## 当前建议

### ✅ 使用方案 1（禁用 MCP）

理由：
1. **Tavily** 已配置且功能强大，专为 AI 应用优化
2. **SerpAPI** 支持百度搜索，适合中文内容
3. **DuckDuckGo** 作为免费后备方案
4. 搜索优化已到位（缓存、批处理、自适应）
5. 避免版本冲突和维护负担

### 性能对比

| 搜索源 | 速度 | 质量 | 成本 | 状态 |
|--------|------|------|------|------|
| 知乎 MCP | ⚡⚡⚡ | ⭐⭐⭐⭐⭐ | 免费 | ❌ 不可用 |
| Tavily | ⚡⚡ | ⭐⭐⭐⭐ | 付费 | ✅ 可用 |
| SerpAPI | ⚡⚡ | ⭐⭐⭐⭐ | 付费 | ✅ 可用 |
| DuckDuckGo | ⚡ | ⭐⭐⭐ | 免费 | ✅ 可用 |

**结论**: 即使没有知乎 MCP，Tavily + SerpAPI + DuckDuckGo 的组合已经足够强大。

---

## 后续跟进

### 定期检查更新

每周检查一次：
```bash
pip install --upgrade langchain-mcp-adapters
pip list | grep -E "(mcp|langchain)"
```

### 测试兼容性

当有新版本时：
```bash
python -c "from langchain_mcp_adapters.client import MultiServerMCPClient; print('✅ 兼容')"
```

如果成功，则可以重新启用 MCP：
```bash
# .env
MCP_SERVERS_JSON={"zhihu":{"command":"node","args":["E:\\\\zhihu_mcp\\\\zhihu-mcp-server"]}}
```

---

## 参考链接

- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
- LangChain MCP: https://github.com/langchain-ai/langchain/tree/master/libs/partners/mcp-adapters
- Tavily API: https://tavily.com/
- SerpAPI: https://serpapi.com/

---

## 更新日志

- **2026-08-02**: 发现 `langchain-mcp-adapters 0.3.1` 与 `mcp 2.0.0` 不兼容
- **当前状态**: 暂时禁用 MCP，使用 Tavily + SerpAPI + DuckDuckGo
- **等待**: 官方发布兼容版本
