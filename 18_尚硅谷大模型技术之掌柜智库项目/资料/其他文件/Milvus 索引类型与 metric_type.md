# Milvus 索引类型与 `metric_type` 

> 适用范围：Milvus 2.x（PyMilvus 常见写法）
> 快速选对 **索引类型** + **距离度量（metric_type）**。

---

## 1. 先记住这句话

在 Milvus 里，检索效果 = **向量本身（模型）** + **metric_type（距离定义）** + **index（加速结构）** + **搜索参数（召回/速度权衡）**。

如果你只改了索引，不匹配 metric_type，结果可能“又慢又不准”。

---

## 2. `metric_type` 是什么？

`metric_type` 决定“相似”的定义方式。

- `L2`：欧氏距离（越小越近）
- `IP`：内积（越大越近）
- `COSINE`：余弦相似度（方向越一致越近）

---

## 3. `IP`、`COSINE`、`L2` 怎么选？

### 3.1 快速决策（最常用）

1. **你用的 embedding 模型官方推荐 Cosine** 
   优先 `COSINE`
2. **你已经把向量做了 L2 归一化（单位向量）** 
   `IP` 与 `COSINE` 排序等价，二选一即可  

### 3.2 一个关键点：`IP` 和归一化

- 未归一化时，`IP` 会受向量长度影响（大范数可能更容易排前）
- 若你想表达“语义方向相似”，通常应：
  - 使用 `COSINE`，或
  - 先归一化，再用 `IP`

> 你当前代码里是 `metric_type="IP"`：如果上游 embedding 没做归一化，建议核查是否符合预期。

---

## 4. Milvus 常见索引类型（Float 向量）

> 索引本质：用空间结构换搜索速度。  
> 一般规律：**越快通常越近似**，召回率需靠参数调优。

### 4.1 `FLAT`
- 精确检索（暴力扫描）
- 召回率最高（理论 100%）
- 数据量大时慢，适合基准验证、小规模数据

### 4.2 `IVF_FLAT`
- 倒排分桶 + 桶内精确
- 参数核心：
  - `nlist`（建索引时）：桶数量
  - `nprobe`（搜索时）：探测桶数量
- 典型：百万级常用，速度与召回平衡好

### 4.3 `IVF_SQ8`
- 在 IVF 基础上做标量量化（更省内存）
- 速度快、内存低，但精度较 IVF_FLAT 略损

### 4.4 `IVF_PQ`
- 产品量化，压缩更强
- 超大规模节省显著，精度损失通常高于 SQ8
- 适合“内存压力大、可接受近似误差”的场景

### 4.5 `HNSW`
- 图索引，业界常用 ANN 方案
- 参数核心：
  - `M`（图连接度）
  - `efConstruction`（建图质量）
  - `ef`（查询扩展深度）
- 常见特点：高召回 + 低延迟，调参空间大

### 4.6 `AUTOINDEX`（托管/自动策略常见）
- 让系统自动选策略，适合先跑通业务
- 若你追求极致性能，后期建议切手工索引细调

---

## 5. 稀疏向量索引（Sparse，若你在做 BM25/稀疏检索）

### 常见
- `SPARSE_INVERTED_INDEX`
- `SPARSE_WAND`

### 常见度量
- 多使用 `IP`（结合稀疏权重打分逻辑）

---

## 6. 索引与 `metric_type` 兼容性（实战记忆版）

> 具体支持项与版本有关，以下是 2.x 常见组合：

- Float 向量：
  - `FLAT` / `IVF_*` / `HNSW` 常配：`L2`、`IP`、`COSINE`
- Binary 向量：
  - 常配：`HAMMING`、`JACCARD`
- Sparse 向量：
  - 常配：`IP`

**原则：**
1. 先按 embedding 语义选 metric  
2. 再在支持该 metric 的索引里选性能方案

---

## 7. 参数怎么调（可直接抄）

### IVF 类（`IVF_FLAT / IVF_SQ8 / IVF_PQ`）
- 建议起步：
  - `nlist = 1024`（10万~100万量级可先试）
  - 搜索 `nprobe = 16` 或 `32`
- 调优方向：
  - 提升召回：增大 `nprobe`
  - 提升速度：减小 `nprobe`
  - 数据更大：适度增大 `nlist`

### HNSW
- 建议起步：
  - `M = 16`
  - `efConstruction = 200`
  - 查询 `ef = 64` 或 `100`
- 调优方向：
  - 提升召回：增大 `ef`
  - 提升速度：减小 `ef`
  - 建索引质量：增大 `efConstruction`（建索引更慢）

---

## 8. PyMilvus 代码模板（Float 向量）

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530", token="root:Milvus")

collection_name = "demo_docs"

# 1) 建索引（示例：HNSW + COSINE）
index_params = client.prepare_index_params()
index_params.add_index(
    field_name="embedding",
    index_type="HNSW",
    metric_type="COSINE",
    params={"M": 16, "efConstruction": 200}
)
client.create_index(collection_name=collection_name, index_params=index_params)

# 2) 查询（HNSW 查询参数用 ef）
res = client.search(
    collection_name=collection_name,
    data=[[0.1, 0.2, 0.3]],   # query vector
    anns_field="embedding",
    limit=10,
    search_params={"params": {"ef": 64}}
)
```

### 如果你坚持用 `IP`（常见于推荐系统）
```python
index_params.add_index(
    field_name="embedding",
    index_type="IVF_FLAT",
    metric_type="IP",
    params={"nlist": 1024}
)

res = client.search(
    collection_name=collection_name,
    data=[[...]],
    anns_field="embedding",
    limit=10,
    search_params={"params": {"nprobe": 32}}
)
```

---

## 9. 常见坑（90% 问题都在这）

1. **metric_type 不一致**  
   - 建索引和搜索时参数要一致（同一向量字段）
2. **`IP` 但没归一化**  
   - 语义检索可能偏离预期
3. **只看延迟不看召回**  
   - 评估必须同时看 Recall@K / NDCG / 业务点击指标
4. **`nprobe`/`ef` 过小**  
   - 快是快，但结果“像随机”
5. **向量维度或字段配置错误**  
   - 插入/检索报错或结果异常
