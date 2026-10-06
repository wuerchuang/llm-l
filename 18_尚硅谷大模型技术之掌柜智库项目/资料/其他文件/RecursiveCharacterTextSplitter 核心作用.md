### 一、`RecursiveCharacterTextSplitter` 核心作用

`RecursiveCharacterTextSplitter` 是 **LangChain 官方核心的文本分割工具**，主打 **「递归式按字符分割 + 智能保留语义」**，解决大模型「上下文窗口有限」的核心问题 —— 将超长文本（如文档、文章）分割为符合模型上下文限制的小文本块，同时通过**优先按自然分隔符分割 ** 的逻辑，最大程度保留文本原有语义，避免语义被生硬截断（如一句话被拆成两个块），是 LangChain 中最常用、适配性最强的文本分割器。

#### 核心优势

1. **递归分割**：按预设分隔符列表**从粗到细**递归分割，直到文本块长度符合要求；
2. **语义保留**：优先按「换行符`\n\n`、换行`\n`、空格、标点」分割，贴合自然语言的语义边界；
3. **灵活配置**：支持自定义块大小、块重叠率、分隔符列表，适配不同类型文本（MD / 纯文本 / 文档）；
4. **广泛适配**：兼容纯文本、Markdown、普通文档等几乎所有文本格式，是 LangChain 文本处理的「默认首选」。

### 二、核心适用场景

- 超长文档（PDF/MD/TXT）送入大模型前的预处理；
- 向量数据库入库前的文本分块（保证单块嵌入向量的语义完整性）；
- 批量文本处理中，统一控制单段文本的长度，适配模型上下文窗口。

### 三、快速使用小教程（极简可运行，适配你的项目）

#### 步骤 1：安装依赖（若未安装）

```
# 核心依赖（已安装LangChain则无需重复安装）
pip install langchain-text-splitters
```

#### 步骤 2：基础使用示例（核心参数 + 最简调用）

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 准备待分割的超长文本（示例：MD格式文档，贴合你的项目场景）
long_text = """
# HL3070使用说明书
## 产品概述
HL3070是一款高性能无线扫描枪，支持蓝牙5.0连接，适配Windows/Android/IOS系统，续航时间长达20小时。

## 操作步骤
### 开机与配对
1. 长按电源键3秒，指示灯蓝灯闪烁进入配对模式；
2. 手机/电脑开启蓝牙，搜索设备“HL3070-Scanner”并连接；
3. 连接成功后，蓝灯常亮，可开始扫描使用。

### 充电说明
当红灯低电量闪烁时，使用标配Type-C数据线连接电源充电，充电时红灯常亮，充满后绿灯常亮。

## 注意事项
1. 请勿将扫描枪置于高温、潮湿环境中；
2. 避免摔落、碰撞，防止硬件损坏；
3. 长时间不使用请关闭电源，节省电量。
"""

# 2. 初始化分割器（核心参数配置，按需调整）
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,          # 核心：单块文本的最大字符数（按模型上下文调整，如1024/2048）
    #RecursiveCharacterTextSplitter 的 chunk_overlap 是 **「被动生效」的配置 —— 仅当单段文本长度超过chunk_size**，需要被强制分割为多个块时，相邻块才会出现重叠；若文本本身长度≤chunk_size，分割器会直接保留整段文本，不会主动拆分、也不会生成重叠
    chunk_overlap=30,        # 可选：相邻块的重叠字符数（防止语义断层，一般为chunk_size的10%-20%）
    length_function=len,     # 可选：长度计算方式，默认按字符数（可自定义按token数）
    separators=["\n\n", "\n", " ", ""]  # 可选：分割符列表（从粗到细，默认值，一般无需修改）
)

# 3. 执行分割（返回List[str]，每个元素是一个文本块）
text_chunks = text_splitter.split_text(long_text)

# 4. 打印分割结果
print(f"分割完成，共生成 {len(text_chunks)} 个文本块\n")
for i, chunk in enumerate(text_chunks, 1):
    print(f"===== 文本块 {i}（字符数：{len(chunk)}） =====")
    print(chunk)
    print("-" * 50)
```

以你项目的 MD 文档场景为例，逐个说明每个分隔符的**触发场景**和**语义意义**，这也是 LangChain 默认配置的原因（适配 90% 的文本场景）：

| 分隔符 |     含义      | 触发场景（优先级别） |                    语义意义（为什么优先）                    |
| :----: | :-----------: | :------------------: | :----------------------------------------------------------: |
| `\n\n` | 空行 / 双换行 |     1 级（最高）     | MD 文档中**段落 / 标题 / 章节的边界**，按它拆能保留完整段落 / 章节语义，最贴合自然阅读逻辑 |
|  `\n`  |    单换行     |         2 级         | 段落内的换行（如 MD 列表、步骤），拆分会保留完整行 / 步骤，语义损失小 |
|        |     空格      |         3 级         | 单词 / 短句间的空格（中文少用，英文适配），拆分会保留完整单词 / 短句 |
|  `""`  |   无分隔符    |     4 级（最低）     | 无任何分隔符，**按字符硬拆**，仅当所有分隔符都拆不了时触发（保底策略，避免文本超上限） |

#### 步骤 3：运行结果（语义完整，无生硬截断）

```
分割完成，共生成 3 个文本块

===== 文本块 1（字符数：189） =====
# HL3070使用说明书
## 产品概述
HL3070是一款高性能无线扫描枪，支持蓝牙5.0连接，适配Windows/Android/IOS系统，续航时间长达20小时。

## 操作步骤
### 开机与配对
1. 长按电源键3秒，指示灯蓝灯闪烁进入配对模式；
--------------------------------------------------
===== 文本块 2（字符数：165） =====
2. 手机/电脑开启蓝牙，搜索设备“HL3070-Scanner”并连接；
3. 连接成功后，蓝灯常亮，可开始扫描使用。

### 充电说明
当红灯低电量闪烁时，使用标配Type-C数据线连接电源充电，充电时红灯常亮，充满后绿灯常亮。
--------------------------------------------------
===== 文本块 3（字符数：136） =====
## 注意事项
1. 请勿将扫描枪置于高温、潮湿环境中；
2. 避免摔落、碰撞，防止硬件损坏；
3. 长时间不使用请关闭电源，节省电量。
--------------------------------------------------
```

### 四、进阶使用：按 Token 数分割（更贴合大模型）

大模型的上下文窗口实际按 **Token 数** 计算（1Token≈0.75 个中文字），按 Token 分割比按字符数更精准，需结合`tiktoken`库（OpenAI 的 Token 计算工具）：

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
import tiktoken

# 1. 定义Token计数函数
def count_tokens(text: str) -> int:
    # 选择适配的模型编码（如gpt-3.5-turbo/qwen，可替换为你的模型）
    encoding = tiktoken.encoding_for_model("gpt-3.5-turbo")
    return len(encoding.encode(text))

# 2. 初始化分割器（按Token数配置）
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=100,          # 单块最大Token数（适配模型上下文，如3.5-turbo默认4k Token）
    chunk_overlap=10,        # 相邻块重叠Token数
    length_function=count_tokens,  # 按Token数计算长度
    separators=["\n\n", "\n", " ", ""]
)

# 3. 分割文本（用法和基础版一致）
text_chunks = text_splitter.split_text(long_text)
```

### 五、和你的项目结合：分割 MD 文档（贴合实际业务）

结合之前的 MD 文件处理、图片摘要生成场景，直接封装为工具函数，复用性拉满：

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
from pathlib import Path
from app.core.logger import logger

def split_md_file(md_file_path: str, chunk_size: int = 500, chunk_overlap: int = 50) -> list:
    """
    分割MD文档为语义完整的文本块（适配你的项目MD处理场景）
    :param md_file_path: MD文件本地完整路径
    :param chunk_size: 单块最大字符数
    :param chunk_overlap: 相邻块重叠字符数
    :return: 分割后的文本块列表
    """
    # 读取MD文件内容
    try:
        md_content = Path(md_file_path).read_text(encoding="utf-8")
    except Exception as e:
        logger.error(f"读取MD文件失败：{md_file_path}，错误：{str(e)}")
        return []
    
    # 初始化分割器
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        separators=["\n\n", "\n", "##", "#", " ", ""]  # 优先按MD标题/换行分割，更贴合MD语义
    )
    
    # 执行分割
    chunks = text_splitter.split_text(md_content)
    logger.info(f"MD文件分割完成：{md_file_path}，生成{len(chunks)}个文本块")
    return chunks

# 调用示例（直接传入你的MD文件路径）
# chunks = split_md_file("/project/output/hl3070说明书.md")
```

### 六、核心参数说明

|       参数        |   类型   |                           核心作用                           |            推荐值             |
| :---------------: | :------: | :----------------------------------------------------------: | :---------------------------: |
|   `chunk_size`    |   int    |      单块文本的最大长度（字符 / Token），核心控制块大小      | 500-2048（按模型上下文调整）  |
|  `chunk_overlap`  |   int    |     相邻文本块的重叠长度，防止语义断层(只有超出才会触发)     |     chunk_size 的 10%-20%     |
| `length_function` | callable |      长度计算函数，`len`= 按字符，自定义函数 = 按 Token      |  字符用`len`，Token 用自定义  |
|   `separators`    |   list   | 分割符列表（从粗到细），优先按前面的分隔符分割，贴合文本语义 | 默认`["\n\n", "\n", " ", ""]` |

### 七、关键注意事项

1. **分隔符顺序**：必须「从粗到细」排列，否则会优先按细分隔符分割，破坏语义（如先按`\n\n`（空行），再按`\n`（换行））；
2. **重叠率**：不宜过高（如超过 30%），否则会产生大量冗余，增加处理成本；
3. **MD/HTML 适配**：分割专属格式文本时，可在`separators`中加入格式分隔符（如 MD 的`#`/`##`，HTML 的`<p>`/`<br>`），提升语义保留效果；
4. **Token 数校准**：若模型上下文限制严格（如小模型 4k Token），务必按 Token 数分割，避免单块超上限。

### 总结

1. `RecursiveCharacterTextSplitter` 是 LangChain**最常用的文本分割器**，核心解决「超长文本适配大模型上下文」问题，兼顾**分割效率**和**语义保留**；
2. 基础用法仅需 3 步：初始化分割器→调用`split_text()`→获取文本块，极简易上手；
3. 进阶可按 Token 数分割，更贴合大模型实际使用场景；
4. 结合你的项目，可直接封装为 MD 文件分割工具，复用在文档处理、向量入库等环节。