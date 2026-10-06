# 知乎 MCP 配置指南

## 当前配置

您的知乎 MCP 服务器位于：`E:\zhihu_mcp\zhihu-mcp-server`

## 配置步骤

### 1. 检查知乎 MCP 服务器

确保服务器文件存在且可执行：
```bash
# 检查文件
ls "E:\zhihu_mcp\zhihu-mcp-server"

# 测试运行（如果是 Node.js 项目）
node "E:\zhihu_mcp\zhihu-mcp-server"
```

### 2. 配置 MCP_SERVERS_JSON

在 `.env` 文件中添加知乎 MCP 配置：

#### 选项 A：仅知乎（推荐先测试）
```bash
MCP_SERVERS_JSON={"zhihu":{"command":"node","args":["E:\\\\zhihu_mcp\\\\zhihu-mcp-server"]}}
```

#### 选项 B：完整配置（包含所有 MCP 服务）
```bash
MCP_SERVERS_JSON={"zhihu":{"command":"node","args":["E:\\\\zhihu_mcp\\\\zhihu-mcp-server"]},"boss-zhipin":{"command":"npx","args":["-y","boss-zhipin-mcp-server"]},"xiaohongshu":{"command":"npx","args":["-y","xiaohongshu-mcp-server"]},"mcp-jobs":{"command":"npx","args":["-y","@mergedao/mcp-jobs"]}}
```

**注意**：JSON 中的路径需要使用双反斜杠 `\\\\` 进行转义。

### 3. 验证配置

运行以下命令测试知乎 MCP 是否正常工作：

```bash
# 测试离线模式（跳过 MCP）
python -m career_agent.cli --resume test.pdf --target "Python工程师" --no-search --max-companies 3

# 测试联网模式（使用 MCP）
python -m career_agent.cli --resume test.pdf --target "Python工程师" --max-companies 3
```

查看日志输出，应该看到类似：
```
[MCP] 已连接工具: search=None, boss=None, xiaohongshu=None, zhihu=zhihu_search, jobs=None
[MCP] 知乎搜索: query=字节跳动 工作体验 评价, limit=6
```

### 4. 常见问题排查

#### 问题 1：MCP 连接失败
```
[MCP] 连接失败: ...
```

**解决方案**：
1. 检查 Node.js 是否安装：`node --version`
2. 检查服务器文件路径是否正确
3. 检查服务器文件是否有执行权限
4. 查看详细错误日志

#### 问题 2：找不到知乎工具
```
[MCP] 知乎工具未配置
```

**解决方案**：
知乎 MCP 服务器需要提供符合规范的工具名称，检查：
- 工具名称应包含 "zhihu" 关键字
- 检查服务器是否正确实现了 MCP 协议

#### 问题 3：JSON 格式错误
```
json.JSONDecodeError: ...
```

**解决方案**：
使用在线 JSON 验证器检查格式：
```json
{
  "zhihu": {
    "command": "node",
    "args": ["E:\\zhihu_mcp\\zhihu-mcp-server"]
  }
}
```

然后转为单行并转义：
```bash
MCP_SERVERS_JSON={"zhihu":{"command":"node","args":["E:\\\\zhihu_mcp\\\\zhihu-mcp-server"]}}
```

### 5. 推荐的 MCP 服务器

如果本地服务器有问题，可以使用官方 npm 包：

```bash
# 安装知乎 MCP 服务器（如果有官方包）
npm install -g zhihu-mcp-server

# 然后配置为
MCP_SERVERS_JSON={"zhihu":{"command":"npx","args":["-y","zhihu-mcp-server"]}}
```

### 6. 调试模式

启用详细日志以查看 MCP 详细信息：

```bash
# 设置日志级别为 DEBUG
python -c "
import logging
logging.basicConfig(level=logging.DEBUG)
# 然后运行 CLI
"
```

或在代码中临时添加：
```python
# src/career_agent/cli.py
logging.basicConfig(
    level=logging.DEBUG,  # 改为 DEBUG
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
    datefmt='%H:%M:%S',
)
```

## 验证是否生效

成功配置后，运行时应该看到：

```
[MCP] 已连接工具: ..., zhihu=zhihu_search, ...
[MCP] 知乎搜索: query=某某公司 工作体验, limit=6
[MCP] 知乎搜索返回: [{"title": "...", "url": "...", ...}]
搜索 ← 6 条结果 (1.2s, provider=zhihu-mcp)
```

## 性能对比

### 无 MCP（使用 DuckDuckGo）
- 搜索速度：2-5秒
- 结果质量：中等
- 限流风险：中等

### 有知乎 MCP
- 搜索速度：1-2秒
- 结果质量：高（直接来自知乎）
- 限流风险：低
- 额外收益：更准确的面经和评价
