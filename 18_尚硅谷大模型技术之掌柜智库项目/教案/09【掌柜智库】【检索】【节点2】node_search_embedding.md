# 掌柜智库项目(RAG)实战

## 9. 检索数据节点实现与测试

### 9.2 向量检索节点（node_search_embedding）

**节点文件**: `app/process/query/agent/nodes/node_search_embedding.py`
**相关工具类位置**: `app/shared/utils/task_utils.py`

#### 场景背景与能力目标

主体识别完成后，系统开始正式检索知识库。这是一套成熟稳定的基础检索能力，核心目的是在指定产品范围内，找出和用户问题最匹配的知识内容。

作为本地知识库的主要召回入口，它需要实现两点要求：

1. 对优化后的提问做向量化处理，检索语义相似的资料；
2. 严格锁定已确认的产品主体，防止检索到其他无关内容。

`node_search_embedding` 具体工作：使用改写后的问句和产品名称作为条件，在 Milvus 中进行混合向量检索并过滤主体，最后将检索结果整理为统一格式，流转至下一环节。

#### 实现思路

这一节的实现思路可以概括成三步：

1. **先校验检索前置条件**：主体已经确认，改写问题已经生成，否则检索没有意义。
2. **再执行混合向量检索**：对改写问题生成 dense / sparse 两类向量，在 Milvus 中做混合检索。
3. **最后统一结果结构**：把 Milvus 原始返回结果整理成查询链内部统一使用的文档结构，交给后续 RRF 节点继续处理。

这一层的重点，不在于“查得多复杂”，而在于把最稳定的一路本地知识召回先做好、做标准。

#### 步骤分解

1. 校验 `item_names` 和 `rewritten_query` 是否存在；
2. 将改写问题编码成 dense / sparse 两类向量；
3. 根据主体名称构造 Milvus 过滤表达式；
4. 构造混合检索请求并执行检索；
5. 将检索结果标准化为统一文档结构；
6. 把结果写入 `embedding_chunks`，供后续融合节点使用。

#### 1. 节点与依赖定位

**节点文件**: `app/process/query/agent/nodes/node_search_embedding.py`
**对应 service 文件**: `app/rag/query/search_embedding_service.py`
**相关基础设施文件**:

- `app/infra/llm/providers.py`
- `app/infra/vectorstore/milvus_gateway.py`

#### 2. 节点职责与 service 入口

**节点作用**: `node_search_embedding` 负责承接查询图状态、记录节点任务进度，并调用下层向量检索主流程。

**对应 service 作用**: 这一节现在由一个 `service` 总入口负责承接节点传入的 `state`，再在内部拆分具体步骤：

1. `search_embedding()`：作为检索服务总入口，统一接收 `state`；
2. `validate_retrieval_state()`：校验当前检索前置条件是否满足；
3. `search_chunks()`：执行真正的混合向量检索。

`search_chunks()` 内部又进一步拆成了 3 个关键动作：

1. `build_item_name_expr()`：构造主体过滤表达式；
2. `milvus_gateway.create_requests()`：构造混合检索请求；
3. `normalize_retrieved_chunk()`：统一整理检索结果结构。

因此，这一节的职责边界非常清楚：

- `process/query` 层负责节点调度；
- `rag/query` 层负责检索业务流程；
- `infra` 层负责向量模型和 Milvus 外部能力出口。

#### 3. 相关基础设施

##### 3.1 LLM Provider

**文件**: `app/infra/llm/providers.py`

当前节点使用的是 `llm_provider` 的向量化能力，而不是聊天能力。

```python
class LLMProvider:
    def embed_documents(self, texts: list[str]) -> dict:
        return generate_embeddings(texts)
```

这一层在当前节点中的作用是：

1. 将改写问题编码成稠密向量；
2. 同时生成稀疏向量；
3. 为 Milvus 混合检索准备统一输入。

这里之所以不是只生成一种向量，是因为当前项目采用的是 **dense + sparse 混合检索** 方案：

- dense 向量更擅长表达语义相似度；
- sparse 向量更擅长表达关键词匹配信号。

##### 3.2 Milvus Gateway

**文件**: `app/infra/vectorstore/milvus_gateway.py`

当前节点里，Milvus 并不是直接用底层客户端完成调用，而是统一走 `milvus_gateway` 这层门面。

```python
class MilvusGateway:
    @property
    def chunks_collection(self) -> str:
        return infra_config.milvus.chunks_collection

    def create_requests(
        self,
        dense_vector: list[float],
        sparse_vector: dict[int, float],
        *,
        expr: str = None,
        limit: int = 5,
    ):
        return create_hybrid_search_requests(
            dense_vector=dense_vector,
            sparse_vector=sparse_vector,
            expr=expr,
            limit=limit,
        )

    def hybrid_search(
        self,
        *,
        collection_name: str,
        reqs: list[Any],
        ranker_weights: tuple[float, float] = (0.5, 0.5),
        norm_score: bool = False,
        limit: int = 5,
        output_fields: list[str] | None = None,
        search_params: dict | None = None,
    ):
        return hybrid_search(
            client=self.client(),
            collection_name=collection_name,
            reqs=reqs,
            ranker_weights=ranker_weights,
            norm_score=norm_score,
            limit=limit,
            output_fields=output_fields,
            search_params=search_params,
        )
```

在这个节点里，它主要承担三件事：

1. 提供正文切片集合名 `chunks_collection`；
2. 负责创建混合检索请求；
3. 执行最终检索。

这一层的意义在于：上层检索逻辑只关心“要检什么、怎么过滤、返回什么”，而不需要自己处理底层客户端细节。

#### 4. 配置常量

当前节点的检索策略常量写在 `search_embedding_service.py` 顶部：

```python
# ====================== 检索配置 ======================
# 默认返回的最大知识库片段数量
RETRIEVAL_DEFAULT_LIMIT = 5
# 混合检索权重：dense向量权重 0.9，sparse向量权重 0.1
RETRIEVAL_RANKER_WEIGHTS = (0.9, 0.1)
```

它们分别控制：

1. 默认最多返回多少条检索结果；
2. dense / sparse 两路检索信号的融合权重。

这里的权重设置可以理解成：

- dense 路更偏语义召回，当前占主导；
- sparse 路主要补关键词匹配信号。

#### 5. 业务步骤分析

这一节涉及 6 个核心函数，主链路如下：

`node_search_embedding -> search_embedding -> validate_retrieval_state / search_chunks -> build_item_name_expr / normalize_retrieved_chunk`

##### 5.1 `node_search_embedding`

**函数签名**: `node_search_embedding(state: dict) -> dict`

**步骤**

1. 记录当前节点开始执行；
2. 调用 `search_embedding(state)` 进入检索服务总入口；
3. 在 `service` 内部完成参数校验和混合向量检索；
4. 记录当前节点执行完成；
5. 返回 `{"embedding_chunks": embedding_chunks}`。

##### 5.2 `search_embedding`

**函数签名**: `search_embedding(state: dict) -> list[dict]`

**步骤**

1. 调用 `validate_retrieval_state()` 校验检索前置条件；
2. 从 `state` 中拿到 `item_names` 和 `rewritten_query`；
3. 调用 `search_chunks()` 执行混合向量检索；
4. 返回标准化后的切片列表。

##### 5.3 `validate_retrieval_state`

**函数签名**: `validate_retrieval_state(state: dict) -> tuple[list[str], str]`

**步骤**

1. 读取 `item_names`；
2. 读取 `rewritten_query`；
3. 校验二者是否为空；
4. 返回主体名称列表和改写问题。

##### 5.4 `search_chunks`

**函数签名**: `search_chunks(*, rewritten_query: str, item_names: list[str], limit: int = RETRIEVAL_DEFAULT_LIMIT) -> list[dict]`

**步骤**

1. 调用 `llm_provider.embed_documents()` 生成 dense / sparse 向量；
2. 调用 `build_item_name_expr()` 构造主体过滤表达式；
3. 调用 `milvus_gateway.create_requests()` 构造混合检索请求；
4. 调用 `milvus_gateway.hybrid_search()` 检索正文切片；
5. 调用 `normalize_retrieved_chunk()` 统一整理结果结构；
6. 返回标准化后的切片列表。

##### 5.5 `build_item_name_expr`

**函数签名**: `build_item_name_expr(item_names: list[str]) -> str`

**步骤**

1. 读取已确认主体名称列表；
2. 按 Milvus 过滤表达式语法拼接；
3. 返回 `item_name in [...]` 风格的表达式。

##### 5.6 `normalize_retrieved_chunk`

**函数签名**: `normalize_retrieved_chunk(chunk: dict) -> dict`

**步骤**

1. 读取 Milvus 原始返回结果；
2. 兼容 `entity` 嵌套结构；
3. 提取统一字段，例如 `chunk_id`、`item_name`、`content`、`score`；
4. 补齐内部统一字段 `type` 与 `url`；
5. 返回查询链内部标准文档结构。

#### 6. 节点代码实现

```python
import sys

from app.shared.runtime.logger import logger, node_log
from app.rag.query.search_embedding_service import search_by_embedding
from app.shared.utils.task_utils import add_done_task, add_running_task

@node_log(node_name="node_search_embedding")
def node_search_embedding(state):
    """
    节点功能：进行向量内容检索
    """
    # 先记录节点开始，和 SSE 进度展示保持一致。
    add_running_task(state["session_id"], sys._getframe().f_code.co_name, state.get("is_stream"))
    # 这里只保留节点调度职责，参数校验和检索执行统一交给 service 入口。
    mivlus_result = search_by_embedding(state)
    add_done_task(state["session_id"], sys._getframe().f_code.co_name, state.get("is_stream"))
    return {"embedding_chunks": mivlus_result}
```

#### 7. service 总入口

`search_embedding()` 是这一节的 service 总入口。它负责承接节点传入的 `state`，先完成检索前置条件校验，再调用真正的检索函数，把“状态输入 -> 参数拆解 -> 检索执行”这一层职责收口在 `rag/query` 里。

```python
from app.infra.llm.providers import llm_provider
from app.infra.vectorstore.milvus_gateway import milvus_gateway
from app.process.query.agent.state import QueryGraphState
from app.shared.runtime.logger import step_log, logger

# ====================== 检索配置 ======================
# 默认返回的最大知识库片段数量
RETRIEVAL_DEFAULT_LIMIT = 5
# 混合检索权重：dense向量权重 0.9，sparse向量权重 0.1
RETRIEVAL_RANKER_WEIGHTS = (0.9, 0.1)

@step_log("search_embedding")
def search_by_embedding(state: QueryGraphState) -> list[dict]:
    """
    【本地知识库向量检索节点】主入口
    作用：根据主体名称 + 改写查询，执行精准的知识库检索
    返回标准化后的检索结果，供后续生成答案使用
    """
    # 校验参数
    item_names, rewritten_query = validate_retrieval_state(state)

    # 执行检索并返回结果
    return search_chunks(
        rewritten_query=rewritten_query,
        item_names=item_names
    )
```

这一段主流程的价值有两点：

1. 让节点层只保留调度职责，把参数校验和业务执行收口到 `service`；
2. 把普通向量检索做成查询链内部统一、稳定的一路召回。

#### 8. 子函数 1：检索前置条件校验

```python
@step_log("validate_retrieval_state")
def validate_retrieval_state(state: dict) -> tuple[list[str], str]:
    """
    校验检索必须的核心参数是否存在
    必须包含：item_names（主体名称）、rewritten_query（改写后的查询语句）
    """
    item_names = state.get("item_names")
    rewritten_query = state.get("rewritten_query")

    # 缺少任意一个都无法执行检索
    if not item_names or not rewritten_query:
        logger.error("item_names或rewritten_query不存在,无法继续业务!")
        raise ValueError("item_names或rewritten_query不存在,无法继续业务!")

    return item_names, rewritten_query
```

这一层的作用，是防止查询链在“主体未确认”或“问题未改写”的情况下继续往下执行。

#### 9. 子函数 2：主体过滤表达式构造

```python
@step_log("build_item_name_expr")
def build_item_name_expr(item_names: list[str]) -> str:
    """
    构造 Milvus 过滤表达式
    作用：只检索属于当前 item_names 的知识库片段，避免跨主体干扰
    """
    return f"item_name in {item_names}"
```

这一层虽然很短，但它非常关键，因为它明确限定了当前检索范围只落在已确认主体对应的知识片段内。

如果没有这一层过滤，普通向量检索很容易把其它主体的相似内容一起召回进来。

#### 10. 子函数 3：结果标准化

```python
@step_log("normalize_retrieved_chunk")
def normalize_retrieved_chunk(chunk: dict) -> dict:
    """
    将 Milvus 原始检索结果，统一格式化为查询链内部标准结构
    确保后续所有节点使用相同的数据结构
    """
    entity = chunk.get("entity", chunk)
    return {
        "chunk_id": chunk.get("id") or entity.get("chunk_id"),       # 片段ID
        "item_name": entity.get("item_name", ""),                   # 归属主体名称
        "title": entity.get("title"),                               # 片段标题
        "parent_title": entity.get("parent_title"),                 # 父标题/章节
        "part": entity.get("part"),                                 # 部分标识
        "file_title": entity.get("file_title"),                     # 来源文件标题
        "content": entity.get("content", ""),                       # 片段文本内容
        "score": chunk.get("distance", 0.0),                        # 相似度分数
        "type": "milvus",                                           # 来源类型（向量库）
        "url": None,                                                # 附件URL（无）
    }
```

这一层的意义在于：Milvus 原始结果格式不适合直接在查询链各节点之间传递，所以需要先整理成统一文档结构。

这样做之后，后面的：

1. `node_rrf`
2. `node_rerank`
3. `node_answer_output`

都可以直接基于统一字段继续处理。

#### 11.子函数 4：向量查询

```python
@step_log("search_chunks")
def search_chunks(
    *,
    rewritten_query: str,
    item_names: list[str],
    limit: int = RETRIEVAL_DEFAULT_LIMIT,
) -> list[dict]:
    """
    基于改写后的查询，执行【带主体过滤】的混合向量检索
    这是本地知识库的核心检索逻辑

    Args:
        rewritten_query: 优化后的查询语句
        item_names: 已确认的主体列表，用于过滤检索范围
        limit: 最多返回几条片段

    Returns:
        标准化后的知识库片段列表
    """
    # 1. 将改写后的查询生成稠密向量 + 稀疏向量（和知识库入库时保持一致）
    embedding_result = llm_provider.embed_documents([rewritten_query])
    dense_vector = embedding_result["dense"][0]
    sparse_vector = embedding_result["sparse"][0]

    # 2. 构造混合检索请求：带主体过滤，只查当前确认的产品
    reqs = milvus_gateway.create_requests(
        dense_vector,
        sparse_vector,
        expr=build_item_name_expr(item_names),
        limit=limit,
    )

    # 3. 执行混合检索（dense + sparse 加权融合）
    resp = milvus_gateway.hybrid_search(
        collection_name=milvus_gateway.chunks_collection,
        reqs=reqs,
        ranker_weights=RETRIEVAL_RANKER_WEIGHTS,
        norm_score=True,
        limit=limit,
        output_fields=["chunk_id", "item_name", "content", "title", "parent_title", "part", "file_title"],
    )

    # 4. 把原始结果格式化为统一结构返回
    return [normalize_retrieved_chunk(chunk) for chunk in (resp[0] if resp else [])]
```

#### 12. 关键语法补充和说明

##### 12.1 为什么这里要同时用 dense 和 sparse 向量

当前节点不是单纯的 dense 检索，而是混合检索：

- dense 向量负责语义相似度；
- sparse 向量负责关键词信号；
- 两者融合后，召回稳定性更高。

所以这里的 `embed_documents()` 返回值不是单一向量，而是同时包含两类向量：

```python
embedding_result = llm_provider.embed_documents([rewritten_query])
dense_vector = embedding_result["dense"][0]
sparse_vector = embedding_result["sparse"][0]
```

##### 12.2 为什么这里要做结果标准化

当前项目后续还有：

1. `HyDE` 检索；
2. `Web Search` 检索；
3. `RRF` 融合；
4. `Rerank` 重排。

如果每一路检索都保留自己的原始结果结构，后面节点会非常难处理。

因此，这一节提前把 Milvus 结果整理成统一结构，后续多路融合时就会顺畅很多。

##### 12.3 `ranker_weights=(0.9, 0.1)` 的含义

这一组权重表示：

- 当前普通向量检索以 dense 语义召回为主；
- sparse 关键词召回作为辅助信号补充。

也就是说，这一路的检索核心仍然是语义匹配，只是在关键词命中上做了一层增强。

#### 13. 测试优化

如果要直接测试当前节点，可以构造一个最小状态：

```python
if __name__ == "__main__":
    test_state = {
        "session_id": "test_search_embedding_001",
        "rewritten_query": "HAK 180 烫金机使用说明",
        "item_names": ["HAK 180 烫金机"],
        "is_stream": False,
    }
    result = node_search_embedding(test_state)
    print(result)
```

这类测试适合验证：

1. 检索前置条件是否满足；
2. 节点入口是否能跑通；
3. `embedding_chunks` 是否正常生成；
4. 返回结构是否已经标准化。
