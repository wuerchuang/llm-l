## 1. Claude Code安装

```shell
# 设置为淘宝镜像源
npm config set registry https://registry.npmmirror.com
# 验证是否设置成功
npm config get registry
# 下载Claude code
npm install -g @anthropic-ai/claude-code
# 验证安装
claude --version
# 简单使用
claude
```

## 2. Claude Code配置

> 安装好 Claude Code 后，需要配置 API 密钥或登录方式才能使用

### 2.1. Claude Code配置说明

> 配置文件在 `用户目录/.claude/settings.json`(此文件如果不存在可以手动创建的)

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-????",
    "ANTHROPIC_BASE_URL": "https://cloud.hongqiye.com",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL_NAME": "claude-opus-4-5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_SONNET_MODEL_NAME": "deepseek-v4-pro",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME": "glm-5"
  },
  "model": "sonnet"
}
```

### 2.2. 核心配置参数速查表

| 参数（环境变量）                 | 作用                         | 何时使用                |
| -------------------------------- | ---------------------------- | ----------------------- |
| `ANTHROPIC_API_KEY`              | Anthropic 官方 API Key       | 直接使用官方服务时      |
| `ANTHROPIC_AUTH_TOKEN`           | 第三方平台的 API Key         | 使用中转 / 第三方模型时 |
| `ANTHROPIC_BASE_URL`             | API 端点地址（覆盖默认地址） | 使用中转 / 第三方服务时 |
| `ANTHROPIC_MODEL`                | 默认使用的模型名称或别名     | 持久指定默认模型        |
| `ANTHROPIC_DEFAULT_OPUS_MODEL`   | opus 槽位映射的具体模型      | 自定义三级槽位映射      |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | sonnet 槽位映射的具体模型    | 自定义三级槽位映射      |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL`  | haiku 槽位映射的具体模型     | 自定义三级槽位映射      |
|                                  |                              |                         |

💡 提示：参数关系说明：

- `ANTHROPIC_API_KEY` 用于官方直连，`ANTHROPIC_AUTH_TOKEN` 用于第三方服务。**两者不要同时设置，否则会冲突**。
- `ANTHROPIC_BASE_URL` 只在使用非官方端点时需要设置。
- 也可以将`ANTHROPIC_AUTH_TOKEN`和`ANTHROPIC_BASE_URL`设置为deepseek或qwen的key和url

### 2.3. AI模型聚合平台（红七叶）

- 官网注册：https://cloud.hongqiye.com/

- 获取API_KEY，替换掉上面配置中的`ANTHROPIC_AUTH_TOKEN`

## 3. pycharm中使用CC GUI

- 插件市场下载CC GUI，重启pycharm，`打开CC插件的设置`

- 设置语言为`简体中文`

- 供应商管理中，Claude Code的供应商选择`使用本地setttings.json`

- SDK依赖管理：下载Claude Code SDK

- 在最下方可以`切换模型`为deepseek的模型

- 回到聊天界面，`新建会话`后提问`你是谁?，你用的什么模型？`, 结果显示

  ```
  你好！我是 Claude Code
  
  。。。。
  
  目前我运行在 DeepSeek-v4-pro 模型上
  ```

  