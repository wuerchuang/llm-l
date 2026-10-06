# 掌柜智库项目(RAG)实战

## 5. 导入数据节点实现与测试

### 5.2 PDF 转 Markdown (node_pdf_to_md)

**节点文件**: `app/process/import_/agent/nodes/node_pdf_to_md.py`
**相关工具类位置**: `app/shared/utils/task_utils.py`

#### 场景背景与能力目标

解决**非结构化 PDF 无法直接用于 RAG 知识库**的核心痛点：

- 普通 PDF 文本乱码、分段错乱、表格丢失、图片无法识别；
- 通过 **MinerU 企业级高精度解析**，将 PDF 一键转为 **结构完整、带图片、带表格、带排版** 的标准 Markdown 文档；
- 为后续**文本分块、向量入库、多模态理解**提供高质量数据源。

**为什么选择 MinerU?**

<img src="assets/image-20260422135823601.png" alt="image-20260422135823601" style="zoom:50%;" />

- **超高精度解析**：文字、标题、目录、表格、公式全还原
- **强 Layout 分析**：保留原文结构，不打乱段落
- **内置 OCR**：图片版 PDF、扫描件也能转文字
- **多模态输出**：自动提取图片并生成引用，支持图文一体知识库
- **云端异步解析**：不占本地 GPU / 内存，大文件不崩溃
- **企业级稳定**：适合工程化、批量入库、生产环境使用
- 官网地址: https://mineru.net/
- 文档地址: https://opendatalab.github.io/MinerU/zh/quick_start/

**实现思路**:

1.  **云端算力卸载**: 考虑到本地 OCR 和布局分析的资源消耗巨大，本节点采用“云端 API”模式，将繁重的解析任务卸载到 MinerU 服务端。
2.  **异步轮询机制**: 针对大文件解析耗时较长的问题，设计了“上传 -> 提交 -> 轮询 -> 下载”的异步交互流程，通过固定间隔轮询状态，避免长连接超时。
3.  **完整性保障**: 解析完成后，不仅提取 Markdown 文本，还同步下载并解压关联的图片资源，保持文档的多模态完整性。

#### 步骤分解

1.  **准备参数**: 获取 PDF 路径和输出目录。
2.  **请求上传**: 调用 MinerU 在线 API (`/file-urls/batch`) 获取上传链接。
3.  **上传文件**: 将 PDF 文件 PUT 到签名 URL。
4.  **轮询结果**: 循环查询任务状态 (`/extract-results/batch/{batch_id}`)，直到完成。
5.  **获取结果**: 下载生成的 ZIP 包，解压并读取 `.md` 文件内容到 state。

**注意**: 需要在 `.env` 中配置 `MINERU_API_TOKEN`。

API申请位置：https://mineru.net/apiManage/token （有效期90天）

参考官方文档：https://mineru.net/doc/docs/index.html?theme=light&v=1.0#%E6%89%B9%E9%87%8F%E6%96%87%E4%BB%B6%E8%A7%A3%E6%9E%90

#### 1. 节点与依赖定位

**节点文件**: `app/process/import_/agent/nodes/node_pdf_to_md.py`
**对应 service 文件**: `app/rag/import_/pdf_parse_service.py`
**第一次引入的 infra 文件**: `app/infra/document_parse/mineru_gateway.py`
**相关配置位置**: `app/rag/import_/config.py`

#### 2. 节点职责与 service 入口

**节点作用**: `node_pdf_to_md` 的核心职责是把 PDF 解析成后续可继续处理的 Markdown 内容。节点本身不承载上传、轮询、下载等细节，而是负责记录节点状态，并调用 `parse_pdf_to_markdown()` 完成 PDF 解析流程。

**对应 service 作用**: `parse_pdf_to_markdown()` 才是这一节真正的业务主函数。它会完成四件事：

1. 校验 PDF 路径和输出目录是否可用
2. 调用 MinerU 上传 PDF 并轮询解析结果
3. 下载解析结果 ZIP 并解压出 Markdown 文件
4. 把 `md_path` 和 `md_content` 回写到 `state`

**为什么这里先引入 MinerU gateway**

这也是当前项目里第一次正式用到 `infra/document_parse`。当前这一层只提供 `base_url` 和 `api_key` 两个配置出口；按照现有实现，`pdf_parse_service.py` 直接读取 `infra_config.mineru` 也可以完成同样的工作。

当前保留 `mineru_gateway` 的主要意义，不在于增加复杂度，而在于保留 MinerU 外部能力的统一出口，便于后续继续收口公共交互逻辑。

可以把它理解成一句话：

- `process` 层负责图节点调度
- `rag` 层负责业务流程
- `infra` 层负责外部能力出口

在这个节点里，外部能力出口就是 `mineru_gateway`。当前它主要统一读取 MinerU 的服务地址和 Token；后续如果需要继续收口公共逻辑，这一层也可以继续向外暴露方法。

#### 3. MinerU gateway 定位

**文件**: `app/infra/document_parse/mineru_gateway.py`

**为什么保留这个 gateway** 

1. **当前实现较轻**: 这一层只封装了 `base_url` 和 `api_key`，因此直接读取 `infra_config.mineru` 也能完成现有功能。
2. **保留统一出口**: 当前虽然主要暴露配置，但第三方文档解析能力仍然统一收口在 `infra`，后续补方法时不需要再改上层调用口径。
3. **便于补充方法**: 例如可以继续补 `build_headers()`、`get_upload_url()`、`poll_result()`、`check_config()` 这类通用方法，使 `rag` 层聚焦业务流程，减少底层交互细节。
4. **符合当前分层口径**: 第三方文档解析服务属于基础设施能力，放在 `infra` 会比散落在 `rag` 业务文件里更清楚。

```python
from app.infra.config.providers import infra_config

class MinerUGateway:
    @property
    def base_url(self) -> str:
        """
         获取 MinerU 服务基础地址。

         Returns:
           str: MinerU 接口基础 URL。
        """
        return infra_config.mineru.base_url

    @property
    def api_key(self) -> str:
        """
         获取 MinerU 服务 API Token。

         Returns:
            str: MinerU 调用所需的 Token。
        """
        return infra_config.mineru.api_key

mineru_gateway = MinerUGateway()

# dataclass版本
from dataclasses import dataclass
from app.infra.config.providers import infra_config

@dataclass(frozen=True)  # frozen=True 代表只读，更安全！
class MinerUGateway:
    # 直接声明属性 + 默认值从配置读取
    base_url: str = infra_config.mineru.base_url
    api_key: str = infra_config.mineru.api_key

# 用法和你原来一模一样！
mineru_gateway = MinerUGateway()
```

**这一层的定位**

- xxxxxxxxxx if __name__ == '__main__':    from app.shared.runtime.logger import logger    from app.process.import_.agent.state import create_default_state    # 单元测试：覆盖不支持类型、MD、PDF三种场景    logger.info("===== 开始node_entry节点单元测试 =====")​    # 测试1: 不支持的TXT文件    test_state1 = create_default_state(        task_id="test_task_001",        local_file_path="联想海豚用户手册.txt"    )    result_1 =  node_entry(test_state1)    print(f"第一次测试结果: \n {json.dumps(result_1, indent=4, ensure_ascii=False)}")    # 测试2: MD文件    test_state2 = create_default_state(        task_id="test_task_002",        local_file_path="小米用户手册.md"    )    result_2 = node_entry(test_state2)    print(f"第二次测试结果: \n {json.dumps(result_2, indent=4, ensure_ascii=False)}")    # 测试3: PDF文件    test_state3 = create_default_state(        task_id="test_task_003",        local_file_path="万用表的使用.pdf"    )    result_3 = node_entry(test_state3)​    print(f"第三次测试结果: \n {json.dumps(result_3, indent=4, ensure_ascii=False)}")​    logger.info("===== 结束node_entry节点单元测试 =====")python
- `mineru_gateway.api_key` 对外提供 MinerU 调用 Token
- 当前这层以统一配置出口为主
- 后续可以从“配置门面”继续扩展为“MinerU 交互门面”

#### 4. 环境变量与解析参数

MinerU 的真实连接信息仍然来自 `.env`，而解析过程中的轮询间隔、超时时间、模型版本等策略，则统一放在 `rag/import_/config.py`。

.env

```ini
MINERU_API_TOKEN=你的_mineru_token
MINERU_BASE_URL=https://mineru.net/api/v4
```

config.py

```python
# MinerU 模型版本配置（vlm = 视觉语言模型，适合PDF/图片高精度解析）
MINERU_MODEL_VERSION = "vlm"

# MinerU 任务轮询最大超时时间（单位：秒），超过则判定任务失败
# 600 -> 一个pdf 约等于 1秒
MINERU_POLL_TIMEOUT_SECONDS = 600

# MinerU 任务轮询间隔时间（单位：秒），每隔多久查询一次任务状态
MINERU_POLL_INTERVAL_SECONDS = 3

# MinerU 文件下载超时时间（单位：秒），下载文件超过此时长则中断
MINERU_DOWNLOAD_TIMEOUT_SECONDS = 30
```

这样设计的目的有三点：

- 连接配置和业务策略分开
- 敏感信息继续放 `.env`
- 解析节奏参数可以集中维护，后期调优更方便

#### 5. 业务步骤分析

这一节涉及 5 个核心函数，主链路如下：

`node_pdf_to_md -> parse_pdf_to_markdown -> validate_pdf_paths / upload_pdf_and_poll / download_and_extract_markdown`

##### 5.1 `node_pdf_to_md`

**函数签名**: `node_pdf_to_md(state: ImportGraphState) -> ImportGraphState`

**步骤**

1. 记录当前节点开始执行
2. 调用 `parse_pdf_to_markdown(state)` 执行 PDF 解析主流程
3. 记录当前节点执行完成
4. 返回更新后的 `state`

##### 5.2 `parse_pdf_to_markdown`

**函数签名**: `parse_pdf_to_markdown(state: dict) -> dict`

**步骤**

1. 调用 `validate_pdf_paths()` 校验 PDF 路径和输出目录
2. 调用 `upload_pdf_and_poll()` 上传 PDF 并轮询解析结果
3. 调用 `download_and_extract_markdown()` 下载 ZIP 并提取 Markdown
4. 回写 `state["md_path"]`
5. 读取 Markdown 全文并回写 `state["md_content"]`
6. 返回最新状态

##### 5.3 `validate_pdf_paths`

**函数签名**: `validate_pdf_paths(state: dict) -> tuple[Path, Path]`

**步骤**

1. 读取 `pdf_path` 和 `local_dir`
2. 校验 `pdf_path` 是否为空
3. 若 `local_dir` 为空，则写入默认输出目录
4. 转换为 `Path` 对象
5. 校验 PDF 文件是否真实存在
6. 若输出目录不存在，则自动创建
7. 返回 `pdf_path_obj` 与 `local_dir_obj`

##### 5.4 `upload_pdf_and_poll`

**函数签名**: `upload_pdf_and_poll(pdf_path_obj: Path) -> str`

**步骤**

1. 校验 MinerU 配置是否完整
2. 调用 `/file-urls/batch` 申请上传地址与 `batch_id`
3. 使用 `Session(trust_env=False)` 上传 PDF 文件
4. 根据 `batch_id` 轮询任务状态
5. 若任务成功，返回 `full_zip_url`
6. 若任务失败或超时，抛出异常

##### 5.5 `download_and_extract_markdown`

**函数签名**: `download_and_extract_markdown(zip_url: str, local_dir_path_obj: Path, stem: str) -> Path`

**步骤**

1. 下载 MinerU 返回的 ZIP 结果包
2. 将 ZIP 保存到输出目录
3. 清理旧解压目录并重新解压
4. 在解压目录中递归查找 `.md` 文件
5. 优先选择与原 PDF 同名的 Markdown 文件
6. 若没有同名文件，则退化选择 `full.md` 或第一个 Markdown 文件
7. 统一重命名为 `{stem}.md` 并返回路径

#### 6. 节点代码实现

```python
import os

from app.shared.runtime.logger import node_log
from app.shared.utils.task_utils import add_done_task, add_running_task
from app.process.import_.agent.state import ImportGraphState , create_default_state
from app.rag.import_.pdf_parse_service import parse_pdf_to_markdown
from app.shared.runtime.logger import logger , PROJECT_ROOT

@node_log("node_pdf_to_md")
def node_pdf_to_md(state: ImportGraphState) -> ImportGraphState:
    """
    节点: PDF转Markdown (node_pdf_to_md)
    为什么叫这个名字: 核心任务是将 PDF 非结构化数据转换为 Markdown 结构化数据。
    """
    add_running_task(state["task_id"], "node_pdf_to_md")
    state = parse_pdf_to_markdown(state)
    add_done_task(state["task_id"], "node_pdf_to_md")
    return state
```

#### 7. service 总入口

`parse_pdf_to_markdown()` 是这一节的 service 总入口。它不直接把所有细节堆在一个函数里，而是负责串联 3 个子函数，并在最后完成状态回写。

```python
import shutil
import time
import requests

from app.infra.document_parse.mineru_gateway import mineru_gateway
from app.process.import_.agent.state import ImportGraphState, create_default_state
from app.rag.import_.config import MINERU_MODEL_VERSION, MINERU_POLL_TIMEOUT_SECONDS, MINERU_POLL_INTERVAL_SECONDS, \
    MINERU_DOWNLOAD_TIMEOUT_SECONDS
from app.shared.runtime.logger import step_log,logger,PROJECT_ROOT
from pathlib import Path

@step_log("parse_pdf_to_markdown")
def parse_pdf_to_markdown(state: ImportGraphState) -> ImportGraphState:
    """
    PDF 解析服务：
    1. 调用 MinerU
    2. 下载并解压解析结果
    3. 获取 Markdown 路径和正文内容
    4. 回写 md_path / md_content / local_dir
    """
    # 先校验 PDF 路径和输出目录，避免把非法输入送进解析服务。
    pdf_path_obj, local_dir_path_obj = validate_pdf_paths(state)
    # 上传 PDF 到 MinerU，并轮询直到服务端返回最终压缩包地址。
    zip_url = upload_pdf_and_poll(pdf_path_obj)
    logger.info(f"minerU返回的zip地址:{zip_url}")
    # 下载结果包并提取最终 Markdown 文件。
    md_path_obj = download_and_extract_markdown(zip_url, local_dir_path_obj, pdf_path_obj.stem)
    state["md_path"] = str(md_path_obj)
    state["md_content"] = md_path_obj.read_text(encoding="utf-8")
    return state
```

#### 8. 子函数 1：路径校验

##### 8.1 函数说明：`validate_pdf_paths`

```python
@step_log("validate_pdf_paths")
def validate_pdf_paths(state: dict) -> tuple[Path, Path]:
    """
    校验PDF文件路径与输出目录，确保文件存在、目录可用并自动补全目录
    :param state: 流程状态字典，包含 pdf_path、local_dir 字段
    :return: 元组(解析后的PDF路径对象, 本地输出目录对象)
    :raises ValueError: pdf_path 为空时抛出异常
    :raises FileNotFoundError: PDF文件不存在时抛出异常
    """
    # 从状态中读取PDF文件路径和本地输出目录
    pdf_path = state.get("pdf_path")
    local_dir = state.get("local_dir")

    # 校验PDF路径是否为空，为空则终止流程并抛出异常
    if not pdf_path:
        logger.error("pdf_path的参数值为空,无法读取文件!")
        raise ValueError("pdf_path的参数值为空,无法读取文件!")

    # 输出目录为空时，使用项目根目录下的output作为默认目录，并回写到状态中
    if not local_dir:
        logger.warning("没有传入local_dir地址,给与默认值!")
        local_dir = PROJECT_ROOT / "output"
        state["local_dir"] = str(local_dir)

    # 转为Path对象，方便后续路径操作
    pdf_path_obj = Path(pdf_path)
    local_dir_obj = Path(local_dir)

    # 校验PDF文件是否真实存在，不存在则抛出异常
    if not pdf_path_obj.exists():
        logger.error(f"pdf_path:{pdf_path_obj},但是没有文件存在!")
        raise FileNotFoundError(f"pdf_path:{pdf_path_obj},但是没有文件存在!")

    # 输出目录不存在则自动创建（支持多级目录，已存在也不报错）
    if not local_dir_obj.exists():
        logger.warning(f"local_dir:{local_dir_obj}地址没有文件夹,我们需要主动创建!")
        local_dir_obj.mkdir(parents=True, exist_ok=True)

    # 返回路径对象供后续业务使用
    return pdf_path_obj, local_dir_obj
```

#### 9. 子函数 2：上传与轮询

##### 9.1 `requests` 基础示例

`requests` 是 Python 中最常用的 HTTP 请求库之一，常见请求方法有 `get()`、`post()`、`put()`。这些函数最常见的参数有 `url`、`headers`、`params`、`json`、`data`、`timeout`：`url` 表示请求地址，`headers` 表示请求头，`params` 用于拼接查询参数，`json` 用于提交 JSON 请求体，`data` 用于提交原始表单或字节内容，`timeout` 用于限制请求等待时间。

请求发送后，会得到一个 `response` 响应对象。常见读取方式有 `response.status_code`、`response.text`、`response.json()`、`response.content`：`status_code` 用来判断 HTTP 状态，`text` 用来读取文本响应，`json()` 用来把 JSON 响应解析成 Python 字典，`content` 用来读取二进制响应内容。

下面这个示例只演示 `requests` 最基础的请求发送与响应接收写法：

```python
# 导入网络请求库
import requests
# 接口请求地址
url = "https://api.example.com/demo"

# 请求头：存放认证信息、内容类型等通用配置
headers = {
    "Content-Type": "application/json",  # 声明请求体为JSON格式
    "Authorization": "Bearer your_token", # 身份令牌，接口鉴权使用
}
# URL查询参数：拼接在请求地址 ? 后面的参数
# ?page=1&size=10 -> params
# 请求体 {json}  -> payload 
params = {
    "page": 1,   # 页码
    "size": 10,  # 每页数据条数
}
# 请求体(Body)：POST接口传递的JSON业务参数
payload = {
    "name": "demo",
    "status": "active",
}

# 发起POST请求
# url: 接口地址
# headers: 请求头
# params: 地址栏查询参数
# json: 自动将字典转为JSON字符串放入请求体，并附带对应Content-Type
# timeout: 请求超时时间(单位：秒)，超时直接抛出异常
response = requests.post(
    url,
    headers=headers,
    params=params,
    json=payload, #请求体的json字符串  dict / baseModel 
    timeout=30,   # 超时时间
    data=字节数据 例如文件  # data -> 请求体 字节输出 上传文件
)

response = requests.get(
    url,
    headers=headers,
    params=params,
    timeout=30,   # 超时时间
)

# http协议 超文本传输协议 客户端和服务端网络通信 
#         协议: 规定: 规定客户端和服务端传递数据时序,数据格式等等
# url概念  指向服务器中的资源或者接口
# http请求数据包和响应数据包
  """
     请求:  请求行 请求方式 请求地址 协议版本
           请求头  key=value 
           请求空行 /n
           请求体  请求体 流 字符串 字节
     响应:  状态行 状态码 协议版本
            响应头
            响应空行
            响应体
  """
# http请求方式  get post put  delete  header option... 11 12种
  # get  查询 获取数据
  # delete 删除 删除数据     请求行 请求头   params 
				# 请求方式最终影响的请求数据包格式 跟响应没关系
  # post 保存 上传          请求行 请求头 空行 请求体
  # put  置换 更新                        params  json data 
# http响应状态码
  # 1xx  2xx 3xx 4xx 5xx -> 后端服务 agent 
  # 1xx 中继 请求还没有结束 中... 
  # 2xx 成功 200 202 成功 
  # 3xx 重定向 我们间接成功...
  # 4xx 客户端 400 int str  404 地址异常  405  get / post 
  # 5xx 服务器内部异常 我们的异常没有进行捕捉 抛给前端 [好好好说]  
==========================================
# 1. 获取HTTP响应状态码（200成功、4xx客户端错误、5xx服务端错误）
print(response.status_code)  # 状态
# 2. 获取响应原始文本内容（字符串格式）
print(response.text)  # @property text()  响应的字符串
# 3. 字节数据（二进制）：下载文件、图片、压缩包时必须用这个
# 这就是你要加的 content
byte_data = response.content # @property content()  获取返回的字节数据 (下载zipurl)
print("响应字节数据长度:", len(byte_data))  # 字节大小
# print("原始字节:", byte_data)
# 4. 将响应的JSON文本解析为Python字典/列表（接口返回非JSON会报错）
result_dict = response.json() # 将返回的json数据转成python中的字典
print(result_dict)
```

状态码可以先记住最常见的几类：`200` 表示请求成功，`201` 表示资源创建成功，`400` 表示请求参数有问题，`401` 表示未认证，`403` 表示无权限，`404` 表示地址不存在，`500 ~ 599` 表示服务端异常。在实际项目中，通常先判断 `status_code`，再继续解析响应体。

##### 9.2 函数说明：`upload_pdf_and_poll`

```python
@step_log("upload_pdf_and_poll")
def upload_pdf_and_poll(pdf_path_obj: Path) -> str:
    """
    上传PDF文件到MinerU服务，并轮询等待解析完成，最终返回解析结果的下载地址
    :param pdf_path_obj: 本地PDF文件路径对象
    :return: 解析完成后的ZIP压缩包下载地址
    """

    # 1. 校验MinerU服务配置（base_url和api_key必须存在）
    if not mineru_gateway.base_url or not mineru_gateway.api_key:
        logger.error("minerU配置错误,请检查minerU配置!")
        raise ValueError("minerU配置错误,请检查minerU配置!")

    # 2. 构造请求地址和请求头
    url = f"{mineru_gateway.base_url}/file-urls/batch"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {mineru_gateway.api_key}",
    }

    # 3. 构造请求参数：文件名 + 使用的模型版本
    payload = {
        "files": [{"name": pdf_path_obj.stem}],
        "model_version": MINERU_MODEL_VERSION
    }

    # 4. 请求MinerU获取文件预上传地址
    response = requests.post(url, headers=headers, json=payload)
    if response.status_code != 200:
        raise RuntimeError(f"申请上传地址失败,返回状态码为:{response.status_code},请检查minerU配置!")

    # 5. 解析返回结果，校验业务状态码
    result_dict = response.json()
    if result_dict["code"] != 0:
        raise RuntimeError(
            f"申请地址网络状态成功!但是业务失败!错误码:{result_dict['code']},失败信息:{result_dict['msg']}"
        )

    # 6. 提取上传URL和任务批次ID
    file_upload_url = result_dict["data"]["file_urls"][0]
    batch_id = result_dict["data"]["batch_id"]

    # 7. 上传PDF文件到MinerU的存储地址
    with requests.Session() as session:
        # session.trust_env = False 是告诉 requests ： 不要信任系统环境变量里的代理、证书、认证等配置 。
        # session.trust_env = False 的意思就是：
        # - 别用系统里自动带来的代理、证书、账号配置
		# - 只按我这个代码里写的内容发请求
        # 上传的地址是, 预签名上传地址 , OSS / S3 / 对象存储直传地址 非常脆弱
        session.trust_env = False
        upload_response = session.put(file_upload_url, data=pdf_path_obj.read_bytes())
        if upload_response.status_code != 200:
            raise RuntimeError(f"上传文件失败,返回状态码为:{upload_response.status_code},请检查minerU配置!")

    # 8. 构造轮询查询地址
    poll_url = f"{mineru_gateway.base_url}/extract-results/batch/{batch_id}"
    timeout = MINERU_POLL_TIMEOUT_SECONDS
    interval_time = MINERU_POLL_INTERVAL_SECONDS
    start_time = time.time()

    # 9. 开始轮询任务解析状态
    while True:
        # 超时判断
        if time.time() - start_time > timeout:
            raise TimeoutError("轮询超时,请检查minerU配置!")

        try:
            # 请求轮询接口
            poll_response = requests.get(poll_url, headers=headers)
        except Exception:
            logger.warning("请求出现异常!可以稍后重试!!")
            time.sleep(interval_time)
            continue

        # 网络异常处理：5xx可重试，其他直接报错
        if poll_response.status_code != 200:
            if 500 <= poll_response.status_code < 600:
                logger.warning(f"可有修复的网络异常,状态码为:{poll_response.status_code}")
                time.sleep(interval_time)
                continue
            raise RuntimeError(f"不可修复的网络状态异常,状态码为:{poll_response.status_code}")

        # 解析返回结果
        poll_response_dict = poll_response.json()
        if poll_response_dict["code"] != 0:
            raise RuntimeError(
                f"轮询业务异常,错误码:{poll_response_dict['code']},失败信息:{poll_response_dict['msg']}"
            )

        # 10. 获取解析任务状态
        extract_result = poll_response_dict["data"]["extract_result"][0]
        extract_result_state = extract_result["state"]

        # 任务完成 → 返回下载地址
        if extract_result_state == "done":
            extract_result_url = extract_result["full_zip_url"]
            if not extract_result_url:
                raise RuntimeError("已经完成了解析,但是zip地址为空!!")
            return extract_result_url

        # 任务失败 → 抛出异常
        if extract_result_state == "failed":
            raise RuntimeError(f"已经完成了解析,但是失败了!!失败信息:{extract_result['err_msg']}")

        # 任务仍在处理中 → 等待后继续轮询
        logger.warning(f"解析正在进行中,状态:{extract_result_state}!")
        time.sleep(interval_time)
```

这一层有三个关键点：

1. **先拿上传地址，再传文件**: 不是直接把 PDF 传给业务接口，而是先通过 MinerU 申请预签名上传地址。
2. **PUT 上传要用 `Session(trust_env=False)`**: 这样可以绕开系统代理，避免预签名 URL 被污染导致上传失败。
3. **轮询参数不写死在函数里**: `timeout`、`interval_time`、`model_version` 全部走配置常量，后期调优更方便。

总结：为什么 PUT 这里需要 `Session(trust_env=False)`？

1. `POST / GET` 一般是普通业务接口：
   它们主要校验地址、参数和认证信息，通常不会像预签名上传那样对请求细节做严格签名校验。
2. `PUT` 这里对应的是预签名 URL 上传：
   它使用的是对象存储直传地址，签名校验更严格。
   
   - 如果请求经过系统代理，代理可能会追加或改写部分请求头，导致签名校验失败。
   - 设置 `Session(trust_env=False)` 后，可以绕过系统代理环境变量，尽量保持上传请求原样发送。

#### 10. 子函数 3：下载与解压

##### 10.1 函数说明：`download_and_extract_markdown`

```python
@step_log("download_and_extract_markdown")
def download_and_extract_markdown(zip_url: str, local_dir_path_obj: Path, stem: str) -> Path:
    """
    下载 MinerU 解析完成的 ZIP 压缩包，解压并提取出标准的 MD 文件
    1. 从 zip_url 下载解析结果压缩包
    2. 解压到指定目录
    3. 自动查找最合适的 MD 文件（优先同名 → full.md → 第一个）
    4. 重命名为统一规范的文件名并返回

    Args:
        zip_url: MinerU 返回的 ZIP 下载地址
        local_dir_path_obj: 本地存放解压文件的目录
        stem: 原始 PDF 的文件名（不带后缀，用于重命名 MD）

    Returns:
        Path: 最终整理好的 MD 文件路径对象
    """
    # ---------------------- 1. 下载 ZIP 压缩包 ----------------------
    # 发送请求下载解析好的 ZIP 文件
    response = requests.get(zip_url, timeout=MINERU_DOWNLOAD_TIMEOUT_SECONDS)
    # 拼接 ZIP 保存路径：输出目录 + 文件名_result.zip
    zip_path_obj = local_dir_path_obj / f"{stem}_result.zip"
    # 将二进制内容写入 ZIP 文件
    zip_path_obj.write_bytes(response.content)

    # ---------------------- 2. 解压 ZIP 文件 ----------------------
    # 解压目录 = 输出目录 / PDF 文件名（无后缀）
    extract_path_obj = local_dir_path_obj / stem
    # 如果解压目录已存在，先删除（防止旧文件干扰）
    if extract_path_obj.exists():
        shutil.rmtree(extract_path_obj)
    # 创建新的解压目录
    extract_path_obj.mkdir(parents=True, exist_ok=True)
    # 解压 ZIP 包到目标目录
    shutil.unpack_archive(zip_path_obj, extract_path_obj)

    # ---------------------- 3. 查找所有 MD 文件 ----------------------
    # 递归查找解压目录下所有 .md 文件
    md_file_list = list(extract_path_obj.rglob("*.md"))
    # 没有找到 MD 文件则抛出异常
    if not md_file_list:
        raise FileNotFoundError(f"文件解压失败,在:{extract_path_obj}没有任何md文件!")

    # ---------------------- 4. 按优先级选择 MD 文件 ----------------------
    # 优先级 1：找和 PDF 同名的 MD（最标准）
    for md_file in md_file_list:
        if md_file.stem == stem:
            return md_file

    # 优先级 2：找不到同名，找 full.md（MinerU 默认完整导出文件）
    target_md_obj = None
    for md_file in md_file_list:
        if md_file.name.lower() == "full.md":
            target_md_obj = md_file
            break

    # 优先级 3：还找不到，直接取第一个 MD
    if not target_md_obj:
        target_md_obj = md_file_list[0]

    # ---------------------- 5. 重命名为统一规范名称 ----------------------
    # 将选中的 MD 重命名为 {stem}.md（和 PDF 同名）
    return target_md_obj.rename(target_md_obj.with_name(f"{stem}.md"))
```

#### 11. 关键语法补充和说明

##### 11.1 `Path` 在这里的作用

`Path` 对象用于统一路径处理、文件读写和目录操作。与字符串路径相比，主要有三个优势：

1. 路径拼接更直观，支持 `/` 运算符
2. 文件属性获取更直接，例如 `name`、`stem`、`suffix`
3. 路径方法更集中，例如 `exists()`、`mkdir()`、`read_text()`

```python
from pathlib import Path

p = Path("D:/test/report_v1.2.pdf")
print(p.name)
print(p.stem)
print(p.suffix)
print(p.parent)


shutil.copytree("src_dir", "dst_dir")  # 目标必须不存在

# shutil.make_archive( 压缩包名字, 格式, 要打包的文件夹 )
shutil.make_archive("backup", "zip", "folder")
# 三个参数终极解释：
# 第1个参数 "backup" → 压缩包**名字**（不用写.zip）
# 第2个参数 "zip"    → 压缩**格式**：支持 zip / tar / gztar / bztar 等
# 第3个参数 "folder" → 要**打包哪个文件夹**（把这个文件夹整个压进去）
```

##### 11.2 关键函数说明

| 语法 / 函数 | 具体作用 | 执行示例 |
| :--- | :--- | :--- |
| `requests.Session()` | 创建可复用会话对象 | 用于 `PUT` 上传 PDF |
| `session.trust_env = False` | 禁用系统代理环境变量 | 避免预签名上传地址被代理污染 |
| `Path.read_bytes()` | 读取 PDF 二进制内容 | 上传文件时直接传 bytes |
| `Path.read_text(encoding="utf-8")` | 读取 Markdown 全文 | 回写 `md_content` |
| `shutil.unpack_archive()` | 解压压缩包 | 把 MinerU 返回的 ZIP 解压到目标目录 |
| `Path.rglob("*.md")` | 递归查找所有 Markdown 文件 | 兼容 MinerU 结果包内部多层目录 |
| `@node_log` / `@step_log` | 记录节点级 / 步骤级日志 | 方便授课演示和排错 |

#### 12. 测试优化

这一节的测试建议拆成两类：

1. **节点联调测试**: 验证 `node_pdf_to_md -> parse_pdf_to_markdown` 整条链路是否打通
2. **service 级测试**: 分别验证路径校验、外部调用、结果回写的关键逻辑

这样更贴合真实项目的排查方式，也便于分层验证。

##### 12.1 节点联调测试

```python
if __name__ == "__main__":
    logger.info("===== 开始 node_pdf_to_md 节点联调测试 =====")

    test_pdf_path = os.path.join(PROJECT_ROOT, "doc", "hak180使用说明书.pdf")
    test_state = create_default_state(
        task_id="test_pdf2md_task_001",
        pdf_path=test_pdf_path,
        local_dir=os.path.join(PROJECT_ROOT, "output"),
    )

    result = node_pdf_to_md(test_state)
    logger.info(f"md_path: {result['md_path']}")
    logger.info(f"md_content长度: {len(result['md_content'])}")
    logger.info("===== 结束 node_pdf_to_md 节点联调测试 =====")
```

**适用场景**

- 本地已经配置好 `MINERU_API_TOKEN`
- `doc` 目录下确实有测试 PDF
- 想验证“节点 + service + MinerU”整条链路都正常

