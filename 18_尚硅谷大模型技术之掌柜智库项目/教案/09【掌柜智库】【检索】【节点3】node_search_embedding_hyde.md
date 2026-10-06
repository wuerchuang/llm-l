# 掌柜智库项目(RAG)实战

## 9. 检索数据节点实现与测试

### 9.3 HyDE 检索节点（node_search_embedding_hyde）

**节点文件**: `app/process/query/agent/nodes/node_search_embedding_hyde.py`
**相关工具类位置**: `app/shared/utils/task_utils.py`

#### 场景背景与能力目标

基础向量检索可实现常规场景的稳定知识召回，但面对用户短问句、口语化、弱语义的提问场景，普遍存在语义信息稀疏、检索匹配信号不足的问题，容易引发知识漏招、召回不完整等情况。

业务中大量用户提问仅描述现象与核心诉求，缺少标准化专业表述，例如“这个怎么调”“报警灯是什么意思”“设备为什么不工作”。此类问题的主体信息完整，但原始查询语义维度单一，无法高效匹配知识库的标准化文本，导致基础检索效果受限。

为此，**node_search_embedding_hyde** 作为独立的增强检索支路，不替代基础向量检索，旨在补充优化弱语义场景的召回能力。该节点基于 HYDE（Hypothetical Document Embeddings）增强检索思路实现，核心逻辑如下：

1. 依托大模型，基于用户原始问题生成贴合业务场景、语义饱满的假设性标准回答；

2. 拼接标准化改写问句与生成的假设文档，构建高语义密度的检索输入；

3. 复用 Milvus 混合向量检索链路，生成第二路增强召回结果。

该节点核心价值：通过语义扩充，将口语化、低语义密度的原始问句，转化为贴合知识库文风的检索文本，弥补原始查询语义短板，有效提升弱语义场景下的知识召回覆盖率与完整性，解决短问句、模糊问句的漏召回问题。

#### HyDE 的核心思想

HyDE 全称是 **Hypothetical Document Embedding**，通常翻译成“假设文档向量检索”或“假设性答案检索”。

它的核心思路并不复杂，可以概括成一句话：

- **先让模型写一段“可能的标准答案”，再拿这段答案去辅助检索真实文档。**

为什么这样做会有效？原因在于：

1. 用户问题通常很短，而知识文档通常更长、更完整；
2. 直接拿短问题做向量检索时，语义信息不足；
3. 如果先补出一段更完整的答案描述，检索输入就会更接近真实文档的表达方式。

但 HyDE 也有一个前提：它生成的是“假设性答案”，并不代表事实本身。所以这一路的定位始终是：

- **增强召回，不直接作为最终答案。**

#### 实现思路

这一节的实现思路可以概括成三步：

1. **先校验检索前置条件**：主体已经确认，改写问题已经生成；
2. **再生成 HyDE 假设答案**：利用当前问题先补出一段更饱满的语义表达；
3. **最后复用普通检索链路**：把“改写问题 + 假设答案”拼接后继续做混合向量检索。

这也是当前项目里 HyDE 路径最重要的一点：

- 它没有单独再写一套 Milvus 检索逻辑，而是复用了普通向量检索的 `search_chunks()`。

这样做有两个好处：

1. 检索行为保持一致，便于后续和普通向量检索做对比与融合；
2. HyDE 只负责“扩展查询表达”，而不是额外增加一整套重复实现。

#### 步骤分解

1. 校验 `item_names` 和 `rewritten_query` 是否存在；
2. 加载 `hyde_prompt.prompt`；
3. 调用大模型生成 HyDE 假设答案；
4. 将 `rewritten_query` 与 `hyde_answer` 拼接成新的检索输入；
5. 复用 `search_chunks()` 执行混合向量检索；
6. 将结果写入 `hyde_embedding_chunks`，供后续 `RRF` 节点融合。

#### 1. 节点与依赖定位

**节点文件**: `app/process/query/agent/nodes/node_search_embedding_hyde.py`
**对应 service 文件**: `app/rag/query/search_embedding_hyde_service.py`
**相关基础设施文件**:

- `app/infra/llm/providers.py`
- `app/infra/vectorstore/milvus_gateway.py`

**相关提示词文件**:

- `app/resources/prompts/hyde_prompt.prompt`

#### 2. 节点职责与 service 入口

**节点作用**: `node_search_embedding_hyde` 负责承接查询图状态、记录节点执行进度，并调用 HyDE 检索主流程。

**对应 service 作用**: 这一节现在由一个 `service` 总入口负责承接节点传入的 `state`，再在内部拆分具体步骤：

1. `search_embedding_hyde()`：作为 HyDE 检索服务总入口，统一接收 `state`；
2. `validate_retrieval_state()`：校验 HyDE 检索前置条件是否满足；
3. `search_chunks_with_hyde()`：把 HyDE 答案和改写问题拼接后，复用普通检索链路完成检索；
4. `generate_hyde_answer()`：基于改写问题生成假设性答案。

也就是说，这一节的职责边界很明确：

- `process/query` 层负责节点调度；
- `rag/query` 层负责 HyDE 检索流程；
- `infra` 层负责模型与 Milvus 访问能力；
- `prompt` 负责约束 HyDE 假设答案的生成风格。

#### 3. 相关基础设施

##### 3.1 LLM Provider

**文件**: `app/infra/llm/providers.py`

这一节里，`llm_provider` 使用的是聊天模型能力。

```python
class LLMProvider:
    def chat(self, model: str | None = None, json_mode: bool = False) -> ChatOpenAI:
        return get_llm_client(model=model, json_mode=json_mode)
```

它在 HyDE 节点中的作用不是结构化抽取，而是：

1. 根据当前改写问题生成一段假设性标准答案；
2. 为后续检索补足语义上下文。

##### 3.2 Milvus Gateway

**文件**: `app/infra/vectorstore/milvus_gateway.py`

这一节虽然不直接在 `search_embedding_hyde_service.py` 里调用 `milvus_gateway`，但它仍然是当前链路的核心基础设施，因为 `search_chunks_with_hyde()` 最终复用了 `search_chunks()`，而 `search_chunks()` 内部会继续通过 `milvus_gateway` 完成混合检索。

所以这一路的实际链路是：

- `generate_hyde_answer()` 负责生成扩展语义；
- `search_chunks()` 负责把扩展后的查询送进 Milvus；
- `milvus_gateway` 负责真正执行检索。

#### 4. 提示词设计

HyDE 节点的提示词文件是：

- `app/resources/prompts/hyde_prompt.prompt`

当前提示词内容如下：

```text
请基于以下用户查询生成一个简洁的回答范文。
用户查询: {rewritten_query}
要求：
1. 回答要简洁明了，包含核心信息即可
2. 假设你是该领域的专家，提供专业的解释
3. 不要使用"假设"、"可能"等不确定的词汇
4. 保持回答与查询主题高度相关
5. 使用中文回答且不超过300字
```

这段提示词的设计重点有两个：

1. 让模型生成一段足够完整、足够像知识文档表达风格的文本；
2. 控制答案不要发散太远，避免 HyDE 生成内容过长或过偏。

#### 5. 业务步骤分析

这一节涉及 5 个核心函数，主链路如下：

`node_search_embedding_hyde -> search_embedding_hyde -> validate_retrieval_state -> search_chunks_with_hyde -> generate_hyde_answer -> search_chunks`

##### 5.1 `node_search_embedding_hyde`

**函数签名**: `node_search_embedding_hyde(state: dict) -> dict`

**步骤**

1. 记录当前节点开始执行；
2. 调用 `search_embedding_hyde(state)` 进入 HyDE 检索服务总入口；
3. 在 `service` 内部完成参数校验与 HyDE 检索主流程；
4. 记录当前节点执行完成；
5. 返回 `{"hyde_embedding_chunks": hyde_embedding_chunks}`。

##### 5.2 `search_embedding_hyde`

**函数签名**: `search_embedding_hyde(state: dict) -> list[dict]`

**步骤**

1. 调用 `validate_retrieval_state()` 校验检索前置条件；
2. 从 `state` 中拿到 `item_names` 和 `rewritten_query`；
3. 调用 `search_chunks_with_hyde()` 执行 HyDE 检索；
4. 返回 HyDE 检索结果列表。

##### 5.3 `validate_retrieval_state`

**函数签名**: `validate_retrieval_state(state: dict) -> tuple[list[str], str]`

**步骤**

1. 读取 `item_names`；
2. 读取 `rewritten_query`；
3. 校验二者是否为空；
4. 返回主体名称列表和改写问题。

##### 5.4 `search_chunks_with_hyde`

**函数签名**: `search_chunks_with_hyde(*, rewritten_query: str, item_names: list[str], limit: int = 5) -> tuple[str, list[dict]]`

**步骤**

1. 调用 `generate_hyde_answer()` 生成 HyDE 文本；
2. 将 `rewritten_query` 与 `hyde_answer` 拼接成 `hybrid_query`；
3. 调用 `search_chunks()` 复用普通向量检索逻辑；
4. 返回 `hyde_answer` 和检索结果列表。

##### 5.5 `generate_hyde_answer`

**函数签名**: `generate_hyde_answer(rewritten_query: str) -> str`

**步骤**

1. 加载 `hyde_prompt` 提示词；
2. 将改写问题填入提示词模板；
3. 调用 `llm_provider.chat()` 获取聊天模型；
4. 调用模型生成假设性答案；
5. 使用 `StrOutputParser()` 提取纯文本结果；
6. 返回 HyDE 假设答案。

#### 6. 节点代码实现

```python
import sys

from app.shared.runtime.logger import logger, node_log
from app.rag.query.search_embedding_hyde_service import search_embedding_hyde
from app.shared.utils.task_utils import add_done_task, add_running_task


@node_log(node_name="node_search_embedding_hyde")
def node_search_embedding_hyde(state):
    add_running_task(state["session_id"], sys._getframe().f_code.co_name, state.get("is_stream"))
    mivlus_result = search_embedding_hyde(state)
    add_done_task(state["session_id"], sys._getframe().f_code.co_name, state.get("is_stream"))
    return {"hyde_embedding_chunks": mivlus_result}
```

这一层仍然很轻，核心作用只有三件事：

1. 节点进度记录；
2. 调用下层 HyDE 检索服务总入口；
3. 返回当前节点的检索结果。

#### 7. service 总入口

`search_embedding_hyde()` 是这一节的 service 总入口。它负责承接节点传入的 `state`，先完成检索前置条件校验，再调用真正的 HyDE 检索函数，把“状态输入 -> 参数拆解 -> HyDE 检索执行”这一层职责收口在 `rag/query` 里。

```python
@step_log("search_embedding_hyde")
def search_embedding_hyde(state: dict) -> list[dict]:
    """
    HYDE增强检索节点入口函数
    作为多路检索的补充支路，专门优化短问句、口语化问句漏召回问题
    Args:
        state: 查询流程图全局状态
    Returns:
        list[dict]: 标准化的增强检索知识片段列表
    """
    # 校验检索前置参数完整性
    item_names, rewritten_query = validate_retrieval_state(state)
    # 执行HYDE增强检索，忽略假设答案，仅返回检索片段供后续流程使用
    chunks = search_chunks_with_hyde(rewritten_query=rewritten_query, item_names=item_names)
    return chunks
```

这一段主流程的价值有两点：

1. 让节点层只保留调度职责，把参数校验和 HyDE 检索执行收口到 `service`；
2. 让 HyDE 检索和普通向量检索保持一致的职责边界。

#### 8. 子函数 1：检索前置条件校验

这一层是 HyDE 路径真正开始执行前的第一步。虽然函数定义在 `search_embedding_service.py` 中，但当前 HyDE 业务链明确复用了这段逻辑。

```python
@step_log("validate_retrieval_state")
def validate_retrieval_state(state: dict) -> tuple[list[str], str]:
    """
    校验HYDE检索所需核心参数合法性
    确保前置主体确认、问句改写流程已正常执行
    Args:
        state: 查询流程图全局状态
    Returns:
        tuple: 已确认主体列表、改写后的标准查询问句
    """
    item_names = state.get("item_names")
    rewritten_query = state.get("rewritten_query")
    # 主体名称和改写问句为检索必填参数，缺失则无法执行检索
    if not item_names or not rewritten_query:
        logger.error("item_names或rewritten_query不存在,无法继续业务!")
        raise ValueError("item_names或rewritten_query不存在,无法继续业务!")
    return item_names, rewritten_query
```

这一组代码的作用，是确保 HyDE 检索启动之前，主体确认和问题改写已经完成。否则即使生成了假设答案，也无法在正确的主体范围内继续检索。

#### 9. 子函数 2：复用普通向量检索链路

```python
@step_log("search_chunks_with_hyde")
def search_chunks_with_hyde(
    *,
    rewritten_query: str,
    item_names: list[str],
    limit: int = 5,
) -> tuple[str, list[dict]]:
    """
    执行HYDE增强检索核心逻辑
    拼接原始改写问句与模型假设答案，构造高语义检索文本，完成知识库召回
    Args:
        rewritten_query: 前置流程优化后的标准查询问句
        item_names: 已确认的业务主体列表，用于检索范围过滤
        limit: 检索返回知识片段最大数量
    Returns:
        tuple: 模型生成的假设答案、HYDE增强检索后的知识片段列表
    """
    # 大模型基于用户问题生成假设性标准答案，扩充弱语义信息
    hyde_answer = generate_hyde_answer(rewritten_query)
    # 拼接原问句与假设答案，构造高语义密度的检索输入
    hybrid_query = f"{rewritten_query},{hyde_answer}"
    # 复用基础检索能力，基于增强后的文本执行Milvus混合检索
    chunks = search_chunks(rewritten_query=hybrid_query, item_names=item_names, limit=limit)
    return chunks
```

这里最重要的一点是：HyDE 节点没有再单独重复实现 Milvus 检索，而是直接复用了 `search_chunks()`。

这样做说明当前项目对两路检索的设计很清楚：

- 普通向量检索，检的是“原始改写问题”；
- HyDE 检索，检的是“改写问题 + 假设答案”。

它们的区别只在输入表达，不在底层检索流程。

#### 10. 子函数 3：生成 HyDE 假设答案

这一组代码发生在 `search_chunks_with_hyde()` 内部，用于先生成一段假设性答案，再参与后续向量检索。

```python
@step_log("generate_hyde_answer")
def generate_hyde_answer(rewritten_query: str) -> str:
    """
    生成HYDE假设答案（核心语义增强能力）
    不用于最终回复，仅用于丰富检索语义、优化向量匹配效果
    Args:
        rewritten_query: 改写后的标准用户问句
    Returns:
        str: 大模型生成的场景化、标准化假设回答文本
    """
    # 加载HYDE专属提示词模板，传入优化后问句
    prompt_str = load_prompt("hyde_prompt", rewritten_query=rewritten_query)
    # 构造大模型对话请求
    messages = [HumanMessage(content=prompt_str)]
    # 调用大模型生成假设答案，纯文本输出
    return (llm_provider.chat() | StrOutputParser()).invoke(messages)
```

这一层的核心目的，不是直接给用户返回答案，而是为了生成一段更适合作为检索输入的扩展文本。

#### 11. 关键语法补充和说明

##### 11.1 `StrOutputParser()`

```python
return (llm_provider.chat() | StrOutputParser()).invoke(messages)
```

这一节不需要结构化 JSON 输出，只需要一段自然语言文本作为 HyDE 假设答案，所以这里使用的是 `StrOutputParser()`。

它的作用是：

1. 直接提取模型返回的纯文本；
2. 省去额外消息对象解析步骤；
3. 适合 HyDE 这类“只要一段文本”的场景。

##### 11.2 为什么 HyDE 不是直接把假设答案拿去回答用户

HyDE 生成的是“假设性答案”，它的定位不是事实输出，而是检索增强。

所以这一层的使用原则非常明确：

- 只把 HyDE 答案当作检索辅助输入；
- 不把它当作最终答案直接返回给用户。

最终答案仍然要基于后续真实召回结果和答案生成节点来完成。

##### 11.3 HyDE 和普通向量检索的区别

两者最大的区别不在底层检索引擎，而在检索输入：

- 普通向量检索：直接使用 `rewritten_query`；
- HyDE 检索：使用 `rewritten_query + hyde_answer`。

因此，HyDE 的本质不是“另一套向量库搜索方案”，而是“对查询表达做一次增强”。

#### 12. 测试优化

##### 12.1 节点联调测试

如果要直接测试当前节点，可以构造一个最小状态：

```python
if __name__ == "__main__":
    mock_state = {
        "session_id": "test_hyde_session_001",
        "original_query": "HAK 180 烫金机怎么操作？",
        "rewritten_query": "HAK 180 烫金机的具体操作步骤是什么？",
        "item_names": ["HAK 180 烫金机"],
        "is_stream": False,
    }
    result = node_search_embedding_hyde(mock_state)
    print(result)
```

这类测试适合验证：

1. 节点入口是否能跑通；
2. HyDE 路检索结果是否正常生成；
3. `hyde_embedding_chunks` 是否写回正确；
4. 与普通向量检索相比，是否能补出更多候选切片。
