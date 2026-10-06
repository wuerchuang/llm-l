# OpenClaw Docker 安装教程

## 1. 准备工作
在开始之前，请确保你的系统已安装以下工具：
- Docker（[官方安装指南](https://docs.docker.com/get-docker/)）
- Git（[官方安装指南](https://git-scm.com/downloads)）

---

## 2. 克隆 OpenClaw 仓库
运行以下命令克隆 OpenClaw 官方仓库到本地：
```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

---

## 3. 构建 Docker 镜像
在仓库根目录下运行以下命令构建 Docker 镜像：
```bash
docker build -t openclaw .
```

---

## 4. 运行 OpenClaw
使用以下命令启动 OpenClaw 容器：
```bash
docker run -it --name openclaw openclaw
```

---

## 5. 访问 OpenClaw
启动后，访问以下地址：
- Web UI: `http://localhost:3000`
- CLI: 直接在容器内运行 `openclaw onboard` 按提示完成配置。

---

## 6. 常见问题
### 6.1 如何更新 OpenClaw？
```bash
docker stop openclaw
cd openclaw && git pull
docker build -t openclaw .
docker run -it --name openclaw openclaw
```

### 6.2 如何查看日志？
```bash
docker logs openclaw
```

---

## 7. 更多内容
参考 [OpenClaw 官方文档](https://github.com/openclaw/openclaw/blob/main/README.md) 获取更详细的配置和使用指南。