# 启动与演示说明

本文档用于项目交付、课程演示和本地自检。

## 一、启动前准备

### 1. 创建虚拟环境并安装依赖

推荐使用 `uv`：

```bash
uv venv
uv sync
```

或者使用 `venv + pip`：

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -e .
```

### 2. 配置环境变量

复制示例文件：

```bash
copy .env.example .env
```

然后填写以下核心配置：

- `OPENAI_BASE_URL`
- `OPENAI_API_KEY`
- `LLM_DEFAULT_MODEL`
- `VL_MODEL`
- `MILVUS_URL`
- `MONGO_URL`
- `MONGO_DB_NAME`
- `MINIO_ENDPOINT`
- `MINIO_ACCESS_KEY`
- `MINIO_SECRET_KEY`
- `MINIO_BUCKET_NAME`
- `MINERU_BASE_URL`
- `MINERU_API_TOKEN`

### 3. 准备外部服务

确保以下依赖服务可访问：

- Milvus
- MongoDB
- MinIO
- 大模型服务
- 视觉模型服务
- MinerU PDF 解析服务
- DashScope WebSearch MCP

## 二、启动项目

推荐先启动导入服务：

```bash
uv run uvicorn app.api.http.import_server:app --host 0.0.0.0 --port 8000 --reload
```

再启动查询服务：

```bash
uv run uvicorn app.api.http.query_server:app --host 0.0.0.0 --port 8001 --reload
```

如果不使用 `uv`：

```bash
.venv\Scripts\python -m uvicorn app.api.http.import_server:app --host 0.0.0.0 --port 8000 --reload
.venv\Scripts\python -m uvicorn app.api.http.query_server:app --host 0.0.0.0 --port 8001 --reload
```

启动成功后可访问：

- 导入健康检查：`http://127.0.0.1:8000/health`
- 导入 Swagger：`http://127.0.0.1:8000/docs`
- 查询健康检查：`http://127.0.0.1:8001/health`
- 查询 Swagger：`http://127.0.0.1:8001/docs`

## 三、建议自检顺序

### 1. 先检查基础健康状态

访问：

- `GET http://127.0.0.1:8000/health`
- `GET http://127.0.0.1:8001/health`

### 2. 再检查导入链

方式一：

- 打开 `GET http://127.0.0.1:8000/html`
- 上传文档

方式二：

- 直接请求 `POST http://127.0.0.1:8000/upload`

然后轮询：

- `GET http://127.0.0.1:8000/status/{task_id}`

观察：

- `status`
- `done_list`
- `running_list`

### 3. 再检查查询链

普通问答：

- `POST http://127.0.0.1:8001/query`

流式问答：

- `POST http://127.0.0.1:8001/query`，请求体中 `is_stream=true`
- `GET http://127.0.0.1:8001/stream/{session_id}`

历史记录：

- `GET http://127.0.0.1:8001/history/{session_id}`

## 四、推荐演示问题

建议准备 3 类问题：

### 1. 标准命中型

- `HAK180 如何恢复出厂设置？`
- `HAK180 的顶部局部烫印应该如何设置？`

### 2. 多跳描述型

- `如果只想把膜转印到纸张顶部 50mm 到 170mm 区域，应该在面板上设置哪些参数？`

### 3. 需要联网补充型

- `这个设备同类产品目前行业里常见的应用场景有哪些？`

## 五、课堂演示建议顺序

### 演示 1：企业化结构

讲解：

- `app/api`
- `app/infra`
- `app/rag`

### 演示 2：导入链

讲解：

- 文档上传
- PDF 转 Markdown
- 图片增强
- 主体识别
- 切块、向量化、入库

### 演示 3：查询链

讲解：

- 问题改写
- 商品确认
- Embedding 与 HyDE 双路召回
- RRF 融合
- Rerank 精排
- 最终答案生成

## 六、已知注意事项

- 本项目对外部依赖较多，课堂环境务必提前验证
- 系统 Python 可能缺依赖，优先使用 `.venv` 或 `uv run`
- WebSearch 依赖 DashScope MCP，如配置不全可能影响联网检索
- 首次加载本地模型耗时较长，建议课前预热一次
