# 第07章：Coze的Windows平台部署

## 1、整体概述

### 1.1 Coze的开源

字节跳动于2025年7月26日开源其AI智能体开发平台Coze（中文名“扣子”），短短48小时内GitHub星标数突破9000+。

最大亮点在于其极致亲民的硬件要求——普通家用电脑（`2核CPU+4GB内存`）即可流畅运行。



**为什么Coze开源是劲爆新闻？**

之前我们在Coze上搭建的智能体只能交付给C端用户，如果交付给B端用户通常都是用Dify、n8n等平台上搭建智能体交付，因为企业用户要求`数据绝对安全`，放在公网上是不能接受的，而Dify恰好是可以私有化部署的。

现在Coze也开源了，意味着以后更多了一种选择，这绝对可以说是一个里程碑式的进步。



**为什么选择开源Coze？**

- 零成本商用：采用Apache 2.0协议，意味着你可以自由地用于商业用途，并进行二次开发
- 全链路开源：覆盖Agent开发（Studio）、测试/运维（Loop）、部署（SDK）
- 硬件平民化：告别动辄16G显存的GPU，普通笔记本即可运行AI工作流

### 1.2 两大核心组件

**两大核心组件：Coze Studio和Coze Loop。**

- 可视化开发工具Coze Studio：提供各类最新大模型和工具、多种开发模式和框架，从开发到部署，为你提供最便捷的 AI Agent 开发环境。github地址：https://github.com/coze-dev/coze-studio

- 运维管理工具Coze Loop：用于AI Agent运维和生命周期管理的平台。github地址：https://github.com/coze-dev/coze-loop

### 1.3 Coze本地化部署流程

Coze的本地化部署主要依赖Docker，其核心流程可以概括为以下四个阶段。

| 阶段        | 核心任务                            | 关键操作/说明                                                |
| :---------- | :---------------------------------- | :----------------------------------------------------------- |
| 1. 环境准备 | 确保系统满足条件<br/>并安装必要软件 | 确认电脑配置（建议双核CPU+4G内存以上），<br>安装`Docker`和`Git`。 |
| 2. 获取代码 | 下载Coze Studio开源<br/>代码到本地  | 通过`git clone`命令或直接从GitHub下载ZIP<br/>压缩包的方式获取源码。 |
| 3. 配置模型 | 配置Coze将要使用的<br/>大语言模型   | 这是关键一步，主要有两种选择：**云端API模型**<br/>（如火山方舟）或**本地模型**（如通过Ollama部署）。 |
| 4. 启动服务 | 使用Docker编译并<br/>运行所有服务   | 在项目目录下执行Docker命令，完成后通过<br/>浏览器访问 `http://localhost:8888`即可。 |

## 2、Coze Studio的安装和配置

![image-20251111142845795](images/image-20251111142845795.png)

### 2.1 安装Docker(环境准备)

**Docker是唯一前置依赖**，用于创建隔离运行环境：

#### ① 下载安装包

- Docker官网下载地址：https://www.docker.com/products/docker-desktop/
- 国内用户若下载慢，可使用飞书镜像包：https://ay6exk7fyt.feishu.cn/drive/folder/IccNf2B4JlJOIvd2kWfcpYnRn4l

![image-20251111160545138](images/image-20251111160545138.png)

#### ② 安装设置及启动

<img src="images/image-20251111160752709.png" alt="image-20251111160752709" style="zoom:80%;" />

<img src="images/image-20251111160850150.png" alt="image-20251111160850150" style="zoom:80%;" />

安装完成后打开Docker Desktop，确认状态栏显示 **“Running”**  ✅

<img src="images/image-20251111161225525.png" alt="image-20251111161225525" style="zoom:80%;" />

![image-20251111175328144](images/image-20251111175328144.png)

> 个别首次安装的小伙伴会被windows系统提示需要安装`适用于Linux的Windows子系统`。这里选择确认安装。稍等片刻后会完成安装。

#### ③ 配置镜像加速器或docker代理

在安装Coze之前，我们要先进行Docker中镜像网站的设置，因为默认的镜像是国外的网址，访问不到或比较慢。需要设置镜像加速器或docker代理。

> 说明：如果大家设置过代理，那么推荐方式2。如果没有，则使用方式1。

##### 方式1：配置镜像加速器

![image-20251118113106995](images/image-20251118113106995.png)

```bash
"registry-mirrors": [  
"https://registry.docker-cn.com",
"https://s4uv0fem.mirror.aliyuncs.com",
"https://docker.1ms.run",
"https://registry.dockermirror.com",
"https://docker.m.daocloud.io",
"https://docker.kubesre.xyz",
"https://docker.mirrors.ustc.edu.cn",
"https://docker.1panel.live",
"https://docker.kejilion.pro",
"https://dockercf.jsdelivr.fyi",
"https://docker.jsdelivr.fyi",
"https://dockertest.jsdelivr.fyi",
"https://hub.littlediary.cn",
"https://proxy.1panel.live",
"https://docker.1panelproxy.com",
"https://image.cloudlayer.icu",
"https://docker.1panel.top",
"https://docker.anye.in",
"https://docker-0.unsee.tech",
"https://hub.rat.dev",
"https://hub3.nat.tf",
"https://docker.1ms.run",
"https://func.ink",
"https://a.ussh.net",
"https://docker.hlmirror.com",
"https://lispy.org",
"https://docker.yomansunter.com",
"https://docker.xuanyuan.me",
"https://docker.mybacc.com",
"https://dytt.online",
"https://docker.xiaogenban1993.com",
"https://dockerpull.cn",
"https://docker.zhai.cm",
"https://dockerhub.websoft9.com",
"https://dockerpull.pw",
"https://docker-mirror.aigc2d.com",
"https://docker.sunzishaokao.com",
"https://docker.melikeme.cn"  
]

```

##### 方式2：配置docker代理

![image-20251124171921123](images/image-20251124171921123.png)

![image-20251124171941066](images/image-20251124171941066.png)

7890更改为本地监听的代理窗口。

到这里，Docker已经安装配置好了。

### 2.2 安装Coze Studio

#### ① 下载Coze安装包

##### 方式1：Github/Gitee

1. 打开Docker Desktop内置终端（右下角Terminal图标）
2. 执行以下命令：

```bash
# 克隆官方仓库代码
git clone https://github.com/coze-dev/coze-studio.git
```

**注意：**由于github下载较慢，大家可以将github上coze-studio镜像下载到gitee平台，然后从gitee平台下载。

1）在gitee平台上：

![image-20251117184702974](images/image-20251117184702974.png)

2）复制Github上的coze-studio地址

![image-20251117184601700](images/image-20251117184601700.png)

粘贴到：

![image-20251117184834778](images/image-20251117184834778.png)

3）下载完成以后，复制gitee上的地址：

![image-20251117184633048](images/image-20251117184633048.png)

> 大家可以直接粘贴如下地址：
>
> git clone https://gitee.com/shkstart/coze-studio.git

粘贴到docker desktop客户端：

![image-20251117184502344](images/image-20251117184502344.png)

> 说明：git是一个从代码仓库拉取代码的工具，大家通过以下网址下载，安装一下即可。安装非常简单，安装以后，就可以使用git命令了。如果你没有安装git，请先安装git：https://git-scm.com/downloads

##### 方式2：解压zip包

如果你不想安装git，你也可以直接下载github上coze的zip包，如下图：

https://github.com/coze-dev/coze-studio

![image-20251114174617570](images/image-20251114174617570.png)

下载后解压到指令目录即可。

#### ② 安装并配置模型

首次部署并启动 Coze Studio 开源版本之前，需要在 Coze Studio 项目中配置模型服务。否则，在创建Agent或工作流时将无法正确选择模型。

1、从模板目录复制doubao-seed-1.6模型的模板文件，并粘贴到配置文件目录中。

首先，进入根目录coze-studio下，在地址栏中输入“cmd”并按回车键。

![image-20251118143432201](images/image-20251118143432201.png)

执行命令：

```bash
copy backend\conf\model\template\model_template_ark_doubao-seed-1.6.yaml backend\conf\model\ark_doubao-seed-1.6.yaml
```

2、修改配置文件目录中的模板文件，填入对应的参数

`id`：Coze Studio 中的模型 ID，由开发者自主定义，必须为非零整数，全局唯一。模型上线后请勿修改模型 ID。
`meta.conn_config.api_key`：在线模型服务的 API Key，获取方式见下方获取Key和Model接入点的api_key。
`meta.conn_config.model`：在线模型服务的模型 ID。本例中为 Volcengine Ark doubao-seed-1.6 模型接入点的 Endpoint ID，获取方式见下方获取Key和Model接入点的model。

**配置方案说明：**

模型配置是部署的核心，主要有以下两种方案，根据自己的需求（如网络条件、数据敏感性、成本）进行选择。

| 配置方式                 | 优点                                           | 缺点                                                         | 适用场景                                                 |
| :----------------------- | :--------------------------------------------- | :----------------------------------------------------------- | :------------------------------------------------------- |
| 云端API模型 (如火山方舟) | 模型能力强，响应速度快，无需消耗本地计算资源。 | 需要API Key（可能产生费用），需要联网，数据需传输到厂商云端。 | 体验Coze全部功能，需要最先进的模型能力，开发测试环境。   |
| 纯本地模型 (通过Ollama)  | `数据完全私密`，离线可用，API调用`免费`。      | 本地硬件要求较高（尤其需要较好GPU），模型性能可能不及顶级云端模型。 | 对数据安全有严格要求，内网环境，希望完全掌控模型的场景。 |

##### 方案1：配置云端API模型（以火山方舟为例）

**1、创建 API Key**

进入火山引擎官网 https://www.volcengine.com ，打开【控制台】。

![img](images/339881e3f53966366727d31e8730fd3f.png)

搜索并进入【火山方舟】

![image-20251118092704469](images/image-20251118092704469.png)

点击【API Key 管理】下的【创建 API Key】，选择【创建】

![img](images/1d6b1dd74052cb2f0fa07bb3a1fa4f75.png)

点击小眼睛即可查看 API Key ，复制备用。

![img](images/e891e1b565f5f58d50e157b82188e335.png)

**2、创建Endpoint**

进入【在线推理】页面，选择【自定义推理接入点】，点击【创建推理接入点】

![img](images/8f1bb883c08c185d60fed4efa5502f49.png)

输入接入点名称，建议以模型命名，点击【添加模型】

![img](images/828b99829c14492a4820c7ddd1ec0ae7.png)

模型目前支持：豆包、DeepSeek、Kimi、Qwen，这里以豆包 1.6 为例，选择后确定。

![img](images/41785ba8ed5b7f0b1bbec5fbc004ad0b.png)

或

![image-20251118152946085](images/image-20251118152946085.png)

勾选协议，点击【开通模型并接入】。

![image-20251118153103848](images/image-20251118153103848.png)

补充：如果是模型首次开通需要实名认证，输入个人信息，手机刷脸验证。

复制Endpoint。注意：模型名称下方的 ID 就是Endpoint。

![image-20251118153248976](images/image-20251118153248976.png)

**3、配置 Coze 文件**

找到前面的文件：ark_doubao-seed-1.6.yaml，进行编辑

- id 修改为任意 5 位以上纯数字。

![image-20251118112100248](images/image-20251118112100248.png)

- 将前面创建的【API Key】和【Endpoint】填入下图位置内，保存并关闭文件。

![image-20251118153405463](images/image-20251118153405463.png)

##### 方案2：配置纯本地模型（通过Ollama）

这种方法可以实现完全离线的私有化部署。

**1、安装Ollama**：首先在本地安装Ollama，它是一个用于在本地运行大模型的工具。

**2、拉取模型**：通过Ollama拉取你想要的模型，例如在命令行中执行 `ollama pull qwen2.5:7b`来下载一个开源模型。

**3、配置Coze**：在Coze项目目录下，找到Ollama的配置文件模板model_template_ollama.yaml，

![image-20251118154113127](images/image-20251118154113127.png)

将其复制并重命名为model_ollama.yaml，保存到 backend/conf/model/ 目录下：

![image-20251118154236734](images/image-20251118154236734.png)

修改其中的 base_url为Ollama的服务地址（通常是 `http://host.docker.internal:11434`），并指定你拉取的 model 名称。

![image-20251118154604090](images/image-20251118154604090.png)

#### ③ 安装并启动Coze

下图中有一个名叫docker的目录，我们要进入到这个项目文件夹中，进行安装。

<img src="images/image-20251118151026988.png" alt="image-20251118151026988" style="zoom:80%;" />

在coze-studio\docker当前目录下，测试如下：

**1、输入docker，点击回车键后，返回下图这样的结果，表明安装是成功的，没有任何问题。**

<img src="images/image-20251118151237965.png" alt="image-20251118151237965" style="zoom:80%;" />

**2、环境变量配置**

执行如下命令，重命名环境配置文件：

```bash
copy .env.example .env   #或执行：cp .env.example .env
```

![img](images/f4834173ceb430f8dd2337b1c56c2e74.jpeg)

**3、在Docker里启动Coze**

首次启动可能需要5-10分钟（依赖网络速度），运行一下这条命令：

```bash
docker compose --profile '*' up -d
```

![image-20251118153815941](images/image-20251118153815941.png)

> 这个命令的含义是：
>
> - `docker compose`：使用Docker Compose运行服务
> - `--profile '*'`：启用所有profile配置
> - `up`：启动服务（没有就创建容器，有就重启）
> - `-d`：detached模式，即在后台运行

出现下图表示成功：

![image-20251118154819866](images/image-20251118154819866.png)

**4、安装结束后，查看运行状态**

```bash
docker compose ps
```

打开安装好的docker客户端，正常就可以看到docker启动起来了。

![image-20251118155151948](images/image-20251118155151948.png)

#### ④ 访问Coze Studio界面

安装好后，我们打开浏览器，访问http://localhost:8888/来打开 Coze Studio ，可以看到如下界面。

![image-20251118155219214](images/image-20251118155219214.png)

登录进去，即可正常使用了

![image-20251118155338542](images/image-20251118155338542.png)

正常就可以看到docker启动起来了。

> 小遗憾：目前功能较商业版还比较简陋，但未来可期！

## 3、CozeLoop(扣子罗盘)指南

### 3.1 介绍

![image-20251118160521163](images/image-20251118160521163.png)

扣子罗盘通过提供全生命周期的管理能力，帮助开发者更高效地开发和运维 AI Agent。无论是提示词工程、AI Agent 评测，还是上线后的监控与调优，扣子罗盘都提供了强大的工具和智能化的支持，极大地简化了 AI Agent 的开发流程，提升了 AI Agent 的运行效果和稳定性。

### 3.2 部署

#### ① 准备工作

安装 CozeLoop 开源版之前，确保您的软硬件环境满足以下要求：

**1、Go语言环境：**已安装Go SDK，且版本为1.23.4及以上版本。配置GOPATH，同时将${GOPATH}/bin加入到环境变量PATH中，保证安装的二进制工具可找到并运行。

Go语言官网：https://go.dev/dl/

![image-20251124100941713](images/image-20251124100941713.png)

下载完成后双击打开，运行

<img src="images/image-20251124101005119.png" alt="image-20251124101005119" style="zoom:80%;" />

按提示一直“Next”即可。

安装完成后，确保GO家目录下的bin目录已在环境变量中。

<img src="images/image-20251127091156466.png" alt="image-20251127091156466" style="zoom:80%;" />

**2、Docker环境**：提前安装Docker、Docker Compose，并启动 Docker 服务

**3、模型**：已开通OpenAI或火山方舟等在线模型服务。

#### ② 下载/克隆仓库

这里推荐方式1，不推荐方式2，是因为方式2下载以后，在⑤启动服务时，总报错。

比如：

![image-20251127180605483](images/image-20251127180605483.png)

##### 方式1：下载zip文件(推荐)

从github下载coze loop的zip文件到本地，解压即可。

```
https://github.com/coze-dev/coze-loop
```

![image-20251127180217630](images/image-20251127180217630.png)

我存放到了如下路径：

![image-20251127180506504](images/image-20251127180506504.png)

##### 方式2：克隆仓库

```bash
git clone https://gitee.com/shkstart/coze-loop.git
```

![image-20251118160139754](images/image-20251118160139754.png)

#### ③ 配置模型

编辑文件coze-loop-main\release\deployment\docker-compose\conf\model_config.yaml，修改 api_key 和 model 字段。以火山方舟为例：

- api_key：火山方舟 API Key。（参考Coze Studio中的同步骤情况）
- model：火山方舟模型接入点的 Endpoint ID。（参考Coze Studio中的同步骤情况）

![image-20251118164111775](images/image-20251118164111775.png)

#### ④ 更改端口

1、打开coze-loop-main\release\deployment\docker-compose目录下的.env，将COZE_LOOP_APP_OPENAPI_PORT更改为`8889`或者其它未被占用的端口。

![image-20251124102156323](images/image-20251124102156323.png)

2、打开coze-loop/release/deployment/docker-compose目录下的docker-compose.yml，将${COZE_LOOP_APP_OPENAPI_PORT}的端口更改为`8889`，注意要和.env中COZE_LOOP_APP_OPENAPI_PORT的端口`保持一致`。

![image-20251124102417593](images/image-20251124102417593.png)

#### ⑤ 启动服务

在coze-loop-main\release\deployment\docker-compose目录下执行以下命令，使用 Docker Compose 快速部署 Coze Loop 开源版。

```cmd
docker compose -f docker-compose.yml --env-file .env --profile "*" up -d
```

如果不希望以守护进程（放到后台，命令行窗口不会打印完整日志，且启动成功后窗口退出则进程中断）的形式启动，则可以去掉`-d`，首次推荐不加`-d`，方便查看日志。

首次启动推荐命令

```cmd
docker compose -f docker-compose.yml --env-file .env --profile "*" up
```

首次启动需要拉取镜像、构建本地镜像，可能耗时较久。看到以下日志，则部署完成。

![image-20251127181022788](images/image-20251127181022788.png)

![image-20251127181129672](images/image-20251127181129672.png)

docker客户端中，当前容器除了xxx-init的镜像启动后又关闭之外，其它的镜像都需要是开启状态的，那就对了。

#### ⑥ 访问 CozeLoop 开源版

通过浏览器访问 Coze Loop 开源版http://localhost:8082。

![image-20251119145214972](images/image-20251119145214972.png)

注册完成后即可进入应用详情页。

![image-20251111113044531](images/image-20251111113044531.png)

### 3.3 Coze Loop的使用

这里演示在线版本：https://www.coze.cn/loop

Coze-loop官方提供了一些示例，位于在线版的**Demo空间**

![image-20251124185613545](images/image-20251124185613545.png)

#### 功能1：Prompt 开发

该模块用于进行提示词的预览与调试

![image-20251124185628090](images/image-20251124185628090.png)

点击详情

![](images/image-20251124185716493.png)

![image-20251124185758202](images/image-20251124185758202.png)

#### 功能2：Playground

该模块和Prompt开发功能类似，区别在于**自由对比模式**。

![image-20251124185909365](images/image-20251124185909365.png)

官方Demo空间中，我们无权开启自由对比模式，切换到个人空间查看

![image-20251124185959202](images/image-20251124185959202.png)

![image-20251124190051796](images/image-20251124190051796.png)

![image-20251124190108340](images/image-20251124190108340.png)

#### 功能3：评测集

该模块用于管理评测数据集。

![image-20251124190150317](images/image-20251124190150317.png)

![image-20251124190204309](images/image-20251124190204309.png)

![image-20251124190234288](images/image-20251124190234288.png)

#### 功能4：评估器

该模块用于构建评估**Prompt 开发**实例的工具，本质上也是添加了提示词的大模型。

![image-20251124190747606](images/image-20251124190747606.png)

![image-20251124190805543](images/image-20251124190805543.png)

本质上也是**提示词+大模型**

![image-20251124190818807](images/image-20251124190818807.png)

#### 功能5：实验

该模块用于创建和管理实验。

![image-20251124190322906](images/image-20251124190322906.png)

同样地，Demo空间无权创建实验，切换到个人空间即可。

![image-20251124190418529](images/image-20251124190418529.png)

实验详情如下。

![image-20251124190652454](images/image-20251124190652454.png)

**指标统计**模块以可视化的方式展示评估结果

![image-20251124190927072](images/image-20251124190927072.png)

新建实验

![image-20251124191513447](images/image-20251124191513447.png)

依次设置基础信息、评测集、评测对象、评估器即可。

#### 功能6：Trace

该模块记录了详细的运行信息。

![image-20251124190956748](images/image-20251124190956748.png)

![image-20251124191026402](images/image-20251124191026402.png)

![image-20251124191036884](images/image-20251124191036884.png)

#### 功能7：统计

该模块统计整个空间的运行情况。

#### 功能8：自动化任务

该模块用于创建和管理自动化评测任务。

![image-20251124191652028](images/image-20251124191652028.png)

详情页

![image-20251124191706788](images/image-20251124191706788.png)

![image-20251124191729562](images/image-20251124191729562.png)



