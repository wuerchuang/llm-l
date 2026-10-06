# 掌柜智库项目(RAG)实战

## 5. 导入数据节点实现与测试

### 5.7 存入 Milvus (node_import_milvus)

**节点文件**: `app/process/import_/agent/nodes/node_import_milvus.py`
**对应 service 文件**: `app/rag/import_/index_service.py`
**相关配置位置**: `app/rag/import_/config.py`

#### 1. 节点作用与实现思路

**节点作用**: 数据加载流程的终点，负责将处理好的结构化数据（切片内容、元数据、向量）持久化存储到向量数据库中，构建可供即时查询的索引。

**实现思路**:

1.  **幂等性设计**: 在插入新数据前，根据 `item_name` 清理旧数据，防止重复导入导致的数据污染。
2.  **Schema 适配**: 严格按照 Milvus 集合的 Schema 定义（主键、Dense字段、Sparse字段、VARCHAR元数据字段）组织数据，确保插入成功率。
3.  **混合索引构建**: 确保存入的数据能够支持 Milvus 的 Hybrid Search（Dense + Sparse 加权），最大化检索效果。

#### 2. 步骤分解

1.  **导入与配置**: 引入必要的库（pymilvus, etc.）及配置参数。
2.  **节点入口函数**: LangGraph 节点的入口，负责记录任务状态并调用 service 层。
3.  **Service 总入口**: `index_chunks()` 是入库总入口，负责串联所有子步骤。
4.  **步骤 1: 校验输入**: 校验 State 中的 `chunks`，缺失时抛出异常。
5.  **步骤 2: 准备集合**: 检查 Milvus 集合是否存在，不存在则创建 schema 和索引。
6.  **步骤 3: 清理旧数据**: 根据 `item_name` 删除已存在的切片，确保幂等性。
7.  **步骤 4: 插入数据**: 批量插入带向量的切片数据，回填生成的 chunk_id。
8.  **单元测试**: 独立运行的测试代码，验证核心流程。

#### 3. 节点与 service 定位

**节点文件**: `app/process/import_/agent/nodes/node_import_milvus.py`
**对应 service 文件**: `app/rag/import_/index_service.py`

**节点作用**: `node_import_milvus` 负责承接图状态、记录任务开始与结束，并调用 `index_chunks()`。

**对应 service 作用**: `index_chunks()` 是入库总入口，负责串联"校验输入 -> 准备集合 -> 清理旧数据 -> 插入新数据"这一整条链路。

##### 3.1 节点代码实现

```python
from app.shared.runtime.logger import logger,node_log
from app.shared.utils.task_utils import add_done_task, add_running_task
from app.process.import_.agent.state import ImportGraphState
from app.rag.import_.index_service import index_chunks

@node_log("node_import_milvus")
def node_import_milvus(state: ImportGraphState) -> ImportGraphState:
    """
    作用: 就是chunks存到milvus!
    入参: chunks
    出参: 不报错
    步骤:
          1. 日志+任务处理
          2. 参数校验(chunks不为空)
          3. 如果没有准备集合,我们创建集合(集合 schema indexs...)
          4. 删除旧数据根据item_name
          5. 插入本次的数据集合
          6. 日志+任务处理
    """
    # 入库是导入链最后一步，开始后意味着前面的解析、切分和向量化都已完成。
    add_running_task(state["task_id"], "node_import_milvus")
    # 具体的建集合、删旧数据、插入新切片逻辑都封装在 rag 层 service 中。
    state = index_chunks(state)
    add_done_task(state["task_id"], "node_import_milvus")
    return state
```

**实现要点**

- 当前节点只做三件事：记录任务开始、调用 service、记录任务结束
- 所有业务逻辑都下沉到 `index_service.py` 中
- 保持节点轻量，便于后续维护和替换

#### 4. Service 总入口

`index_chunks()` 是这一节的总入口。它本身不直接做所有细节，而是把校验、准备集合、清理旧数据和插入新数据这几步串起来。

```python
from pymilvus import DataType

from app.shared.runtime.logger import logger, step_log
from app.infra.vectorstore.milvus_gateway import milvus_gateway
from app.rag.import_.config import (
    MILVUS_CHUNK_CONTENT_MAX_LENGTH,
    MILVUS_DEFAULT_VARCHAR_MAX_LENGTH,
    MILVUS_VECTOR_DIM,
)

@step_log("index_chunks")
def index_chunks(state: dict) -> dict:
    """
    入库服务总入口
    功能：校验切块数据 → 准备 Milvus 集合 → 清理旧数据 → 批量插入新数据
    输出：更新后的 state（chunks 中会回填 chunk_id）
    """
    # 先校验切片存在，避免把空数据写入向量库
    chunks = require_chunks(state)
    
    # 集合不存在时先自动创建，保证首次导入也能直接跑通
    prepare_chunks_collection()
    
    # 获取主体名称，用于幂等性清理
    item_name = state.get("item_name", "")
    
    # 同一主体重复导入时先删旧数据，保持当前导入结果覆盖旧版本
    if item_name:
        remove_old_chunks(item_name)
    
    # 批量插入新数据
    insert_chunks(chunks)
    
    return state
```

**实现要点**

- 当前函数是整个入库流程的编排中心
- 每个子步骤都有独立的 `@step_log` 装饰器，便于追踪
- 调用 `require_chunks()` 进行输入校验，调用其他函数执行具体操作

#### 5. 业务步骤分析

这一节的主链路是：

`node_import_milvus -> index_chunks -> require_chunks -> prepare_chunks_collection -> remove_old_chunks -> insert_chunks`

##### 5.1 `node_import_milvus`

**函数签名**: `node_import_milvus(state: ImportGraphState) -> ImportGraphState`

**步骤**

1. 记录当前节点开始执行
2. 调用 `index_chunks(state)` 执行入库主流程
3. 记录当前节点执行完成
4. 返回更新后的 `state`

##### 5.2 `require_chunks`

**函数签名**: `require_chunks(state: dict) -> list[dict]`

**步骤**

1. 从 state 中获取 `chunks` 字段
2. 如果 `chunks` 为空，抛出异常终止流程
3. 返回校验后的 `chunks` 列表

##### 5.3 `prepare_chunks_collection`

**函数签名**: `prepare_chunks_collection() -> None`

**步骤**

1. 获取 Milvus 客户端
2. 检查集合是否存在，存在则直接返回
3. 创建 schema，定义字段：chunk_id（主键自增）、file_title、item_name、title、parent_title、part、content、dense_vector、sparse_vector
4. 准备索引参数：
   - 稠密向量：使用 AUTOINDEX，metric_type 为 IP
   - 稀疏向量：使用 SPARSE_INVERTED_INDEX，metric_type 为 IP，算法为 DAAT_MAXSCORE
5. 创建集合并应用索引

##### 5.4 `remove_old_chunks`

**函数签名**: `remove_old_chunks(item_name: str) -> None`

**步骤**

1. 获取 Milvus 客户端
2. 根据 `item_name` 删除已存在的切片记录
3. 完成旧数据清理

##### 5.5 `insert_chunks`

**函数签名**: `insert_chunks(chunks: list[dict]) -> None`

**步骤**

1. 获取 Milvus 客户端
2. 批量插入切片数据到集合中
3. 获取插入结果，记录插入条数和生成的 chunk_id
4. 完成数据入库

#### 6. 子函数 1：校验输入

##### 6.1 函数说明：`require_chunks`

```python
@step_log("require_chunks")
def require_chunks(state: dict) -> list[dict]:
    """
    校验导入状态中是否已经生成切块结果
    功能：确保后续流程有有效的输入数据，缺失时抛出异常
    :param state: LangGraph 流程状态字典
    :return: 已通过校验的切块列表
    """
    # 从 state 中获取核心数据
    chunks = state.get("chunks", [])
    
    # ===================== 校验 chunks =====================
    # 如果 chunks 为空，无法继续业务，直接抛出异常终止流程
    if not chunks:
        logger.error("chunks为空,无法继续业务!!")
        raise ValueError("chunks为空,无法继续业务!!")
    
    # 返回校验后的数据
    return chunks
```

**实现要点**

- `chunks` 是必填项，缺失时直接抛异常，因为无切片就无法入库
- 使用 `state.get("chunks", [])` 提供默认值，避免 KeyError
- 校验逻辑简单明确，失败时给出清晰的错误提示

#### 7. 子函数 2：准备集合

##### 7.1 配置说明

**文件**: `app/rag/import_/config.py`

```python
# Milvus VARCHAR 字段最大长度（用于 title、item_name 等短文本）
MILVUS_DEFAULT_VARCHAR_MAX_LENGTH = 512
# Milvus content 字段最大长度（用于存储长文本内容）
MILVUS_CHUNK_CONTENT_MAX_LENGTH = 65535
# Milvus 向量维度（BGE-M3 稠密向量维度）
MILVUS_VECTOR_DIM = 1024
```

当前真正参与入库逻辑的主要是这三个配置：

- `MILVUS_DEFAULT_VARCHAR_MAX_LENGTH`：控制短文本字段的最大长度
- `MILVUS_CHUNK_CONTENT_MAX_LENGTH`：控制 content 字段的最大长度
- `MILVUS_VECTOR_DIM`：控制稠密向量的维度，必须与 BGE-M3 输出一致

需要特别注意：

- 向量维度固定为 1024，与 BGE-M3 模型输出一致
- content 字段设置为 65535，支持长文本存储

##### 7.2 函数说明：`prepare_chunks_collection`

```python
@step_log("prepare_chunks_collection")
def prepare_chunks_collection() -> None:
    """
    准备 Milvus 切片集合
    功能：检查集合是否存在，不存在则创建 schema 和索引
    :return: 无返回值
    """
    # 获取 Milvus 客户端
    milvus_client = milvus_gateway.client()
    
    # 获取集合名称（从配置中读取）
    collection_name = milvus_gateway.chunks_collection
    
    # 如果集合已存在，直接返回，无需重复创建
    if milvus_client.has_collection(collection_name=collection_name):
        return

    # ===================== 创建 Schema =====================
    # 创建 schema，启用自动 ID 和动态字段
    schema = milvus_client.create_schema(auto_id=True, enable_dynamic_field=True)
    
    # 添加主键字段：chunk_id，INT64 类型，自增
    schema.add_field(field_name="chunk_id", datatype=DataType.INT64, is_primary=True, auto_id=True)
    
    # 添加文件标题字段：VARCHAR 类型，最大长度 512
    schema.add_field(field_name="file_title", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
    # 添加主体名称字段：VARCHAR 类型，最大长度 512
    schema.add_field(field_name="item_name", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
    # 添加切片标题字段：VARCHAR 类型，最大长度 512
    schema.add_field(field_name="title", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
    # 添加父标题字段：VARCHAR 类型，最大长度 512
    schema.add_field(field_name="parent_title", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
    # 添加切片序号字段：INT8 类型
    schema.add_field(field_name="part", datatype=DataType.INT8)
    
    # 添加内容字段：VARCHAR 类型，最大长度 65535（支持长文本）
    schema.add_field(field_name="content", datatype=DataType.VARCHAR, max_length=MILVUS_CHUNK_CONTENT_MAX_LENGTH)
    
    # 添加稠密向量字段：FLOAT_VECTOR 类型，维度 1024
    schema.add_field(field_name="dense_vector", datatype=DataType.FLOAT_VECTOR, dim=MILVUS_VECTOR_DIM)
    
    # 添加稀疏向量字段：SPARSE_FLOAT_VECTOR 类型
    schema.add_field(field_name="sparse_vector", datatype=DataType.SPARSE_FLOAT_VECTOR)

    # ===================== 创建索引 =====================
    # 准备索引参数
    index_params = milvus_client.prepare_index_params()
    
    # 为稠密向量创建索引：使用 AUTOINDEX，metric_type 为 IP（内积）
    index_params.add_index(
        field_name="dense_vector",
        index_type="AUTOINDEX",
        index_name="dense_vector_index",
        metric_type="IP",
    )
    
    # 为稀疏向量创建索引：使用 SPARSE_INVERTED_INDEX，算法为 DAAT_MAXSCORE
    index_params.add_index(
        field_name="sparse_vector",
        index_type="SPARSE_INVERTED_INDEX",
        index_name="sparse_vector_index",
        metric_type="IP",
        params={"inverted_index_algo": "DAAT_MAXSCORE"},
    )
    
    # 创建集合并应用索引
    milvus_client.create_collection(collection_name=collection_name, schema=schema, index_params=index_params)
```

**实现要点**

- 集合只会创建一次，后续调用会直接返回
- 使用 `AUTOINDEX` 作为稠密向量索引，Milvus 会自动选择最优索引类型
- 稀疏向量使用 `SPARSE_INVERTED_INDEX`，配合 `DAAT_MAXSCORE` 算法提升检索效率
- xxxxxxxxxx if __name__ == '__main__':    # 加载环境变量：定位项目根目录下的.env，读取模型路径/设备等配置    current_dir = os.path.dirname(os.path.abspath(__file__))    project_root = os.path.dirname(os.path.dirname(current_dir))    load_dotenv(os.path.join(project_root, ".env"))​    # 构造模拟测试状态：模拟上游节点输出的chunks数据，贴合真实业务场景    test_state = ImportGraphState({        "task_id": "test_task_embedding_001",  # 测试任务ID        "chunks": [  # 模拟带item_name的文本切片（上游商品名称识别节点产出）            {                "content": "这是一个测试文档的内容，用于验证向量化是否成功。",                "title": "测试文档标题",                "item_name": "测试项目",                "file_title": "测试文件.pdf"            },            {                "content": "这是第二个测试文档的内容，用于验证批量处理逻辑。",                "title": "测试文档标题2",                "item_name": "测试项目",                "file_title": "测试文件.pdf"            }        ]    })​    # 执行本地测试    logger.info("=== BGE-M3向量化节点本地单元测试启动 ===")    try:        # 调用核心节点函数        result_state = node_bge_embedding(test_state)        # 提取测试结果        result_chunks = result_state.get("chunks", [])​        # 打印测试结果统计        logger.info(f"=== 向量化节点本地测试完成 ===")        logger.info(f"测试任务ID：{test_state.get('task_id')}")        logger.info(f"待处理切片数：2 | 实际处理切片数：{len(result_chunks)}")        logger.info(f"返回的结果:{result_chunks}")​​    except Exception as e:        logger.error(f"=== 向量化节点本地测试失败 ===" f"错误原因：{str(e)}", exc_info=True)        # 新手友好提示：给出核心排查方向        logger.warning("排查提示：请检查BGE-M3模型路径、显存是否充足、环境变量配置是否正确")python

#### 8. 子函数 3：清理旧数据

##### 8.1 函数说明：`remove_old_chunks`

```python
@step_log("remove_old_chunks")
def remove_old_chunks(item_name: str) -> None:
    """
    根据主体名称删除已存在的切片记录
    功能：实现幂等性，确保同一主体重复导入时覆盖旧数据
    :param item_name: 主体名称
    :return: 无返回值
    """
    # 获取 Milvus 客户端并执行删除操作
    milvus_gateway.client().delete(
        collection_name=milvus_gateway.chunks_collection,
        filter=f"item_name=='{item_name}'",
    )
```

**实现要点**

- 采用"先删后插"策略，保证同一个 `item_name` 的数据是最新的
- 删除操作基于 `item_name` 过滤，确保只删除当前主体的旧数据
- 删除后无需手动调用 `load_collection()`，Milvus 会自动处理

#### 9. 子函数 4：插入数据

##### 9.1 函数说明：`insert_chunks`

```python
@step_log("insert_chunks")
def insert_chunks(chunks: list[dict]) -> None:
    """
    批量插入切片数据到 Milvus 集合
    功能：将带向量的切片数据持久化存储到向量库
    :param chunks: 带向量字段的切片列表
    :return: 无返回值
    """
    # 执行批量插入操作
    result = milvus_gateway.client().insert(
        collection_name=milvus_gateway.chunks_collection,
        data=chunks,
    )
    
    # 记录插入结果
    logger.info(f"插入数据成功! 总条数:{result.get('insert_count', 0)}")
    logger.info(f"插入数据主键回显:{result.get('ids', [])}")
```

**实现要点**

- 批量插入所有切片数据，Milvus 会自动生成 `chunk_id`
- 插入结果包含 `insert_count`（插入条数）和 `ids`（生成的主键列表）
- 日志记录插入结果，便于排查问题和监控导入进度
- 插入的数据必须包含所有 schema 定义的字段（除主键外）

#### 10. 关键语法补充和说明

##### 10.1 `milvus_gateway.client()`

```python
from app.infra.vectorstore import milvus_gateway

client = milvus_gateway.client()
```

作用是获取 Milvus 客户端单例实例，统一管理数据库连接。当前节点里它用于执行集合创建、删除、插入等操作。

##### 10.2 `auto_id=True`

```python
schema.add_field(field_name="chunk_id", datatype=DataType.INT64, is_primary=True, auto_id=True)
```

作用是启用主键自增，Milvus 会自动为每条记录生成唯一的 `chunk_id`，无需手动指定。当前节点里它用于简化数据插入流程。

##### 10.3 `enable_dynamic_field=True`

```python
schema = milvus_client.create_schema(auto_id=True, enable_dynamic_field=True)
```

作用是允许插入未在 schema 中定义的字段，提升数据结构的灵活性。当前节点里它为未来可能的字段扩展预留空间。

##### 10.4 幂等性设计

入库节点采用"先删后插"策略实现幂等性：

1. **删除阶段**：根据 `item_name` 删除已存在的旧数据
2. **插入阶段**：插入最新的数据

这样做的好处是：

- 重复导入同一文档时，不会产生重复数据
- 始终保留最新版本的切片数据
- 简化数据管理，无需手动去重

#### 11. 测试优化

这一节建议拆成两类测试：

1. **节点联调测试**：先跑上游节点（如 `node_bge_embedding`），再接 `node_import_milvus`，验证导入链连续节点能否打通
2. **service 级测试**：优先测试 `prepare_chunks_collection()` 或 `insert_chunks()`，这样更容易观察入库逻辑本身是否符合预期

##### 11.1 节点联调测试

```python
if __name__ == '__main__':
    # --- 单元测试 ---
    # 目的：验证 Milvus 导入节点的完整流程，包括连接、创建集合、清理旧数据和插入新数据。
    import sys
    import os
    from dotenv import load_dotenv

    # 加载环境变量 (自动寻找项目根目录的 .env)
    current_dir = os.path.dirname(os.path.abspath(__file__))
    project_root = os.path.dirname(os.path.dirname(current_dir))
    load_dotenv(os.path.join(project_root, ".env"))

    # 构造测试数据
    dim = 1024
    test_state = {
        "task_id": "test_milvus_task",
        "item_name":"测试项目_Milvus",
        "file_title": "test.pdf",
        "embeddings_content": [
            {
                "content": "Milvus 测试文本 1",
                "title": "测试标题",
                "item_name": "测试项目_Milvus",  # 必须有 item_name，用于幂等清理
                "parent_title":"test.pdf",
                "part":1,
                "file_title": "test.pdf",
                "dense_vector": [0.1] * dim,  # 模拟 Dense Vector
                "sparse_vector": {1: 0.5, 10: 0.8}  # 模拟 Sparse Vector
            }
,
            {
                "content": "Milvus 测试文本 2",
                "title": "测试标题2",
                "item_name": "测试项目_Milvus",  # 必须有 item_name，用于幂等清理
                "parent_title": "test.pdf2",
                "part": 1,
                "file_title": "test.pdf",
                "dense_vector": [0.2] * dim,  # 模拟 Dense Vector
                "sparse_vector": {1: 0.5, 10: 0.8}  # 模拟 Sparse Vector
            }
        ]
    }

    print("正在执行 Milvus 导入节点测试...")
    try:
        # 执行节点函数
        result_state = node_import_milvus(test_state)
    except Exception as e:
        print(f"❌ 测试失败: {e}")
```
