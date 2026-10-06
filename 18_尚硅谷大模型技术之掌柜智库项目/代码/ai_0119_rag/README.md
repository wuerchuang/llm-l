# 企业化 RAG 智能体项目

一个面向知识库问答场景的 RAG 智能体项目，当前已经完成兼容式企业化重构，具备以下能力：

- 文档导入：支持 `PDF -> Markdown -> 图片增强 -> 主体识别 -> 切块 -> 向量化 -> Milvus 入库`
- 智能查询：支持 `问题改写 -> 商品确认 -> Embedding 召回 -> HyDE 召回 -> WebSearch -> RRF -> Rerank -> 答案生成`
- 流式问答：支持 SSE 流式输出

## 项目特点

- 保留原有导入流程与查询流程的可运行能力，并统一收口到 `app/process`
- 当前采用 `api / infra / shared / rag / process` 分层
- 查询链核心能力已按节点职责下沉到 `app/rag/query`
- 导入链核心能力已下沉到 `app/rag/import_`
- 保持导入服务与查询服务双入口、双端口运行

## 目录结构

```text
app/
├─ api/                           # 接口层
│  ├─ http/                       # 导入/查询两个可运行 server 入口
│  └─ schemas/                    # 请求与响应数据结构
├─ infra/                         # 基础设施正式出口
│  ├─ config/                     # 应用运行配置与端口设置
│  ├─ document_parse/             # PDF 解析服务门面（MinerU）
│  ├─ llm/                        # LLM / Embedding / Reranker 提供者出口
│  ├─ object_storage/             # 对象存储门面（MinIO）
│  ├─ persistence/                # 持久化门面（聊天历史等）
│  └─ vectorstore/                # 向量库门面（Milvus）
├─ process/                       # 流程编排层
│  ├─ import_/                    # 导入图、导入状态与导入页面
│  │  ├─ agent/                   # LangGraph 导入图与各节点
│  │  └─ page/                    # 导入演示页面
│  └─ query/                      # 查询图、查询状态与查询页面
│     ├─ agent/                   # LangGraph 查询图与各节点
│     └─ page/                    # 查询演示页面
├─ rag/                           # RAG 核心能力层
│  ├─ import_/                    # 导入域能力：入口识别、解析、图片增强、切块、主体识别、向量化、入库
│  └─ query/                      # 查询域能力：主体确认、检索、HyDE、WebSearch、RRF、Rerank、答案生成
├─ resources/                     # 应用资源目录
│  └─ prompts/                    # 提示词模板
├─ shared/                        # 公共底座
│  ├─ clients/                    # 底层客户端工具（Milvus / Mongo / MinIO）
│  ├─ config/                     # 原子配置读取与各组件配置对象
│  ├─ model/                      # 模型工具封装
│  ├─ runtime/                    # 运行时能力（日志、Prompt 加载）
│  ├─ tool/                       # 模型下载等辅助脚本
│  └─ utils/                      # 通用工具（SSE、任务状态、限流、路径等）
└─ __init__.py

docs/
└─ architecture.md                # 当前项目架构说明

test/                             # 测试与实验脚本
```

## 运行环境

- Python `3.11+`
- 推荐使用 `uv`
- 推荐使用项目本地虚拟环境 `.venv`
- 需要准备外部依赖：
  - Milvus
  - MongoDB
  - MinIO
  - 大模型与视觉模型服务
  - MinerU PDF 解析服务
  - DashScope WebSearch MCP

## 安装依赖

### 方式一：使用 uv

```bash
uv venv
uv sync
```

### 方式二：使用 pip

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -e .
```

## 环境变量

1. 复制示例文件：

```bash
copy .env.example .env
```

2. 按实际环境填写：

- 应用基础配置
- LLM / VL 模型配置
- Embedding / Reranker 模型配置
- Milvus / Mongo / MinIO 配置
- MinerU 配置
- DashScope MCP 配置

详细变量说明见 `.env.example`

## 启动方式

### 启动导入服务

```bash
uv run uvicorn app.api.http.import_server:app --host 0.0.0.0 --port 8000 --reload
```

### 启动查询服务

```bash
uv run uvicorn app.api.http.query_server:app --host 0.0.0.0 --port 8001 --reload
```

如果不使用 `uv`：

```bash
.venv\Scripts\python -m uvicorn app.api.http.import_server:app --host 0.0.0.0 --port 8000 --reload
.venv\Scripts\python -m uvicorn app.api.http.query_server:app --host 0.0.0.0 --port 8001 --reload
```

### 健康检查

- 导入服务：`GET http://127.0.0.1:8000/health`
- 查询服务：`GET http://127.0.0.1:8001/health`

### 文档入口

- 导入 Swagger: `http://127.0.0.1:8000/docs`
- 查询 Swagger: `http://127.0.0.1:8001/docs`

## 主要入口

### 导入相关

- 页面：`GET /html`
- 上传：`POST /upload`
- 状态：`GET /status/{task_id}`
- 默认端口：`8000`

### 查询相关

- 页面：`GET /html`
- 查询：`POST /query`
- SSE：`GET /stream/{session_id}`
- 历史：`GET /history/{session_id}`
- 清空历史：`DELETE /history/{session_id}`
- 默认端口：`8001`

## 推荐演示流程

### 1. 先演示导入

- 打开 `http://127.0.0.1:8000/html`
- 上传一份 PDF 或 Markdown 文档
- 观察状态接口中的 `done_list / running_list`

### 2. 再演示普通问答

- 打开 `http://127.0.0.1:8001/html`
- 提问一个与文档内容强相关的问题
- 观察回答与 SSE 输出

## 当前重构状态

- 已完成双服务双端口入口恢复
- 已完成接口入口统一收敛到 `app/api`
- 已完成查询链能力层下沉
- 已完成导入链能力层下沉

## 当前已知注意事项

- 运行前必须准备完整 `.env`
- 本项目较重，首次加载本地模型耗时较长
- WebSearch 依赖 DashScope MCP 配置
- 如果系统 Python 环境缺少 `fastapi` 等依赖，请务必使用 `.venv` 或 `uv run`

## 参考文档

- 架构说明：`docs/architecture.md`
