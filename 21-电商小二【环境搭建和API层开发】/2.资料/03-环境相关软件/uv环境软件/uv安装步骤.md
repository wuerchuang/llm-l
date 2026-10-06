## 离线手动安装

### 1、下载 uv 二进制包

浏览器打开：https://github.com/astral-sh/uv/releases
下载 uv-x86_64-pc-windows-msvc.zip



### 2、解压

取出里面 3 个 exe：uv.exe、uvx.exe、uvw.exe
创建目录：C:\tools\uv，把 3 个 exe 放进去



### 3、配置环境变量

一键写入系统 PATH（管理员 PowerShell 执行）

```shell
[Environment]::SetEnvironmentVariable("PATH", "$env:PATH;C:\tools\uv", "User")
```

注：如果命令执行失败，手动配置环境变量 PATH



### 4、验证

powershell执行

输出版本号 = 安装完成

```shell
uv --version
```

输出版本号 表示安装完成