# 掌柜智库项目(RAG)实战

## 6. 导入web服务集成和测试

本章节介绍如何将上述实现的 LangGraph 知识库导入流程集成到 FastAPI Web 服务中，并提供可视化界面供用户上传文件和查看导入进度。

### 6.1 FastAPI 快速入门

FastAPI 是一个现代、高性能的 Python Web 框架。本节仅介绍本项目用到的核心功能。

#### 1. 为什么选择 FastAPI？

- **极速性能**：基于 Starlette 和 Pydantic，性能名列前茅
- **开发快**：简捷语法减少约 40% 代码量
- **原生异步**：完美支持 `async/await`
- **自动文档**：启动后访问 `/docs` 即可获得 Swagger UI

#### 2. 安装与启动

```bash
uv add fastapi "uvicorn[standard]" python-multipart
```

创建 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}
```

启动服务：

```bash
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host=settings.app_host, port=settings.import_app_port)
```

访问 http://127.0.0.1:8000/docs 查看自动生成的接口文档。

#### 3. 本项目用到的核心特性

##### (1) 路径参数与查询参数

**路径参数**：URL 中的一部分，用 `{}` 包裹。

**查询参数**：URL `?` 后面的键值对，如 `?q=value&limit=10`。

```python
from fastapi import FastAPI

app = FastAPI()

# 访问 http://127.0.0.1:8000/items/5?q=手机
@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    """
    item_id: 路径参数（必填），自动转为 int 类型
    q: 查询参数（可选），默认为 None
    """
    return {"item_id": item_id, "q": q}
```

**测试方法**：
- 浏览器访问：`http://127.0.0.1:8000/items/5?q=手机`
- 返回结果：`{"item_id": 5, "q": "手机"}`

**关键点**：
- FastAPI 会自动进行类型转换和验证
- 如果传入 `item_id=abc`（非数字），会自动返回 422 错误

##### (2) 请求体与数据验证

当客户端发送 POST 请求时，通常在请求体（Body）中传递 JSON 数据。

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 定义数据模型（类似数据库表结构）
class Item(BaseModel):
    name: str          # 必填，字符串类型
    price: float       # 必填，浮点数类型
    description: str | None = None  # 可选，默认为 None

# POST 请求接收 JSON 数据
@app.post("/items/")
def create_item(item: Item):
    """
    FastAPI 会自动：
    1. 解析请求体中的 JSON 数据
    2. 验证数据类型是否正确
    3. 如果验证失败，返回详细的错误信息
    """
    return {
        "name": item.name,
        "price": item.price,
        "description": item.description
    }
```

**测试方法**（使用 Swagger UI）：
1. 访问 `http://127.0.0.1:8000/docs`
2. 找到 `POST /items/` 接口，点击 "Try it out"
3. 在请求体中输入：
   ```json
   {
     "name": "iPhone 15",
     "price": 5999.0,
     "description": "最新款苹果手机"
   }
   ```
4. 点击 "Execute"，查看响应结果

**验证示例**：
- ✅ 正确：`{"name": "iPhone", "price": 5999.0}` → 返回成功
- ❌ 错误：`{"name": "iPhone", "price": "abc"}` → 返回 422 错误，提示 "price 必须是浮点数"

##### (3) 文件上传

文件上传是本项目最常用的功能，用于接收用户上传的 PDF/MD 文件。

```python
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    """
    file: UploadFile 对象，包含以下属性：
      - file.filename: 文件名（如 "document.pdf"）
      - file.content_type: MIME 类型（如 "application/pdf"）
      - file.file: 文件内容（异步文件对象）
    
    File(...): 表示该参数为必填项
    """
    # 读取文件内容（异步操作，需要用 await）
    content = await file.read()
    
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(content)
    }
```

**测试方法**（使用 Swagger UI）：
1. 访问 `http://127.0.0.1:8000/docs`
2. 找到 `POST /upload` 接口，点击 "Try it out"
3. 点击 "Choose File"，选择一个 PDF 文件
4. 点击 "Execute"，查看响应结果

**关键点**：
- `await file.read()`: 异步读取文件内容，必须用 `await`
- `File(...)`: `...` 表示必填项，不传文件会返回 422 错误
- `File(None)`: `None` 表示可选项
- 大文件建议分块读取，避免内存溢出：
  ```python
  CHUNK_SIZE = 1024 * 1024  # 1MB
  while True:
      chunk = await file.read(CHUNK_SIZE)
      if not chunk:
          break
      # 处理每个分块
  ```

##### (4) 后台任务

后台任务用于在返回 HTTP 响应后，继续执行耗时操作（如发送邮件、处理数据、执行 LangGraph 流程）。

**为什么需要后台任务？**
- 如果不使用后台任务，用户需要等待所有操作完成后才能收到响应
- 使用后台任务，可以立即返回响应，耗时操作在后台异步执行

```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

# 定义一个耗时任务（普通函数即可）
def process_data(task_id: str, data: str):
    """
    模拟耗时操作（如处理文件、调用 AI 模型等）
    这个函数会在后台异步执行，不阻塞 HTTP 响应
    """
    print(f"[{task_id}] 开始处理数据: {data}")
    time.sleep(5)  # 模拟耗时 5 秒
    print(f"[{task_id}] 数据处理完成")
    # 可以在这里更新数据库、写入文件等

@app.post("/start-task")
async def start_task(background_tasks: BackgroundTasks):
    """
    background_tasks: FastAPI 提供的后台任务管理器
    add_task(): 将任务加入后台队列
    """
    task_id = "task-001"
    
    # 将耗时任务加入后台队列
    # 注意：这里只是"注册"任务，不会立即执行
    background_tasks.add_task(process_data, task_id, "测试数据")
    
    # 立即返回响应，不需要等待 process_data 执行完毕
    return {
        "task_id": task_id,
        "message": "Task started in background"
    }
```

**执行流程**：
1. 客户端发送 POST 请求到 `/start-task`
2. FastAPI 将 `process_data` 加入后台任务队列
3. **立即返回** `{"task_id": "task-001", ...}` 给客户端
4. 后台异步执行 `process_data("task-001", "测试数据")`
5. 5 秒后，`process_data` 执行完成（客户端已收到响应）

**本项目的应用**：
```python
# 在 /upload 接口中，启动 LangGraph 后台任务
background_tasks.add_task(
    run_graph_task,      # 后台执行的函数
    task_id,             # 参数1：任务ID
    str(task_local_dir), # 参数2：本地目录
    str(local_file_abs_path)  # 参数3：文件路径
)
```

**关键点**：
- 后台任务在返回响应**之后**执行
- 适合耗时操作，避免用户长时间等待
- 任务执行状态需要通过其他方式（如数据库、内存字典）追踪

##### (5) 跨域配置 (CORS)

**什么是跨域？**
- 前端页面运行在 `http://localhost:3000`
- 后端 API 运行在 `http://localhost:8000`
- 浏览器出于安全考虑，默认禁止不同端口的页面互相访问

**解决方案**：在后端配置 CORS（Cross-Origin Resource Sharing）

```python
from fastapi import FastAPI
from starlette.middleware.cors import CORSMiddleware

app = FastAPI()

# 添加 CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],           # 允许所有来源（生产环境建议指定具体域名）
    allow_credentials=True,         # 允许携带凭证（如 Cookie）
    allow_methods=["*"],            # 允许所有 HTTP 方法（GET、POST、PUT、DELETE 等）
    allow_headers=["*"],            # 允许所有请求头
)

@app.get("/api/data")
def get_data():
    return {"message": "Hello from backend"}
```

**配置说明**：
- `allow_origins`: 允许访问的前端域名列表
  - `["*"]`: 允许所有域名（开发环境常用）
  - `["http://localhost:3000"]`: 只允许特定域名（生产环境推荐）
- `allow_credentials`: 是否允许携带 Cookie 等凭证
- `allow_methods`: 允许的 HTTP 方法
- `allow_headers`: 允许的请求头

**本项目的应用**：
```python
# 从配置文件读取允许的域名
app.add_middleware(
    CORSMiddleware,
    allow_origins=list(settings.cors_origins) or ["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**测试方法**：
1. 创建一个 HTML 文件，运行在 `http://localhost:3000`
2. 在 HTML 中用 JavaScript 请求 `http://localhost:8000/api/data`
3. 如果没有配置 CORS，浏览器会报错
4. 配置 CORS 后，请求成功

##### (6) 常用响应类型

| 响应类型 | 用途 | 示例 |
|---------|------|------|
| `JSONResponse` | 返回 JSON 数据（默认） | `return {"code": 200}` |
| `FileResponse` | 返回文件 | `return FileResponse(path="file.pdf")` |
| `HTMLResponse` | 返回 HTML 字符串 | `return HTMLResponse(content="<html>...")` |
| `PlainTextResponse` | 返回纯文本 | `return PlainTextResponse(content="OK")` |
| `RedirectResponse` | 重定向 | `return RedirectResponse(url="/new-path")` |

更多细节请参考 [FastAPI 官方文档](https://fastapi.tiangolo.com/)。附简单示例：

1. JSONResponse（最常用）

   - **作用**：返回 JSON 格式的数据（FastAPI 默认的响应类型）；

   - **场景**：接口返回普通数据（列表、字典、状态信息等）；

   - 示例

     ```python
     from fastapi.responses import JSONResponse
     
     @app.get("/api/user")
     def get_user():
         # 等价于直接 return {"name": "张三", "age": 20}（FastAPI 自动转 JSONResponse）
         return JSONResponse(
             content={"name": "张三", "age": 20},
             status_code=200,  # 可选，默认 200
             headers={"X-Custom-Header": "custom-value"}  # 可选，自定义响应头
         )
     ```

2. FileResponse（文件专用）

   - **作用**：返回文件（支持大文件、静态文件、下载文件）；

   - **场景**：返回 HTML / 图片 / 视频 / Excel 等文件；

   - 关键参数

     - `path`：文件路径（必填）；
     - `filename`：下载时显示的文件名（可选）；
     - `media_type`：手动指定 MIME 类型（比如 `media_type="application/pdf"`）；

   - 示例:

     ```python
     from fastapi.responses import FileResponse
     
     @app.get("/download/excel")
     def download_excel():
         excel_path = "./data/report.xlsx"
         # 返回文件并指定下载文件名
         return FileResponse(
             path=excel_path,
             filename="月度报表.xlsx",
             media_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
         )
     ```

3. HTMLResponse

   - **作用**：返回 HTML 字符串（直接渲染页面）；

   - **场景**：动态生成 HTML 内容（比如拼接变量到 HTML 中）；

   - 示例:

     ```python
     from fastapi.responses import HTMLResponse
     
     @app.get("/hello")
     def hello(name: str = "游客"):
         html_content = f"""
         <html>
             <body>
                 <h1>你好，{name}！</h1>
             </body>
         </html>
         """
         return HTMLResponse(content=html_content, status_code=200)
     ```

     > 注意：如果是返回静态 HTML 文件，优先用 `FileResponse`；动态生成 HTML 用 `HTMLResponse`。

4. PlainTextResponse

   - **作用**：返回纯文本格式的数据（非 JSON、非 HTML）；

   - **场景**：返回简单的文本提示、日志内容等；

   - 示例：

     ```python
     from fastapi.responses import PlainTextResponse
     
     @app.get("/text")
     def get_text():
         return PlainTextResponse(content="这是纯文本响应", status_code=200)
     ```

5. RedirectResponse

   - **作用**：实现页面重定向；

   - **场景**：登录成功后跳转到首页、旧接口重定向到新接口等；

   - 示例:

     ```python
     from fastapi.responses import RedirectResponse
     
     @app.get("/old-path")
     def redirect_old_path():
         # 重定向到 /new-path，状态码 307 表示临时重定向
         return RedirectResponse(url="/new-path", status_code=307)
     
     @app.get("/new-path")
     def new_path():
         return {"message": "这是新接口"}
     ```

6. StreamingResponse（流式响应）

   - **作用**：返回流式数据（逐块传输，不一次性加载到内存）；

   - **场景**：返回大文件、LLM 流式输出（比如 ChatGPT 逐字回复）、实时日志流等；

   - 示例（LLM 流式输出）:

     ```python
     from fastapi.responses import StreamingResponse
     import asyncio
     
     async def generate_stream():
         # 模拟流式输出（逐字返回）
         words = ["你", "好", "，", "这", "是", "流", "式", "响", "应"]
         for word in words:
             await asyncio.sleep(0.5)
             yield word.encode("utf-8")  # 流式输出需返回字节流
     
     @app.get("/stream")
     async def stream_response():                             
         return StreamingResponse(generate_stream(), media_type="text/event-stream")
     ```

7. Response（基础响应类）

   - **作用**：所有响应类的父类，用于自定义任意格式的响应；

   - **场景**：需要高度定制响应（比如自定义 MIME 类型、响应体格式）；

   - 示例：

     ```python
     from fastapi.responses import Response
     
     @app.get("/custom")
     def custom_response():
         # 返回二进制数据，指定自定义 MIME 类型
         return Response(
             content=b"custom binary data",
             media_type="application/octet-stream",
             status_code=200
         
     ```

总结

1. `FileResponse` 是 FastAPI 专门用于**返回文件**的响应类，支持流式传输，适合静态文件 / 下载场景；
2. FastAPI 核心响应类型按场景分类：
   - 常规数据：`JSONResponse`（默认）；
   - 文件：`FileResponse`；
   - 动态 HTML：`HTMLResponse`；
   - 纯文本：`PlainTextResponse`；
   - 重定向：`RedirectResponse`；
   - 流式数据：`StreamingResponse`；
   - 自定义响应：`Response`（父类）；
3. 选择响应类型的核心原则：**匹配返回数据的格式和业务场景**（比如静态文件用 `FileResponse`，流式输出用 `StreamingResponse`）。

### 6.2 web服务实现过程

**文件位置**:`app/import_process/api/file_import_service.py`

本节将展示完整的 Web 服务实现代码。该服务基于 FastAPI 构建，提供了文件上传、后台任务调度、状态查询以及静态页面服务功能。

#### 1. 依赖库 

在运行本服务前，请确保安装以下 Python 依赖库（已包含在 `requirements.txt` 中）：

```bash
uv add fastapi uvicorn python-multipart python-dotenv minio
```

此外，本项目依赖 LangGraph 和 LangChain 相关库来执行后台任务。

#### 2. 接口介绍

本 Web 服务提供 **3 个核心接口**，分别负责文件上传、状态查询和页面访问：

##### 接口 1：文件上传接口 (`POST /upload`)

**功能**：接收前端上传的文件（支持批量上传），保存到本地临时目录，并为每个文件生成唯一的 TaskID，启动后台的 LangGraph 导入流程。

**请求方式**：`POST`

**请求参数**：
- `files`: 文件列表（form-data 格式），支持多文件同时上传

**响应数据**：
```json
{
  "code": 200,
  "message": "Files uploaded successfully, total: 2",
  "task_ids": ["uuid-1", "uuid-2"]
}
```

**处理流程**：
1. 为每个上传的文件生成唯一的 TaskID（UUID4）
2. 按日期和 TaskID 分层创建本地存储目录（`output/YYYYMMDD/{task_id}/`）
3. 将文件保存到本地临时目录
4. 启动后台任务 `run_graph_task()`，异步执行 LangGraph 全流程
5. 返回所有 TaskID 给前端，供后续轮询状态

**关键特性**：
- 支持批量上传，一次请求可上传多个文件
- 每个文件独立生成 TaskID，互不影响
- 使用 FastAPI 的 `BackgroundTasks` 实现异步处理，不阻塞 HTTP 响应
- 实时更新任务状态到内存字典，供前端轮询

---

##### 接口 2：任务状态查询接口 (`GET /status/{task_id}`)

**功能**：根据 TaskID 查询单个文件的处理进度和全局状态，前端通过轮询此接口实时展示导入进度。

**请求方式**：`GET`

**请求参数:**

- `task_id`: 路径参数，由 `/upload` 接口返回的唯一任务 ID

**响应数据**：
```json
{
  "code": 200,
  "task_id": "uuid-1",
  "status": "processing",
  "done_list": ["node_entry", "node_pdf_to_md", "node_md_img"],
  "running_list": ["node_document_split"]
}
```

**字段说明**：
- `status`: 任务全局状态，可选值：`pending`（待处理）、`processing`（处理中）、`completed`（已完成）、`failed`（失败）
- `done_list`: 已完成的节点名称列表，按执行顺序排列
- `running_list`: 正在执行的节点名称列表（通常只有一个）

**处理流程**：
1. 从内存字典中读取任务状态（高性能，无 IO 操作）
2. 返回任务的当前状态、已完成节点和正在执行的节点
3. 前端每 2 秒轮询一次，动态更新进度条和日志展示

**关键特性**：
- 纯内存读取，响应速度极快（毫秒级）
- 实时反映 LangGraph 节点的执行进度
- 支持前端动态渲染节点执行日志

---

##### 接口 3：导入演示页面接口 (`GET /html`)

**功能**：返回一个原生 HTML/JS 页面，提供可视化的文件上传和进度监控界面。

**请求方式**：`GET`

**请求参数**：无

**响应数据**：HTML 页面（`import.html`）

**页面功能**：
1. **文件拖拽/选择**：支持 PDF 和 MD 文件上传
2. **上传进度条**：显示文件上传到服务器的进度
3. **状态轮询**：上传成功后，自动每 2 秒请求一次 `/status` 接口
4. **日志展示**：根据后端返回的 `done_list` 和 `running_list`，动态渲染任务执行日志
5. **状态提示**：用不同颜色标识任务状态（绿色=完成，红色=失败，蓝色=处理中）

**访问地址**：`http://127.0.0.1:8000/html`

**关键特性**：
- 无需任何前端框架，纯原生 HTML/JS 实现
- 简洁直观的交互界面，适合快速测试和演示
- 实时展示 LangGraph 节点的执行过程

#### 3. 代码实现详情

##### (0) 定义响应数据模型

位置: `app/api/schemas/import_.py`

```python
"""
应用主包 / 接口层 / 数据模型层中的 import_ 模块，负责承载对应场景的具体实现逻辑。
"""
from pydantic import BaseModel

# 继承 BaseModel ，是为了让 FastAPI / Pydantic 把这个类当成“数据模型”处理。
# 这样它才会帮你做这些事：
#- 解析 JSON
#- 类型校验
#- 自动补默认值
#- 自动生成接口文档
#- 自动把对象转成 JSON 响应
class UploadResponse(BaseModel):
    code: int = 200
    message: str
    task_ids: list[str]


class ImportStatusResponse(BaseModel):
    code: int = 200
    task_id: str
    status: str | None = None
    done_list: list[str]
    running_list: list[str]
```

##### (1) 引入依赖与环境配置

首先，导入必要的系统库、FastAPI 组件以及我们的工具类 (`minio_utils`, `task_utils`, `main_graph`)。同时加载 `.env` 环境变量。

```python
"""
导入服务 HTTP 入口模块，直接承载导入接口与相关接口业务逻辑。
"""
import sys
import uuid
from datetime import datetime
from mimetypes import guess_type
from pathlib import Path

from fastapi import BackgroundTasks, FastAPI, File, UploadFile
from fastapi.responses import FileResponse
from starlette.middleware.cors import CORSMiddleware

from app.api.schemas.import_ import ImportStatusResponse, UploadResponse
from app.shared.runtime.logger import PROJECT_ROOT, logger
from app.process.import_.agent.main_graph import kb_import_app
from app.process.import_.agent.state import get_default_state
from app.infra.config import settings
from app.shared.utils.task_utils import (
    TASK_STATUS_COMPLETED,
    TASK_STATUS_FAILED,
    TASK_STATUS_PROCESSING,
    get_done_task_list,
    get_running_task_list,
    get_task_status,
    update_task_status,
)
```

##### (2) 应用初始化与跨域配置

初始化 FastAPI 应用，并配置 CORS（跨域资源共享）以允许前端调用。

```python
app = FastAPI(
    title=settings.import_app_name,
    description="企业化 RAG 导入服务，负责文件上传、导入执行与状态查询。",
    version="0.2.0",
)
app.add_middleware(
    CORSMiddleware,
    allow_origins=list(settings.cors_origins) or ["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/html")
def import_html():
    """
    返回导入演示页面。
    Returns:
        FileResponse: 本地导入演示页面文件响应。
    """
    html_path = PROJECT_ROOT / "app" / "process" / "import_" / "page" / "import.html"
    return FileResponse(path=html_path, media_type=guess_type(html_path.name)[0])
```

##### (3) 后台任务逻辑

这是连接 Web 服务与 LangGraph 的桥梁。`run_graph_task` 函数会在后台运行（不阻塞 HTTP 响应），它监听图的执行事件，并实时更新任务状态。

```python
# --------------------------
# 后台任务：LangGraph全流程执行
# 独立于主请求线程，由BackgroundTasks触发，避免阻塞接口响应
# --------------------------
def run_graph_task(task_id: str, local_dir: str, local_file_path: str):
    """
    LangGraph全流程执行后台任务
    核心流程：初始化状态 → 流式执行图节点 → 实时更新任务状态 → 异常捕获
    任务状态更新：pending → processing → completed/failed
    节点进度更新：每完成一个节点，将节点名加入done_list，供前端轮询查看

    :param task_id: 全局唯一任务ID，关联单个文件的全流程处理
    :param local_dir: 该任务的本地文件存储目录（含临时文件/解析结果）
    :param local_file_path: 上传文件的本地绝对路径
    """
    try:
        # 1. 更新任务全局状态为：处理中
        update_task_status(task_id, "processing")
        logger.info(f"[{task_id}] 开始执行LangGraph全流程，本地文件路径：{local_file_path}")

        # 2. 初始化LangGraph状态：加载默认状态 + 注入当前任务的核心参数
        init_state = get_default_state()
        init_state["task_id"] = task_id  # 任务ID关联
        init_state["local_dir"] = local_dir  # 任务本地目录
        init_state["local_file_path"] = local_file_path  # 上传文件本地路径

        # 3. 流式执行LangGraph全流程（stream模式：实时获取每个节点的执行结果）
        for event in kb_import_app.stream(init_state):
            for node_name, node_result in event.items():
                # 记录每个节点完成的日志，包含任务ID和节点名，方便追踪执行顺序
                logger.info(f"[{task_id}] LangGraph节点执行完成：{node_name}")
                # 将完成的节点名加入【已完成列表】，前端轮询/status/{task_id}可实时获取
                add_done_task(task_id, node_name)

        # 4. 全流程执行完成，更新任务全局状态为：已完成
        update_task_status(task_id, "completed")
        logger.info(f"[{task_id}] LangGraph全流程执行完毕，任务完成")

    except Exception as e:
        # 5. 捕获全流程异常，更新任务全局状态为：失败，并记录错误日志（含堆栈）
        update_task_status(task_id, "failed")
        logger.error(f"[{task_id}] LangGraph全流程执行失败，异常信息：{str(e)}", exc_info=True)
```

##### (4) 文件上传接口 (/upload)

```python
@app.post("/upload", summary="文件上传接口", description="支持多文件批量上传")
async def upload_files(
    background_tasks: BackgroundTasks,
    files: List[UploadFile] = File(...)
):
    # 1. 构建本地存储根目录：output/YYYYMMDD
    today_str = datetime.now().strftime("%Y%m%d")
    date_based_root_dir: Path = PROJECT_ROOT / "output" / today_str

    task_ids = []

    # 2. 遍历处理每个上传的文件
    for file in files:
        task_id = str(uuid.uuid4())
        task_ids.append(task_id)

        # 3. 标记「文件上传」阶段为「运行中」
        add_running_task(task_id, "upload_file")

        # 4. 构建任务的本地独立目录
        task_local_dir: Path = date_based_root_dir / task_id
        task_local_dir.mkdir(parents=True, exist_ok=True)

        # 5. 保存文件到本地
        local_file_abs_path: Path = task_local_dir / file.filename
        with local_file_abs_path.open("wb") as file_buffer:
            # copyfileobj 好处：
            # 流式读取：一次只读一小段（默认 64KB）
            # 读完一段写一段，循环直到写完
            # 内存永远只占用 64KB，不管文件多大
            # 速度极快，系统底层优化
            # 不会阻塞服务器，支持高并发
            # 自带缓冲区，不用自己处理
            shutil.copyfileobj(file.file, file_buffer)

        # 6. 标记「文件上传」阶段为「已完成」
        add_done_task(task_id, "upload_file")

        # 7. 启动后台任务
        background_tasks.add_task(
            run_graph_task,
            task_id,
            str(task_local_dir),
            str(local_file_abs_path)
        )

    return UploadResponse(
        code=200,
        message=f"Files uploaded successfully, total: {len(files)}",
        task_ids=task_ids
    )
```

##### (5) 任务状态查询接口 (/status)

```python
@app.get("/status/{task_id}", summary="任务状态查询", response_model=ImportStatusResponse)
async def get_task_progress(task_id: str):
    status = get_task_status(task_id)
    done_list = get_done_task_list(task_id)
    running_list = get_running_task_list(task_id)

    logger.info(f"[{task_id}] 任务状态查询，当前状态：{status}，已完成节点：{done_list}")

    return ImportStatusResponse(
        code=200,
        task_id=task_id,
        status=status,
        done_list=done_list,
        running_list=running_list
    )
```

##### (6) 启动入口

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host=settings.app_host, port=settings.import_app_port)
```
