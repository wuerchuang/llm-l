# 1. 运行项目测试

## 1.1. 项目代码说明

- docker：后端支撑服务（mysql数据库、qdrant向量索引库和es全文索引库）
- data-agent： 掌柜问数的后端项目
- data-agent_frontend: 掌柜问数的前端项目

## 1.2. 通过docker安装中间件启动服务

- 启动window版本docker（docker desktop）
- 进入docker目录下执行命令：docker compose up -d

> 注意：
>
> 1. 第一次运行整个过程时间会比较长，耐心等待
>
> 2. 需要下载的embedding包可能需要科学上网才能成功
> 3. mysql服务可能无法启动，原因：windows中已经启动了mysql的服务，需要先停止已有的mysql服务

## 1.3. 构建知识库

- 进入后端项目目录data-agent，运行构建脚本：app/scripts/build_meta_knowledge.py

## 1.4. 运行后端项目

- 进入后端项目目录data-agent，运行启动脚本： main.py

## 1.5. 运行前端项目

- 进入前端项目目录data-agent_frontend，执行命令：npm run dev

## 1.6. 访问测试

- 浏览器上访问： http://127.0.0.1:5173/
- 搜索相关问题
  - 华北地区销售总额
  - 2025年各地区平均销售额
  - iPhone在各个地区去年卖了多少钱？
  - 各地区销量排名前三的商品