# 掌柜智库项目(RAG)实战

## 5. 导入数据节点实现与测试

### 5.5 主体识别 (node_item_name_recognition)

**节点文件**: `app/process/import_/agent/nodes/node_item_name_recognition.py`
**对应 service 文件**: `app/rag/import_/item_name_service.py`
**相关配置位置**: `app/rag/import_/config.py`

#### 1. 节点作用与实现思路

作为文档结构化解析与差异化分类的**核心关键节点**，依托大语言模型深度语义理解能力，精准萃取文档**核心主体**、业务实体与专属概念，快速判定文档所属品类与内容属性。

通过全局主体标识绑定，实现多源文档精准区分、内容归类、数据去重与精细化管控，搭建**实体与文本切片的强关联映射体系**，为后续实体级检索、语义对齐、定向过滤、结构化问答筑牢底层支撑，大幅提升知识库检索精准度与数据治理能力。

<img src="assets/image-20260425093305664.png" alt="image-20260425093305664" style="zoom:40%;" />

**实现思路**：

1. **关键上下文精准裁剪**：优先截取文档高价值头部切片，涵盖标题、概述等核心摘要信息，精简输入上下文，在控制推理成本的同时，保障大模型主体识别的准确率。
2. **大模型语义萃取识别**：结合定制化业务提示词，依托 LLM 深层语义解析能力，智能提取文档核心主体、专属名词与业务标识，完成文档类型自动判别与内容切面划分。
3. **全链路容错兜底设计**：针对大模型输出不稳定、识别异常、返回空值等场景，配置异常捕获与默认兜底策略，保障导入流程稳定运行，避免单点故障中断全链路任务。
4. **实体向量化预处理**：将识别后的标准主体实体统一完成向量编码，对接向量库实现跨表述语义匹配，打通别名关联、语义联动能力，实现模糊检索与精准召回。

#### 2. 基础设施能力说明

##### 2.1 准备embedding模型和加载工具

###### 2.1.1 什么是 “生成词向量”？

词向量（Word Vector/Embedding）就是把**文字（比如 “苏泊尔 5000W 大功率电磁炉”）转换成计算机能理解的数字列表（向量）** 的过程。

打个比方

- 人类理解文字：“苹果手机”= 品牌（苹果）+ 品类（手机）；
- 计算机理解文字：没法直接懂 “苹果手机”，但能懂 `[0.23, -0.56, 1.89, ...]` 这样的数字列表；
- 词向量的作用：把文字的**语义信息**（含义、特征、关联度）编码成数字，让计算机能 “计算文字相似度”“分类文字”“检索相似内容”。

举个简单例子

|    文字    | 对应的词向量（简化版，实际是几百 / 几千维） |
| :--------: | :-----------------------------------------: |
|  苹果手机  |          [0.23, -0.56, 1.89, 0.78]          |
|  华为手机  |          [0.21, -0.58, 1.91, 0.76]          |
| 苹果笔记本 |          [0.22, -0.55, 0.87, 0.79]          |

计算机通过对比这些数字列表的相似度，就能判断：

- “苹果手机” 和 “华为手机” 更像（数字差异小）；
- “苹果手机” 和 “苹果笔记本” 相似度低（数字差异大）。

###### 2.1.2  “稀疏向量 + 稠密向量” 

代码是基于 `BGE-M3` 模型生成两种词向量（这是当前主流的多模态嵌入方案）拆解：

|           类型            |                             特点                             |                             用途                             |
| :-----------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 稠密向量（Dense Vector）  | 长度固定（比如 768 维 / 1024 维），每个位置都是连续数值（如 0.23、-0.56） | 捕捉文字的**语义信息**（比如 “苹果手机” 的核心含义），适合相似度计算 |
| 稀疏向量（Sparse Vector） | 长度极长（比如几十万维），但只有少数位置有非 0 值，其余都是 0 | 捕捉文字的**关键词 / 字面特征**（比如 “苹果”“5000W”“电磁炉”），适合精准检索 |

BGE-M3 模型同时输出这两种向量，结合使用能兼顾 “语义理解” 和 “精准匹配”。

###### 2.1.3 bge-m3模型下载和加载

如果访问 HuggingFace 较慢，可以使用阿里云(阿里巴巴通义实验室（原达摩院）)的 ModelScope 社区下载。

https://www.modelscope.cn/models/BAAI/bge-m3 

**1.** **安装 modelscope 库**：

```python
uv add modelscope [已安装]
```

**2.** **运行 Python 脚本下载**： 创建一个临时的 Python 脚本（例如 download_bge.py）并运行：

位置: app/shared/tools/download_bgem3.py

```python
from modelscope.hub.snapshot_download import snapshot_download

# 下载模型到当前目录下的 models/bge-m3 文件夹
model_dir = snapshot_download('BAAI/bge-m3', cache_dir='D:/ai_models/modelscope_cache/models')
print(f"模型已下载到: {model_dir}")
```

**3. .env配置**

```ini
#embedding配置
# BGE-M3模型本地缓存/部署路径（本地加载模型时使用，指向ModelScope下载的模型目录）
BGE_M3_PATH=D:\ai_models\modelscope_cache\models\BAAI\bge-m3
# BGE-M3模型官方标识（ModelScope/HuggingFace通用，拉取模型时使用）
BGE_M3=BAAI/bge-m3
# BGE-M3运行设备，cuda:0表示使用第1块GPU，cpu表示使用CPU，cuda:N表示第N+1块GPU
BGE_DEVICE=cuda:0 
# BGE-M3是否开启FP16半精度推理，1=开启（GPU加速更高效），0=关闭（兼容低版本GPU/CPU）
BGE_FP16=1
```

**安装gpu版本torch**

```bash
# ===================== 环境安装命令（适配BGE-M3+Milvus，GPU/CPU版区分）=====================
# 【GPU版】安装CUDA 12.4版PyTorch（含torchvision/torchaudio，NVIDIA显卡GPU加速必备）
# 适配：有NVIDIA独显且驱动≥551.61，后续BGE-M3可开启FP16半精度推理
uv pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124

# 【备用-CPU版】无NVIDIA显卡（AMD/Intel集显）请用此命令，直接安装CPU版PyTorch
# 注释掉上方GPU版命令，取消注释下方即可
uv add torch torchvision torchaudio

# 安装Milvus和BGE-M3核心依赖（所有环境必装，无GPU/CPU区分）
# pymilvus[model]：Milvus Python客户端（带模型相关依赖，适配向量入库/检索）
# FlagEmbedding：BGE-M3向量生成模型的核心依赖（不可替代）
# transformers：FlagEmbedding底层依赖，Hugging Face模型运行库
uv add  pymilvus[model] FlagEmbedding transformers

#⚠️：安装FlagEmbedding的时候会自动安装一个cpu版本的torch替换掉之前的gpu版本的torch，
# 要解决这个问题需要做以下几个步骤
# 步骤1 先删除已经安装的FlagEmbedding（先在pyproject.toml中确定一下自己安装的版本）
uv remove FlagEmbeddin

# 步骤2 将以下内容配置在pyproject.toml中
dependencies = [
     其他之前安装过的配置,
    "flagembedding>=v1.3.5",
    "torch>=2.10.0",
    "torchvision>=0.25.0",
    "torchaudio>=2.10.0",
]

[tool.uv.sources]
# 强制从 NVIDIA 源安装
torch = { index = "pytorch-cuda" }
torchvision = { index = "pytorch-cuda" }
torchaudio = { index = "pytorch-cuda" }

[[tool.uv.index]]
name = "pytorch-cuda"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

# 步骤3 删除锁文件并重新锁定
rm uv.lock
uv lock

# 步骤4：重新同步环境
uv sync --reinstall

# 步骤5：验证
uv run python -c "import torch; print('GPU:', torch.cuda.is_available())"
```

**4. 配置参数读取 [已声明]**

文件：`app/shared/config/embedding_config.py`

```py
"""
Embedding 配置模块，负责读取向量模型相关环境变量。
"""
from dataclasses import dataclass
from app.shared.config.common import env_bool, env_str

@dataclass
class EmbeddingConfig:
    bge_m3_path: str
    bge_m3: str
    bge_device: str
    bge_fp16: bool

embedding_config = EmbeddingConfig(
    bge_m3_path=env_str("BGE_M3_PATH"),
    bge_m3=env_str("BGE_M3"),
    bge_device=env_str("BGE_DEVICE"),
    bge_fp16=env_bool("BGE_FP16"),
)
```

**5. 工具代码导入[已声明]**

文件：`app.lm.embedding_utils.py`

```python
from pymilvus.model.hybrid import BGEM3EmbeddingFunction
from app.core.logger import logger
from app.conf.embedding_config import embedding_config

# 模型单例对象，避免重复初始化
_bge_m3_ef = None

def get_bge_m3_ef():
    """
    获取BGE-M3模型单例对象，自动加载环境变量配置
    :return: 初始化完成的BGEM3EmbeddingFunction实例
    """
    global _bge_m3_ef
    # 单例模式：已初始化则直接返回，避免重复加载模型
    if _bge_m3_ef is not None:
        logger.debug("BGE-M3模型单例已存在，直接返回实例")
        return _bge_m3_ef

    # 从环境变量加载配置，无配置则使用默认值
    # 本地有可以使用本地地址！ 没有使用 "BAAI/bge-m3" 会自动下载！ 如果云端部署也可以使用url地址！
    model_name = embedding_config.bge_m3_path or "BAAI/bge-m3"
    device = embedding_config.bge_device or "cpu"
    use_fp16 = embedding_config.bge_fp16 or False

    # 打印模型初始化配置，便于问题排查
    logger.info(
        "开始初始化BGE-M3模型",
        extra={
            "model_name": model_name,
            "device": device,
            "use_fp16": use_fp16,
            "normalize_embeddings": True
        }
    )

    try:
        # 初始化BGE-M3模型，开启原生L2归一化（适配Milvus IP内积检索）
        # pymilvus.model.hybrid.BGEM3EmbeddingFunction ，在工程上最大的好处是： 
        # 和 Milvus 检索链路天然对齐 ，上线更稳更省事。
        _bge_m3_ef = BGEM3EmbeddingFunction(
            model_name=model_name,
            device=device,
            use_fp16=use_fp16,
            normalize_embeddings=True  # 模型原生对稠密+稀疏向量做L2归一化
        )
        logger.success("BGE-M3模型初始化成功，已开启原生L2归一化")
        # “它把所有向量拉伸到统一长度（模长为1），让我们能在数据库中放心使用最快的内积（IP）检索，既提速又不丢精度。”
        return _bge_m3_ef
    except Exception as e:
        logger.error(f"BGE-M3模型初始化失败：{str(e)}", exc_info=True)
        raise  # 向上抛出异常，由调用方处理


def generate_embeddings(texts):
    """
    为文本列表生成稠密+稀疏混合向量嵌入（模型原生L2归一化）
    :param texts: 要生成嵌入的文本列表，单文本也需封装为列表
    :return: 字典格式的向量结果，key为dense/sparse，对应嵌套列表/字典列表
    :raise: 向量生成过程中的异常，由调用方捕获处理
    """
    # 入参合法性校验
    if not isinstance(texts, list) or len(texts) == 0:
        logger.warning("生成向量入参不合法，texts必须为非空列表")
        raise ValueError("参数texts必须是包含文本的非空列表")

    logger.info(f"开始为{len(texts)}条文本生成混合向量嵌入")
    try:
        # 加载BGE-M3模型单例
        model = get_bge_m3_ef()
        # 模型编码生成向量，返回dense（稠密向量）+sparse（CSR格式稀疏向量）
        embeddings = model.encode_documents(texts)
        logger.debug(f"模型编码完成，开始解析稀疏向量格式，共{len(texts)}条")

        # 初始化稀疏向量处理结果，解析为字典格式（适配序列化/存储）
        processed_sparse = []
      	# 把模型输出的 CSR 稀疏矩阵 ，按“每条文本一行”拆成 {特征索引: 权重} 字典
        # - indices ：非零元素的“列号（特征ID）”
		# - data ：对应列号的权重值
		# - indptr ：每一行在 indices/data 里的起止位置指针 
        # 数据示例:
        # indices = [3, 8, 20, 1, 9]
		# data    = [0.7, 0.2, 0.1, 0.6, 0.4]
        # indptr  = [0, 3, 5]
        # 获取对应的数据
        # - 第0条文本用 0:3 => indices=[3,8,20] , data=[0.7,0.2,0.1]
		# - 第1条文本用 3:5 => indices=[1,9] , data=[0.6,0.4]
        for i in range(len(texts)):
            # 提取第i个文本的稀疏向量索引：np.int64 → Python int（满足字典key可哈希要求）
            sparse_indices = embeddings["sparse"].indices[
                embeddings["sparse"].indptr[i]:embeddings["sparse"].indptr[i + 1]
            ].tolist()
            # 提取第i个文本的稀疏向量权重：np.float32 → Python float（适配JSON序列化/接口返回）
            sparse_data = embeddings["sparse"].data[
                embeddings["sparse"].indptr[i]:embeddings["sparse"].indptr[i + 1]
            ].tolist()
            # 构造{特征索引: 归一化权重}的稀疏向量字典
            sparse_dict = {k: v for k, v in zip(sparse_indices, sparse_data)}
            processed_sparse.append(sparse_dict)

        # 构造最终返回结果，稠密向量转列表（解决numpy数组不可序列化问题）
        result = {
            "dense": [emb.tolist() for emb in embeddings["dense"]],  # 嵌套列表，与输入文本一一对应
            "sparse": processed_sparse  # 字典列表，模型已做L2归一化
        }
        logger.success(f"{len(texts)}条文本向量生成完成，格式已适配工业级使用")
        return result

    except Exception as e:
        logger.error(f"文本向量生成失败：{str(e)}", exc_info=True)
        raise  # 不吞异常，向上传递让调用方做重试/降级处理


"""
核心设计亮点&适配说明：
1. normalize_embeddings=True 的价值：
- 检索更稳定 ：不同文本长短、词频差异不会把分数拉偏。
- IP 可近似 cosine ：向量都归一化后， Inner Product 和余弦相似度等价，Milvus 用 IP 检索就很合适。
- dense/sparse 都统一标尺 ：混合检索时两路分数更容易做融合，不容易一边压死另一边。
- 减少异常高分 ：防止“模长大”的向量仅靠长度拿高分。
2. 彻底解决NumPy类型做key问题：sparse_indices加.tolist()，将np.int64转为Python原生int，满足字典key的可哈希要求，无报错风险；
3. 稀疏值适配序列化：sparse_data加.tolist()，将np.float32转为Python原生float，支持JSON写入/接口返回/Milvus入库等所有场景；
4. 单例模式优化：模型仅初始化一次，避免重复加载耗时耗资源，提升批量处理效率；
5. 格式匹配业务调用：返回dense嵌套列表、sparse字典列表，与vector_result["dense"][0]/sparse_vector["sparse"][0]取值逻辑完美契合；
6. 分级日志覆盖：从模型初始化、向量生成到异常报错，全流程日志记录，便于生产环境问题排查；
7. 入参合法性校验：防止空列表/非列表入参导致的内部报错，提升工具类健壮性。
"""
```

###### 2.1.4 修改infra/llm模型公开门类

位置: `app/infra/llm/providers.py`

```python
"""
模型提供者模块，统一封装聊天模型、Embedding 与 Reranker 的访问方式。
"""
from langchain_openai import ChatOpenAI

from app.infra.config import infra_config
from app.shared.model import generate_embeddings, get_bge_m3_ef, get_llm_client, get_reranker_model


class LLMProvider:
    def chat(self, model: str | None = None, json_mode: bool = False) -> ChatOpenAI:
        """
        获取聊天模型客户端。

        Args:
            model: 可选模型名；为空时使用默认聊天模型。
            json_mode: 是否启用 JSON 输出模式，适用于结构化抽取场景。

        Returns:
            ChatOpenAI: 可直接调用或流式调用的聊天模型客户端。
        """
        return get_llm_client(model=model, json_mode=json_mode)

    def vision_chat(self) -> ChatOpenAI:
        """
        获取视觉模型客户端。

        Returns:
            ChatOpenAI: 面向图片理解场景的视觉模型客户端。
        """
        return get_llm_client(model=infra_config.llm.lv_model)
    
    # todo 新添加内容 
    def embedding_model(self):
        """
        获取 Embedding 模型对象。

        Returns:
            Any: BGE-M3 Embedding 模型实例。
        """
        return get_bge_m3_ef()

    def embed_documents(self, texts: list[str]) -> dict:
        """
        为文本列表生成向量表示。

        Args:
            texts: 待向量化的文本列表。

        Returns:
            dict: 同时包含稠密向量与稀疏向量的结果字典。
        """
        return generate_embeddings(texts)

llm_provider = LLMProvider()
```

##### 2.2 准备milvus环境和工具配置

###### 2.2.1 **部署 Milvus 2.4.11**

```bash
centos 

# 1. 创建并进入Milvus工作目录（无则新建，有则进入原有目录）
mkdir -p ~/milvus && cd ~/milvus

# 2. 停止当前运行的旧版Milvus（全新部署执行此命令无影响）
docker compose down

# 3. 备份原有数据（关键！防止数据丢失，若为全新部署，无volumes目录可跳过此步）
mv volumes volumes.bak_2.3.5

# 4. 下载 Milvus 2.4.11 官方单机版docker-compose配置文件（覆盖原有文件）
wget https://github.com/milvus-io/milvus/releases/download/v2.4.11/milvus-standalone-docker-compose.yml -O docker-compose.yml

# 5. 启动 Milvus 2.4.11（-d 后台运行）
docker compose up -d

# 6. 检查Milvus运行状态（等待3-5秒，所有容器状态为 Up 即启动成功）
docker compose ps

# 【常用运维命令】后续管理Milvus可使用
# 停止Milvus：docker compose stop
# 重启Milvus：docker compose restart
# 查看运行日志（排查问题用）：docker compose logs milvus-standalone
```

docker-compose.yml 是一个 Docker 多容器配置文件

它的作用是：把项目需要的多个服务写在一个文件里，统一管理、统一启动

比如一个项目需要：

- web 后端服务
- redis 缓存
- mysql 数据库
- 以前要分别执行很多条 docker run 命令
- 现在只需要一个 docker-compose.yml ，再执行一条命令就能一起启动  下面加一个带有全面元素的示例 services images 都加上的示例 不要太复杂  有代码注释

示例(了解):

```yaml
# 定义所有要一起启动的服务
services:
  # 1. Web 后端服务
  app:
    image: nginx:latest              # 使用的镜像名称:版本
    container_name: my_app           # 容器名称
    ports:
      - "8080:80"                    # 本机8080 -> 容器80
    depends_on:
      - redis                        # 表示 app 依赖 redis
      - mysql                        # 表示 app 依赖 mysql
    restart: always                  # 容器异常退出后自动重启

  # 2. Redis 缓存服务
  redis:
    image: redis:7
    container_name: my_redis
    ports:
      - "6379:6379"                  # 本机6379 -> 容器6379
    restart: always

  # 3. MySQL 数据库服务
  mysql:
    image: mysql:8.0
    container_name: my_mysql
    ports:
      - "3306:3306"                  # 本机3306 -> 容器3306
    environment:
      MYSQL_ROOT_PASSWORD: 123456    # 设置 MySQL root 密码
      MYSQL_DATABASE: demo_db        # 启动时自动创建数据库
    volumes:
      - mysql_data:/var/lib/mysql    # 数据持久化，避免容器删除后数据丢失
    restart: always
# 定义数据卷
volumes:
  mysql_data:
```

**解决 MinIO 端口冲突问题（修改 docker-compose.yml）**

Milvus 内置 MinIO 用于存储向量数据，默认端口 `9000/9001` 若被本地服务占用，需修改映射端口，**编辑 `~/milvus/docker-compose.yml`**，找到 `minio` 节点，修改 ports 配置：

```yml
minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin  # 内置账号，无需修改
      MINIO_SECRET_KEY: minioadmin  # 内置密码，无需修改
    ports:
      - "9003:9001"  # 控制台端口：原9001→改为9003（避开占用）
      - "9002:9000"  # 数据端口：原9000→改为9002（避开占用）
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data  # 数据持久化目录
    command: minio server /minio_data --console-address ":9001"  # 容器内端口不变，仅改宿主机映射
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

> 改完端口后，需重启 Milvus 生效：`cd ~/milvus && docker compose restart`

**部署 Attu 2.4.0 可视化客户端（适配 Milvus 2.4.11）**

```bash
# 1. 先删除旧版Attu（若未安装，此命令无影响）
docker rm -f attu

# 2. 启动新版Attu 2.4.0（适配Milvus 2.4.11，后台运行）
# 关键：MILVUS_URL 填写Milvus服务器的IP+默认端口19530（本地部署填127.0.0.1:19530，远程服务器填公网/内网IP）
docker run -d --name attu \
    -p 8000:3000 \  # 宿主机8000端口映射容器3000端口
    -e MILVUS_URL=47.94.86.115:19530 \
    zilliz/attu:v2.4.0
```

**第五步：Attu 连接测试（验证 Milvus 部署成功）**

1. 服务器开放端口：若为云服务器 / 防火墙开启状态，需放行 8000 端口（Milvus 默认 19530 端口无需外部访问，Attu 用 8000 端口）

   ```
   # CentOS 放行8000端口
   firewall-cmd --add-port=8000/tcp --permanent
   firewall-cmd --reload
   ```

2. 浏览器访问：打开本地浏览器，输入地址 

   ```
   http://<Linux服务器IP>:8000
   ```

   - 本地虚拟机 / 服务器本地访问：`http://127.0.0.1:8000`
   - 远程服务器访问：`http://服务器公网/内网IP:8000`

3. 连接 Milvus：页面无需输入账号密码，直接点击「Connect」按钮，若能进入 Attu 可视化界面，即**Milvus 部署 + Attu 连接全部成功**。

###### 2.2.2 准备milvus配置和封装类

**1. 配置参数声明**

位置: .env

```ini
# Milvus 配置
# 切换成你milvus的url地址
MILVUS_URL=http://39.105.9.90:19530
# 存储切片集合
CHUNKS_COLLECTION=kb_chunks
# 预留
ENTITY_NAME_COLLECTION=kb_graph_entity_names
# 存储每个文档对应的实体类
ITEM_NAME_COLLECTION=kb_item_names
```

**2. 加载配置参数 [已声明]**

位置: app/shared/config/milvus_config.py

```python
"""
Milvus 配置模块，负责读取向量库相关环境变量。
"""
from dataclasses import dataclass

from app.shared.config.common import env_str

@dataclass
class MilvusConfig:
    milvus_url: str
    chunks_collection: str
    entity_name_collection: str
    item_name_collection: str

milvus_config = MilvusConfig(
    milvus_url=env_str("MILVUS_URL"),
    chunks_collection=env_str("CHUNKS_COLLECTION"),
    entity_name_collection=env_str("ENTITY_NAME_COLLECTION"),
    item_name_collection=env_str("ITEM_NAME_COLLECTION"),
)
```

**3. 解读milvus封装工具类**

位置: app/shared/clients/milvus_utils.py

```python
"""
工具模块，负责提供 milvus 相关的辅助能力。
"""
from pymilvus import MilvusClient, AnnSearchRequest, WeightedRanker
from app.shared.config.milvus_config import milvus_config
from app.shared.runtime.logger import logger

# 全局Milvus客户端实例，实现单例复用
_milvus_client: MilvusClient | None = None


def get_milvus_client() -> MilvusClient | None:
    """
    Milvus客户端单例获取方法
    实现客户端连接复用，避免重复创建连接消耗资源
    :return: MilvusClient实例，连接失败返回None
    """
    try:
        global _milvus_client
        # 单例判断：未初始化则创建新连接
        if _milvus_client is None:
            milvus_uri = milvus_config.milvus_url
            # 校验Milvus连接地址配置
            if not milvus_uri:
                logger.error("Milvus客户端连接失败：缺少MILVUS_URL环境变量配置")
                return None
            # 初始化Milvus客户端
            _milvus_client = MilvusClient(uri=milvus_uri)
            logger.info("Milvus客户端连接成功")
        return _milvus_client
    except Exception as e:
        logger.error(f"Milvus客户端连接异常：{str(e)}", exc_info=True)
        return None


def create_hybrid_search_requests(dense_vector, sparse_vector, dense_params=None, sparse_params=None, expr=None,
                                  limit=5):
    """
    构建Milvus混合搜索请求对象
    分别创建稠密/稀疏向量的搜索请求，用于后续混合搜索融合
    :param dense_vector: 文本生成的稠密向量
    :param sparse_vector: 文本生成的稀疏向量
    :param dense_params: 稠密向量搜索参数，默认使用余弦相似度
    :param sparse_params: 稀疏向量搜索参数，默认使用内积相似度
    :param expr: 搜索过滤表达式，用于精准筛选数据
    :param limit: 单向量搜索返回结果数量，默认5
    :return: 搜索请求列表，包含[dense_req, sparse_req]
    """
    # 稠密向量默认搜索参数：余弦相似度（COSINE），适配BGE-M3稠密向量并与建库参数保持一致
    if dense_params is None:
        dense_params = {"metric_type": "IP"}
    # 稀疏向量默认搜索参数：内积（IP），适配BGE-M3稀疏向量
    if sparse_params is None:
        sparse_params = {"metric_type": "IP"}

    # 构建稠密向量搜索请求，关联Milvus的dense_vector字段 近似最近邻（ANN）检索请求的核心类
    dense_req = AnnSearchRequest(
        data=[dense_vector],
        anns_field="dense_vector",
        param=dense_params,
        expr=expr, # 混合搜索的过滤条件   # 单列搜索 过滤条件 filter =
        limit=limit
    )

    # 构建稀疏向量搜索请求，关联Milvus的sparse_vector字段
    sparse_req = AnnSearchRequest(
        data=[sparse_vector],
        anns_field="sparse_vector",
        param=sparse_params,
        expr=expr,
        limit=limit
    )

    return [dense_req, sparse_req]


def hybrid_search(client, collection_name, reqs, ranker_weights=(0.5, 0.5), norm_score=False, limit=5,
                  output_fields=None, search_params=None):
    """
    执行Milvus稠密+稀疏向量混合搜索
    基于WeightedRanker实现双向量搜索结果加权融合，提升检索准确性
    :param client: MilvusClient实例
    :param collection_name: 集合名称
    :param reqs: 搜索请求列表，固定为[dense_req, sparse_req]
    :param ranker_weights: 加权融合权重，默认(0.5,0.5)，依次对应稠密/稀疏向量
    :param norm_score: 是否归一化评分后再融合，避免评分量级差异导致权重失效
    :param limit: 混合搜索最终返回结果数量，默认5
    :param output_fields: 需要返回的字段列表，默认返回item_name
    :param search_params: 搜索参数，如ef/topk等，默认None
    :return: 混合搜索结果列表，搜索失败返回None
    """
    try:
        # 初始化加权排名器：按权重融合稠密/稀疏向量的搜索结果
        # norm_score=True：先将两个向量评分归一化到0~1区间，再加权计算
        rerank = WeightedRanker(ranker_weights[0], ranker_weights[1], norm_score=norm_score)

        # 默认返回字段：文档标识字段
        if output_fields is None:
            output_fields = ["item_name"]

        # 执行混合搜索：融合稠密+稀疏向量结果，按权重重新排序
        res = client.hybrid_search(
            collection_name=collection_name,
            reqs=reqs,
            ranker=rerank,
            limit=limit,
            output_fields=output_fields,
            search_params=search_params
        )
        # res [[{id:111,distance:0.9,entity:{item_name:烫金机}},{},{},{},{}]]  || data = [1,2,3] => [[],[],[]]
        logger.info(f"Milvus混合搜索完成，集合[{collection_name}]共检索到{len(res[0])}条结果")
        return res
    except Exception as e:
        logger.error(f"Milvus混合搜索执行失败，集合[{collection_name}]：{str(e)}", exc_info=True)
        return None
```

###### 2.2.3 定义infra/vectorstore公开类

位置: app/infra/vectorstore/milvus_gateway.py

```python
"""
Milvus 门面模块，统一封装向量库客户端与检索相关操作。
"""
from typing import Any

from app.shared.clients.milvus_utils import (
    create_hybrid_search_requests,
    get_milvus_client,
    hybrid_search,
)
from app.infra.config.providers import infra_config


class MilvusGateway:
    @property
    def chunks_collection(self) -> str:
        """
        获取文档切块集合名称。

        Returns:
            str: Milvus 中存放知识切块的集合名。
        """
        return infra_config.milvus.chunks_collection

    @property
    def item_name_collection(self) -> str:
        """
        获取主体名称集合名称。

        Returns:
            str: Milvus 中存放主体名称向量的集合名。
        """
        return infra_config.milvus.item_name_collection

    def client(self):
        """
        获取 Milvus 客户端实例。

        Returns:
            Any: 底层 Milvus 客户端对象。
        """
        return get_milvus_client()

    def create_requests(
        self,
        dense_vector: list[float],
        sparse_vector: dict[int, float],
        *,
        expr: str = None,
        limit: int = 5,
    ):
        """
        创建 Milvus 混合检索请求对象。

        Args:
            dense_vector: 稠密向量表示。
            sparse_vector: 稀疏向量表示。
            expr: 可选过滤表达式，用于限定检索范围。
            limit: 单路检索返回条数上限。

        Returns:
            Any: 底层 Milvus 混合检索请求列表。
        """
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
        """
        执行 Milvus 混合检索。

        Args:
            collection_name: 目标集合名称。
            reqs: 检索请求列表。
            ranker_weights: 稠密与稀疏路召回结果的融合权重。
            norm_score: 是否对分数做归一化。
            limit: 最终返回条数上限。
            output_fields: 需要返回的字段列表。
            search_params: 额外检索参数。

        Returns:
            Any: Milvus 返回的原始检索结果。
        """
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

milvus_gateway = MilvusGateway()
```

#### 3. 节点与 service 定位

**节点文件**: `app/process/import_/agent/nodes/node_item_name_recognition.py`
**对应 service 文件**: `app/rag/import_/item_name_service.py`

**节点作用**: `node_item_name_recognition` 负责承接图状态、记录任务开始与结束，并调用 `recognize_and_index_item_name()`。

**对应 service 作用**: `recognize_and_index_item_name()` 是主体识别总入口，负责串联"校验输入 -> 构建上下文 -> 调用 LLM 识别 -> 回填数据 -> 生成向量 -> 准备集合 -> 入库"这一整条链路。

##### 3.1 节点代码实现

```python
from app.shared.runtime.logger import logger,node_log
from app.shared.utils.task_utils import add_done_task, add_running_task
from app.process.import_.agent.state import ImportGraphState
from app.rag.import_.item_name_service import recognize_and_index_item_name

@node_log("node_item_name_recognition")
def node_item_name_recognition(state: ImportGraphState) -> ImportGraphState:
    """
    节点: 主体识别 (node_item_name_recognition)
    为什么叫这个名字: 识别文档核心描述的物品/商品名称 (Item Name)。
    """
    # 主体识别节点开始后，会尝试从文档内容中抽取当前资料对应的核心产品名。
    add_running_task(state["task_id"], "node_item_name_recognition")
    # 识别出的主体名不仅会写回 state，还会同步写入主体向量索引。
    state = recognize_and_index_item_name(state)
    add_done_task(state["task_id"], "node_item_name_recognition")
    return state
```

**实现要点**

- 当前节点只做三件事：记录任务开始、调用 service、记录任务结束
- 所有业务逻辑都下沉到 `item_name_service.py` 中
- 保持节点轻量，便于后续维护和替换

#### 4. 业务步骤分析

这一节的主链路是：

`node_item_name_recognition -> recognize_and_index_item_name -> validate_chunks_and_title -> build_document_context -> recognize_item_name -> apply_item_name -> embed_item_name -> upsert_item_name`

##### 4.1 `node_item_name_recognition`

**函数签名**: `node_item_name_recognition(state: ImportGraphState) -> ImportGraphState`

**步骤**

1. xxxxxxxxxx if __name__ == '__main__':    # 加载环境变量：定位项目根目录下的.env，读取模型路径/设备等配置    current_dir = os.path.dirname(os.path.abspath(__file__))    project_root = os.path.dirname(os.path.dirname(current_dir))    load_dotenv(os.path.join(project_root, ".env"))​    # 构造模拟测试状态：模拟上游节点输出的chunks数据，贴合真实业务场景    test_state = ImportGraphState({        "task_id": "test_task_embedding_001",  # 测试任务ID        "chunks": [  # 模拟带item_name的文本切片（上游商品名称识别节点产出）            {                "content": "这是一个测试文档的内容，用于验证向量化是否成功。",                "title": "测试文档标题",                "item_name": "测试项目",                "file_title": "测试文件.pdf"            },            {                "content": "这是第二个测试文档的内容，用于验证批量处理逻辑。",                "title": "测试文档标题2",                "item_name": "测试项目",                "file_title": "测试文件.pdf"            }        ]    })​    # 执行本地测试    logger.info("=== BGE-M3向量化节点本地单元测试启动 ===")    try:        # 调用核心节点函数        result_state = node_bge_embedding(test_state)        # 提取测试结果        result_chunks = result_state.get("chunks", [])​        # 打印测试结果统计        logger.info(f"=== 向量化节点本地测试完成 ===")        logger.info(f"测试任务ID：{test_state.get('task_id')}")        logger.info(f"待处理切片数：2 | 实际处理切片数：{len(result_chunks)}")        logger.info(f"返回的结果:{result_chunks}")​​    except Exception as e:        logger.error(f"=== 向量化节点本地测试失败 ===" f"错误原因：{str(e)}", exc_info=True)        # 新手友好提示：给出核心排查方向        logger.warning("排查提示：请检查BGE-M3模型路径、显存是否充足、环境变量配置是否正确")python
2. 调用 `recognize_and_index_item_name(state)` 执行主体识别主流程
3. 记录当前节点执行完成
4. 返回更新后的 `state`

##### 4.2 `validate_chunks_and_title`

**函数签名**: `validate_chunks_and_title(state: dict) -> tuple[list[dict], str]`

**步骤**

1. 从 state 中获取 `chunks` 和 `file_title`
2. 如果 `chunks` 为空，抛出异常终止流程
3. 如果 `file_title` 为空，使用默认值 "default_title" 兜底
4. 返回校验后的 `chunks` 和 `file_title`

##### 4.3 `build_document_context`

**函数签名**: `build_document_context(chunks: list[dict]) -> str`

**步骤**

1. 截取前 K 个切片（由 `ITEM_NAME_CONTEXT_CHUNK_K` 控制）
2. 遍历切片，拼接格式化字符串："切片:{index},标题:{title},内容:{content}"
3. 将所有切片字符串用换行符连接
4. 截断到最大字符数限制（由 `ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS` 控制）
5. 返回拼接后的上下文字符串

##### 4.4 `recognize_item_name`

**函数签名**: `recognize_item_name(context: str, file_title: str) -> str`

**步骤**

1. 获取 LLM 客户端
2. 加载系统提示词模板 `product_recognition_system`
3. 加载用户提示词模板 `item_name_recognition`，传入 `file_title` 和 `context`
4. 构造消息列表（SystemMessage + HumanMessage）
5. 调用 LLM 并解析输出
6. 如果识别结果为空，使用 `file_title` 兜底
7. 返回识别出的主体名称

##### 4.5 `apply_item_name`

**函数签名**: `apply_item_name(chunks: list[dict], item_name: str) -> list[dict]`

**步骤**

1. 遍历所有切片
2. xxxxxxxxxx if __name__ == '__main__':    # --- 单元测试 ---    # 目的：验证 Milvus 导入节点的完整流程，包括连接、创建集合、清理旧数据和插入新数据。    import sys    import os    from dotenv import load_dotenv​    # 加载环境变量 (自动寻找项目根目录的 .env)    current_dir = os.path.dirname(os.path.abspath(__file__))    project_root = os.path.dirname(os.path.dirname(current_dir))    load_dotenv(os.path.join(project_root, ".env"))​    # 构造测试数据    dim = 1024    test_state = {        "task_id": "test_milvus_task",        "item_name":"测试项目_Milvus",        "file_title": "test.pdf",        "embeddings_content": [            {                "content": "Milvus 测试文本 1",                "title": "测试标题",                "item_name": "测试项目_Milvus",  # 必须有 item_name，用于幂等清理                "parent_title":"test.pdf",                "part":1,                "file_title": "test.pdf",                "dense_vector": [0.1] * dim,  # 模拟 Dense Vector                "sparse_vector": {1: 0.5, 10: 0.8}  # 模拟 Sparse Vector            },            {                "content": "Milvus 测试文本 2",                "title": "测试标题2",                "item_name": "测试项目_Milvus",  # 必须有 item_name，用于幂等清理                "parent_title": "test.pdf2",                "part": 1,                "file_title": "test.pdf",                "dense_vector": [0.2] * dim,  # 模拟 Dense Vector                "sparse_vector": {1: 0.5, 10: 0.8}  # 模拟 Sparse Vector            }        ]    }​    print("正在执行 Milvus 导入节点测试...")    try:        # 执行节点函数        result_state = node_import_milvus(test_state)    except Exception as e:        print(f"❌ 测试失败: {e}")python
3. 返回更新后的切片列表

##### 4.6 `embed_item_name`

**函数签名**: `embed_item_name(item_name: str) -> tuple[list[float], dict[int, float]]`

**步骤**

1. 调用 `llm_provider.embed_documents()` 生成向量
2. 提取稠密向量（dense）和稀疏向量（sparse）
3. 返回两个向量

##### 4.7 `prepare_item_name_collection`

**函数签名**: `prepare_item_name_collection() -> None`

**步骤**

1. 获取 Milvus 客户端
2. 检查集合是否存在，存在则直接返回
3. 创建 schema，定义字段：pk（主键自增）、file_title、item_name、dense_vector、sparse_vector
4. 准备索引参数：
   - 稠密向量：使用 AUTOINDEX，metric_type 为 IP
   - 稀疏向量：使用 SPARSE_INVERTED_INDEX，metric_type 为 IP，算法为 DAAT_MAXSCORE
5. 创建集合并应用索引

##### 4.8 `upsert_item_name`

**函数签名**: `upsert_item_name(item_name: str, file_title: str, dense_vector: list[float], sparse_vector: dict[int, float]) -> None`

**步骤**

1. 获取 Milvus 客户端
2. 调用 `prepare_item_name_collection()` 确保集合存在
3. 对 `item_name` 进行转义处理，防止注入攻击
4. 删除已存在的相同 `item_name` 记录（保证幂等性）
5. 插入新记录，包含 file_title、item_name、dense_vector、sparse_vector
6. 完成入库操作

#### 5. Service 总入口

`recognize_and_index_item_name()` 是这一节的总入口。它本身不直接做所有细节，而是把校验、上下文构建、LLM 识别、数据回填、向量生成和入库这几步串起来。

```python
from app.shared.runtime.load_prompt import load_prompt
from app.shared.runtime.logger import logger, step_log
from app.infra.llm import llm_provider
from app.infra.vectorstore import milvus_gateway
from app.rag.import_.config import (
    ITEM_NAME_CONTEXT_CHUNK_K,
    ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS,
    MILVUS_DEFAULT_VARCHAR_MAX_LENGTH,
    MILVUS_VECTOR_DIM,
)
from app.shared.utils.escape_milvus_string_utils import escape_milvus_string


@step_log("recognize_and_index_item_name")
def recognize_and_index_item_name(state: dict) -> dict:
    """
    主体识别服务总入口
    功能：从文档切片中识别主体名称 → 回填到 state 和 chunks → 生成向量 → 写入 Milvus 主体索引
    输出：更新后的 state，包含 item_name 和带有 item_name 的 chunks
    """
    # 1. 校验输入数据，确保 chunks 和 file_title 有效
    chunks, file_title = validate_chunks_and_title(state)
    
    # 2. 从前若干个切片拼接上下文，给模型一个足够稳定的识别窗口
    context = build_document_context(chunks)
    
    # 3. 让模型输出当前文档对应的主体名，识别失败时会回退到文件标题
    item_name = recognize_item_name(context, file_title)
    
    # 4. 将识别结果写回 state 和 chunks
    state["item_name"] = item_name
    state["chunks"] = apply_item_name(chunks, item_name)
    
    # 5. 主体名本身也会生成向量，便于查询阶段做主体确认
    dense_vector, sparse_vector = embed_item_name(item_name)
    
    # 6. 准备集合并写入 Milvus
    upsert_item_name(item_name, file_title, dense_vector, sparse_vector)
    
    return state
```

**实现要点**

- 当前函数是整个主体识别流程的编排中心
- 每个子步骤都有独立的 `@step_log` 装饰器，便于追踪
- 向量生成和入库都在同一个函数中完成，保证事务一致性

**配置说明**

**文件**: `app/rag/import_/config.py`

```python
# 主体识别上下文切片数：取前 K 个切片用于 LLM 识别
ITEM_NAME_CONTEXT_CHUNK_K = 5
# 主体识别上下文总字符数上限：防止上下文过长导致大模型输入超限
ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS = 10000

# Milvus 向量维度（BGE-M3 稠密向量维度）
MILVUS_VECTOR_DIM = 1024
# Milvus VARCHAR 字段最大长度
MILVUS_DEFAULT_VARCHAR_MAX_LENGTH = 512
```

当前真正参与主体识别逻辑的主要是前两个：

- `ITEM_NAME_CONTEXT_CHUNK_K`：控制用于识别的切片数量
- `ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS`：控制上下文字符总数上限

需要特别注意：

- 当前 `item_name_service.py` 的主流程里，实际使用的是这两个配置
- 向量维度固定为 1024，与 BGE-M3 模型输出一致

#### 6. 子函数 1：校验输入

##### 6.1 函数说明：`validate_chunks_and_title`

```python
@step_log("validate_chunks_and_title")
def validate_chunks_and_title(state: dict) -> tuple[list[dict], str]:
    """
    校验并提取 state 中的 chunks 和 file_title
    功能：确保后续流程有有效的输入数据，缺失时提供兜底策略
    :param state: LangGraph 流程状态字典
    :return: (chunks列表, file_title字符串)
    """
    # 从 state 中获取核心数据
    chunks = state.get("chunks", [])
    file_title = state.get("file_title")
    
    # ===================== 校验 chunks =====================
    # 如果 chunks 为空，无法继续业务，直接抛出异常终止流程
    if not chunks:
        logger.error("chunks没有内容,无法继续业务!")
        raise ValueError("chunks没有内容,无法继续业务!")
    
    # ===================== 处理 file_title 缺失场景 =====================
    # 如果标题为空，使用默认值兜底，避免后续流程出错
    if not file_title:
        logger.warning("file_title为空给与默认值处理!")
        file_title = "default_title"
        state["file_title"] = file_title  # 回填到 state
    
    # 返回校验后的数据
    return chunks, file_title
```

**实现要点**

- `chunks` 是必填项，缺失时直接抛异常，因为无切片就无法识别主体
- `file_title` 是可选的，缺失时使用默认值兜底，不会中断流程
- 两个字段都会回填到 state，保证下游节点能正常使用

#### 7. 子函数 2：构建上下文

##### 7.1 函数说明：`build_document_context`

```python
@step_log("build_document_context")
def build_document_context(chunks: list[dict]) -> str:
    """
    从前 K 个切片中构建用于 LLM 识别的上下文字符串
    功能：提取文档头部高价值信息，控制上下文长度，降低推理成本
    :param chunks: 文档切片列表，每个元素包含 title、content 等字段
    :return: 拼接后的上下文字符串，已做长度截断
    """
    # 截取前 K 个切片（由配置 ITEM_NAME_CONTEXT_CHUNK_K 控制）
    current_chunks = chunks[:ITEM_NAME_CONTEXT_CHUNK_K]
    
    # 存储格式化后的切片字符串
    chunk_str_list: list[str] = []
    
    # 遍历切片，拼接格式化字符串
    for index, item in enumerate(current_chunks, start=1):
        chunk_str_list.append(f"切片:{index},标题:{item['title']},内容:{item['content']}")
    
    # 用换行符连接所有切片
    chunk_str = "\n".join(chunk_str_list)
    
    # 截断到最大字符数限制（由配置 ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS 控制）
    return chunk_str[:ITEM_NAME_CONTEXT_TOTAL_MAX_CHARS]
```

**实现要点**

- 只取前 K 个切片，因为文档头部通常包含标题、概述等高价值信息
- 每个切片都格式化为统一结构，便于 LLM 理解
- 最终结果会做长度截断，防止超出大模型上下文窗口

#### 8. 子函数 3：调用 LLM 识别

##### 8.1 函数说明：`recognize_item_name`

```python
@step_log("recognize_item_name")
def recognize_item_name(context: str, file_title: str) -> str:
    """
    调用大语言模型识别文档主体名称
    功能：基于文档上下文和提示词，让 LLM 输出当前文档对应的核心产品名
    :param context: 由前 K 个切片拼接的上下文字符串
    :param file_title: 文件标题，用于兜底
    :return: 识别出的主体名称，如果识别失败则返回 file_title
    """
    # 获取 LLM 客户端（从 llm_provider 中获取）
    llm = llm_provider.chat()
    
    # 加载系统提示词模板
    system_prompt_str = load_prompt("product_recognition_system")
    
    # 加载用户提示词模板，传入 file_title 和 context
    user_prompt_str = load_prompt("item_name_recognition", file_title=file_title, context=context)
    
    # 构造消息列表：系统提示词 + 用户提示词
    messages = [
        SystemMessage(content=system_prompt_str),
        HumanMessage(content=user_prompt_str),
    ]
    
    # 调用 LLM 并解析输出（使用 StrOutputParser 去除多余格式）
    item_name = (llm | StrOutputParser()).invoke(messages)
    
    # 兜底处理：如果识别结果为空，使用 file_title
    return item_name or file_title
```

**实现要点**

- 使用 `llm_provider.chat()` 获取 LLM 客户端，统一模型管理
- 提示词分为系统提示词和用户提示词，职责清晰
- 使用 `StrOutputParser` 确保输出是纯文本，避免 JSON 等格式干扰
- 识别失败时使用 `file_title` 兜底，保证流程不中断

#### 9. 子函数 4：回填数据

##### 9.1 函数说明：`apply_item_name`

```python
@step_log("apply_item_name")
def apply_item_name(chunks: list[dict], item_name: str) -> list[dict]:
    """
    将识别出的主体名称回填到所有切片中
    功能：为每个切片添加 item_name 字段，建立切片与主体的关联关系
    :param chunks: 文档切片列表
    :param item_name: 识别出的主体名称
    :return: 更新后的切片列表
    """
    # 遍历所有切片，添加 item_name 字段
    for chunk in chunks:
        chunk["item_name"] = item_name
    
    # 返回更新后的切片列表
    return chunks
```

**实现要点**

- 简单直接的遍历赋值，为每个切片添加相同的 `item_name`
- 这样做的好处是后续检索时，可以通过 `item_name` 过滤出同一文档的切片
- 保持数据结构的一致性，便于下游节点使用

#### 10. 子函数 5：生成向量

##### 10.1 函数说明：`embed_item_name`

```python
@step_log("embed_item_name")
def embed_item_name(item_name: str) -> tuple[list[float], dict[int, float]]:
    """
    为主体名称生成稠密和稀疏向量
    功能：调用 Embedding 模型，将文本转换为可用于向量检索的数字表示
    :param item_name: 主体名称字符串
    :return: (稠密向量列表, 稀疏向量字典)
    """
    # 调用 llm_provider 的 embed_documents 方法生成向量
    # 注意：即使只有一个文本，也要封装为列表传入
    result = llm_provider.embed_documents([item_name])
    
    # 提取第一个（也是唯一一个）文本的稠密向量和稀疏向量
    # dense: list[float] - 长度为 1024 的浮点数列表
    # sparse: dict[int, float] - {特征索引: 权重} 的字典
    return result["dense"][0], result["sparse"][0]
```

**实现要点**

- 使用 `llm_provider.embed_documents()` 统一调用 Embedding 模型
- 返回的是 BGE-M3 模型输出的混合向量：稠密向量 + 稀疏向量
- 稠密向量用于语义相似度计算，稀疏向量用于关键词匹配
- 两者结合可以兼顾“语义理解”和“精准匹配”

#### 11. 子函数 6：准备集合并入库

##### 11.1 函数说明：`prepare_item_name_collection`

```python
@step_log("prepare_item_name_collection")
def prepare_item_name_collection() -> None:
    """
    准备 Milvus 主体名称集合
    功能：检查集合是否存在，不存在则创建 schema 和索引
    :return: 无返回值
    """
    # 获取 Milvus 客户端
    milvus_client = milvus_gateway.client()
    
    # 获取集合名称（从配置中读取）
    collection_name = milvus_gateway.item_name_collection
    
    # 如果集合已存在，直接返回，无需重复创建
    if milvus_client.has_collection(collection_name=collection_name):
        return

    # ===================== 创建 Schema =====================
    # 创建 schema，启用自动 ID 和动态字段
    schema = milvus_client.create_schema(auto_id=True, enable_dynamic_field=True)
    
    # 添加主键字段：pk，INT64 类型，自增
    schema.add_field(field_name="pk", datatype=DataType.INT64, is_primary=True, auto_id=True)
    
    # 添加文件标题字段：VARCHAR 类型，最大长度 512 nullable=True
    schema.add_field(field_name="file_title", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
    # 添加主体名称字段：VARCHAR 类型，最大长度 512
    schema.add_field(field_name="item_name", datatype=DataType.VARCHAR, max_length=MILVUS_DEFAULT_VARCHAR_MAX_LENGTH)
    
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
- 两个向量字段都使用 `IP`（内积）作为相似度度量方式

##### 11.2 函数说明：`upsert_item_name`

```python
@step_log("upsert_item_name")
def upsert_item_name(item_name: str, file_title: str, dense_vector: list[float], sparse_vector: dict[int, float]) -> None:
    """
    将主体名称及向量写入 Milvus 集合（先删后插，保证幂等性）
    功能：确保同一个 item_name 在集合中只有一条最新记录
    :param item_name: 主体名称
    :param file_title: 文件标题
    :param dense_vector: 稠密向量列表
    :param sparse_vector: 稀疏向量字典
    :return: 无返回值
    """
    # 获取 Milvus 客户端
    milvus_client = milvus_gateway.client()
    
    # 确保集合已创建
    prepare_item_name_collection()
    
    # 对 item_name 进行转义处理，防止特殊字符导致注入攻击
    safe_item_name = escape_milvus_string(item_name)
    
    # ===================== 删除已有记录 =====================
    # 先删除已存在的相同 item_name 记录，保证幂等性
    milvus_client.delete(
        collection_name=milvus_gateway.item_name_collection,
        filter=f"item_name == '{safe_item_name}'",
    )
    
    # ===================== 插入新记录 =====================
    # 插入包含完整信息的新记录
    milvus_client.insert(
        collection_name=milvus_gateway.item_name_collection,
        data=[
            {
                "file_title": file_title,
                "item_name": item_name,
                "dense_vector": dense_vector,
                "sparse_vector": sparse_vector,
            }
        ],
    )
```

**实现要点**

- 采用“先删后插”策略，保证同一个 `item_name` 只有一条记录
- 使用 `escape_milvus_string()` 对 `item_name` 进行转义，防止注入攻击
- 插入的数据包含 file_title、item_name、dense_vector、sparse_vector 四个字段
- 主键 `pk` 由 Milvus 自动生成，无需手动指定

#### 12. 关键语法补充和说明

##### 12.1 `escape_milvus_string`

```python
from app.shared.utils.escape_milvus_string_utils import escape_milvus_string

safe_name = escape_milvus_string("华为 Mate60 Pro")
print(safe_name)
```

作用是对字符串进行转义处理，防止特殊字符（如单引号、双引号等）导致 Milvus 查询语句出错或注入攻击。当前节点里它用于保护 `item_name` 过滤条件。

##### 12.2 `llm_provider.chat()`

```python
from app.infra.llm import llm_provider

llm = llm_provider.chat()
```

作用是获取 LLM 客户端实例，统一管理模型配置和调用。当前节点里它用于调用大模型识别主体名称。

##### 12.3 `llm_provider.embed_documents()`

```python
from app.infra.llm import llm_provider

result = llm_provider.embed_documents(["华为 Mate60 Pro"])
dense = result["dense"][0]
sparse = result["sparse"][0]
```

作用是调用 Embedding 模型生成混合向量。返回结果是一个字典，包含 `dense`（稠密向量列表）和 `sparse`（稀疏向量字典列表）。当前节点里它用于为主体名称生成向量。

#### 13. 测试优化

这一节建议拆成两类测试：

1. **节点联调测试**：先跑上游节点（如 `node_document_split`），再接 `node_item_name_recognition`，验证导入链连续节点能否打通
2. **service 级测试**：优先测试 `recognize_item_name()` 或 `build_document_context()`，这样更容易观察识别逻辑本身是否符合预期

##### 13.1 节点联调测试

```python
# ===================== 本地测试方法（直接运行调试，无需启动LangGraph） =====================
def test_node_item_name_recognition():
    """
    商品名称识别节点本地测试方法
    功能：模拟LangGraph流程输入，独立测试node_item_name_recognition节点全链路逻辑
    适用场景：本地开发、调试、单节点功能验证，无需启动整个LangGraph流程
    测试前准备：
        1. 确保项目环境变量配置完成（MILVUS_URL/ITEM_NAME_COLLECTION等）
        2. 确保大模型、Milvus、BGE-M3服务均可正常访问
        3. 确保prompt模板（item_name_recognition/product_recognition_system）已存在
    使用方法：
        直接运行该函数：if __name__ == "__main__": test_node_item_name_recognition()
    """
    logger.info("=== 开始执行商品名称识别节点本地测试 ===")
    try:
        # 1. 构造模拟的ImportGraphState状态（模拟上游节点产出数据）
        mock_state = ImportGraphState({
            "task_id": "test_task_123456",  # 测试任务ID
            "file_title": "华为Mate60 Pro手机使用说明书",  # 模拟文件标题
            "file_name": "华为Mate60Pro说明书.pdf",  # 模拟原始文件名（兜底用）
            # 模拟文本切片列表（上游切片节点产出，含title/content字段）
            "chunks": [
                {
                    "title": "产品简介",
                    "content": "华为Mate60 Pro是华为公司2023年发布的旗舰智能手机，搭载麒麟9000S芯片，支持卫星通话功能，屏幕尺寸6.82英寸，分辨率2700×1224。"
                },
                {
                    "title": "拍照功能",
                    "content": "华为Mate60 Pro后置5000万像素超光变摄像头+1200万像素超广角摄像头+4800万像素长焦摄像头，支持5倍光学变焦，100倍数字变焦。"
                },
                {
                    "title": "电池参数",
                    "content": "电池容量5000mAh，支持88W有线超级快充，50W无线超级快充，反向无线充电功能。"
                }
            ]
        })

        # 2. 调用商品名称识别核心节点
        result_state = node_item_name_recognition(mock_state)

        # 3. 打印测试结果（调试用）
        logger.info("=== 商品名称识别节点本地测试完成 ===")
        logger.info(f"测试任务ID：{result_state.get('task_id')}")
        logger.info(f"最终识别商品名称：{result_state.get('item_name')}")
        logger.info(f"切片数量：{len(result_state.get('chunks', []))}")
        logger.info(f"第一个切片商品名称：{result_state.get('chunks', [{}])[0].get('item_name')}")

    except Exception as e:
        logger.error(f"商品名称识别节点本地测试失败，原因：{str(e)}", exc_info=True)


# 测试方法运行入口：直接执行该文件即可触发测试
if __name__ == "__main__":
    # 执行本地测试
    test_node_item_name_recognition()
```



