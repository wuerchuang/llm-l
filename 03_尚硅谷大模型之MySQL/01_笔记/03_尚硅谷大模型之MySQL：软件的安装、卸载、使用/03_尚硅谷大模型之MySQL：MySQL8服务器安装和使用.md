# 第1章 MySQL服务器端的安装与使用（Windows）

## 1.1 MySQL服务器端的卸载

### 1.1.1 卸载准备

学习网络编程时，TCP/IP协议程序有服务器端和客户端。Mysql这个数据库管理软件是使用TCP/IP协议。我们现在要卸载的是mysql的服务器端，它没有界面。

 【计算】-->右键-->【管理】-->【服务】-->【mysql的服务】-->【停止】

![image-20211127131123822](images/image-20211127131123822.png)

### 1.1.2 卸载

**方式一：通过控制面板卸载**

![image-20210727180032098](images/image-20210727180032098.png)

![image-20211127105018580](images/image-20211127105018580.png)

方式二：通过mysql8的安装向导卸载

1、双击mysql8的安装向导

![image-20211127111201381](images/image-20211127111201381.png)

2、取消更新

![image-20211127111141171](images/image-20211127111141171.png)

![image-20211127111248477](images/image-20211127111248477.png)

3、选择要卸载的mysql服务器软件的具体版本

![image-20211127111636535](images/image-20211127111636535.png)

![image-20211127111756888](images/image-20211127111756888.png)

4、确认删除数据目录

![image-20211127111904711](images/image-20211127111904711.png)

5、执行删除

![image-20211127112049612](images/image-20211127112049612.png)

![image-20211127112146641](images/image-20211127112146641.png)

6、完成删除

![image-20211127112224983](images/image-20211127112224983.png)

![image-20211127112303454](images/image-20211127112303454.png)

### 1.1.3 清理环境变量

找到path环境变量，将其中关于mysql的环境变量删除即，**切记<font color='red'>不要</font>把整个path删除。**

例如：删除  D:\ProgramFiles\MySQL\MySQLServer8.0_Server\bin;  这个部分

![image-20211127113140430](images/image-20211127113140430-172483406442975.png)

![image-20211127113205093](images/image-20211127113205093-172483406442976.png)

![image-20211127113258108](images/image-20211127113258108-172483406442977.png)

![image-20211127113327805](images/image-20211127113327805-172483406442978.png)

## 1.2 MySQL服务器端的安装

<font color='red'>**注意：**</font>

<font color='red'>**必须用系统管理员身份运行mysql安装程序。**</font>

<font color='red'>**安装目录切记不要用中文。**</font>



步骤一：双击mysql8的安装向导

![image-20211127111201381](images/image-20211127111201381.png)

步骤二：分为首次安装和再安装

1、首次安装

（1）如果是首次安装mysql系列的产品，需要先安装mysql产品的安装向导

![](images/微信图片_20211127130718.jpg)

（2）选择安装模式

![image-20211128175722806](images/image-20211128175722806.png)



2、不是首次安装

（1）取消更新（如果电脑上有mysql相关软件才有）

![image-20211127113631758](images/image-20211127113631758.png)



![image-20211127111248477](images/image-20211127111248477.png)

（2）选择Add安装

![image-20211127113738546](images/image-20211127113738546.png)

步骤三：选择要安装的产品

![image-20211127114653481](images/image-20211127114653481.png)

![image-20211127114719245](images/image-20211127114719245.png)

![image-20211127114744905](images/image-20211127114744905.png)

步骤四：设置软件安装目录<font color='red'>（切记服务安装目录不要有中文字符，否则有问题）</font>

![image-20211127115035455](images/image-20211127115035455.png)

![image-20211127115150647](images/image-20211127115150647.png)

![image-20211127115242110](images/image-20211127115242110.png)

![image-20211127115529359](images/image-20211127115529359.png)

![image-20211127115719270](images/image-20211127115719270.png)

步骤五：部分同学问题缺少C++库，不缺的没有这一步

![image-20221019171405576](images/image-20221019171405576.png)

![image-20221019171520254](images/image-20221019171520254.png)

![image-20221019171637281](images/image-20221019171637281.png)

步骤六：执行安装

![image-20211127115748337](images/image-20211127115748337.png)

![image-20211127115812289](images/image-20211127115812289.png)



步骤六：完成安装

![image-20211127115844966](images/image-20211127115844966.png)

步骤七：准备设置

![image-20211127120041368](images/image-20211127120041368.png)

## 1.3 MySQL实例初始化和设置

步骤一：选择安装的电脑类型、设置端口号

![image-20211127120247934](images/image-20211127120247934.png)

![image-20211127120515458](images/image-20211127120515458.png)

步骤二：选择mysql账号密码加密规则

在MySQL 5.x中默认的身份认证插件为“mysql_native_password”。

在MySQL 8.x中，默认的身份认证插件是“caching_sha2_password”，替代了之前的“mysql_native_password”。

![image-20211127120743104](images/image-20211127120743104.png)

步骤三：设置root账户密码

![image-20211127121133127](images/image-20211127121133127.png)

步骤四：设置mysql服务名和服务启动策略

如果电脑上可能安装多个版本mysql，请在服务名后面保留版本标识，例如：MySQL80，这样可以区别用哪个版本的mysql

![image-20211127121615732](images/image-20211127121615732.png)

步骤五：执行设置（初始化mysql实例）

![image-20211127121929986](images/image-20211127121929986.png)

![image-20211127122012517](images/image-20211127122012517.png)

步骤六：完成设置

![image-20211127122037556](images/image-20211127122037556.png)

![image-20211127122105747](images/image-20211127122105747.png)

![image-20211127122124855](images/image-20211127122124855.png)

![image-20211127130231815](images/image-20211127130231815.png)

## 1.4 MySQL数据库环境变量的配置

```mysql
mysql -hlocalhost -P3306 -uroot -p回车
Enter password：输入密码
```

如果运行mysql命令，报错如下错误，说明需要配置环境变量

![image-20211128172817265](images/image-20211128172817265.png)

![image-20211127133531030](images/image-20211127133531030.png)



| 环境变量名 | 操作 |                 环境变量值                  |
| :--------: | :--: | :-----------------------------------------: |
| MYSQL_HOME | 新建 | D:\ProgramFiles\MySQL\MySQLServer8.0_Server |
|    path    | 编辑 |              %MYSQL_HOME%\bin               |

或者直接

| 环境变量名 | 操作 |                   环境变量值                    |
| :--------: | :--: | :---------------------------------------------: |
|    path    | 编辑 | D:\ProgramFiles\MySQL\MySQLServer8.0_Server\bin |

![image-20211127165256909](images/image-20211127165256909.png)





## 1.5 MySQL数据库服务的启动和停止

MySQL软件的服务器端必须先启动，客户端才可以连接和使用使用数据库。

如果接下来天天用，可以设置自动启动。

### 1.5.1 图形化方式

* 计算机（点击鼠标右键）》管理（点击）》服务和应用程序（点击）》服务（点击）》MySQL80（点击鼠标右键）==》启动或停止（点击）
* 控制面板（点击）》系统和安全（点击）》管理工具（点击）》服务（点击）》MySQL80（点击鼠标右键）==》启动或停止（点击）
* 任务栏（点击鼠标右键）》启动任务管理器（点击）》服务（点击）》MySQL80（点击鼠标右键）》启动或停止（点击）

### 1.5.2 命令行方式

必须是系统管理员才能运行下面的命令。

```cmd
启动 MySQL 服务命令：
net start MySQL80

停止 MySQL 服务命令：
net stop MySQL80
```

## 1.6 MySQL客户端的登录

```java
MySQL服务器默认在3306端口。

MySQL的客户端有哪些？
（1）cmd命令行
（2）mysql数据库管理系统的服务器本地有一个自带客户端，
只能以'root'@'localhost'用户从本地登录，只需要输入密码即可。
（3）可视化图形界面工具
SQLyog、Navicat、MySQL Front、DBeaver、MySQLWorkbench等
```

### 1.6.1 MySQL自带客户端

开始菜单==》所有程序==》MySQL==》MySQL Server 8.0==》MySQL 8.0 Command Line Client

![image-20211127163824213](images/image-20211127163824213.png)

> 说明：仅限于root用户

### 1.6.2 cmd命令行客户端

**mysql -h 主机名 -P 端口号 -u 用户名 -p密码**

```sql
例如：mysql -h localhost -P 3306 -u root -proot   

-h：host 主机名/IP地址
-P：port端口号
-u：user 用户名
-p：password密码
```

注意：

（1）-p与密码之间不能有空格，其他参数名与参数值之间可以有空格也可以没有空格

```sql
mysql -hlocalhost -P3306 -uroot -proot
```

（2）密码建议在下一行输入

```sql
mysql -h localhost -P 3306 -u root -p
Enter password:****
```

（3）如果是连本机：-hlocalhost就可以省略，如果端口号没有修改：-P3306也可以省略

  简写成：

```sql
mysql -u root -p
Enter password:******
```

（4）如果输入mysql命令报“不是内部或外部命令”，把mysql安装目录的bin目录配置到环境变量path中

![image-20211127165424591](images/image-20211127165424591.png)

### 1.6.3 可视化工具Navicat

可视化图形界面工具有：SQLyog、Navicat、Datagrip、MySQL Front、DBeaver、MySQLWorkbench等

Navicat是一套可创建多个连接的数据库管理工具，用以方便管理 MySQL、Oracle、PostgreSQL、SQLite、SQL Server、MariaDB 和 MongoDB 等不同类型的数据库，它与阿里云、腾讯云、华为云、Amazon RDS、Amazon Aurora、Amazon Redshift、Microsoft Azure、Oracle Cloud 和 MongoDB Atlas等云数据库兼容。你可以创建、管理和维护数据库。Navicat 的功能足以满足专业开发人员的所有需求，但是对数据库服务器初学者来说又简单易操作。Navicat 的用户界面 (GUI) 设计良好，让你以安全且简单的方法创建、组织、访问和共享信息。

![image-20221105185217908](images/image-20221105185217908.png)

![image-20221105185300029](images/image-20221105185300029.png)





# 第2章 MySQL服务器端的安装与使用（Linux）

## 2.1 安装

在 Ubuntu 20.04/22.04、Debian10/11/12 这些主流系统中，**系统自带的官方软件源里，已经没有「MySQL 原版」了**，取而代之的是 `MariaDB`（MySQL 的分支版本）。

如果你直接执行：`sudo apt install mysql-server`，系统给你装的根本不是 Oracle 官方的 MySQL，而是 **MariaDB**，虽然用法相似，但版本、功能、兼容性都和原版 MySQL 有差异，生产环境中如果要求用纯 MySQL，这个方式就完全不行。

所以我们需要先从Oracle官网下载`mysql-apt-xxx.deb`文件（例如：`mysql-apt-config_0.8.34-1_all.deb`，`all`表示这个包适配所有 CPU 架构）。 这个文件**不是 MySQL 数据库本体、不是服务、不是插件**，它是一个「轻量化的配置包」，大小只有几十 KB，它的唯一功能就是：

1. 安装后，在你的系统目录 `/etc/apt/sources.list.d/` 下，新增一个 **MySQL 官方的源配置文件**；
2. 同时给系统导入 MySQL 官方的软件包公钥，让系统信任这个源下载的软件包，避免安装时报「签名验证失败」；
3. 弹出可视化交互界面，让你**选择要安装的 MySQL 版本（8.0/5.7）、MySQL 产品类型**。

简单理解：这个包就是一个「钥匙」，帮你打开「MySQL 官方软件仓库」的大门，之后你的`apt`命令就能从官方仓库下载正版 MySQL 了。

### 2.1.1 下载mysql-apt-xxx.deb源配置包

第一步：https://www.mysql.com/downloads/

<img src="images/image-20260116104725235.png" alt="image-20260116104725235" style="zoom:67%;" />

第二步：根据操作系统选择

<img src="images/image-20260116104821830.png" alt="image-20260116104821830" style="zoom:67%;" />

第三步：下载

<img src="images/image-20260116105012046.png" alt="image-20260116105012046" style="zoom: 80%;" />

<img src="images/image-20260116105056378.png" alt="image-20260116105056378" style="zoom:67%;" />

### 2.1.2 将mysql-apt-xxx.deb文件上传到虚拟机

第一步：创建software目录，用于存放各种软件的安装包

```bash
sudo mkdir /opt/software
```



![image-20260116111526997](images/image-20260116111526997.png)



第二步：将MySQL安装文件mysql-apt-config_0.8.36-1_all.deb上传到/home/atguigu目录下

![image-20260116112013214](images/image-20260116112013214.png)

第三步：将mysql-apt-config_0.8.36-1_all.deb移动到/opt/software

```bash
sudo mv /home/atguigu/mysql-apt-config_0.8.36-1_all.deb /opt/software
```

![image-20260116120005645](images/image-20260116120005645.png)

### 2.1.3 安装mysql-apt-xxx.deb源配置包

安装 **MySQL 官方的 APT 源配置包**，给你的 Linux 系统「添加 MySQL 官方的软件源地址」，让你的`apt`命令可以下载安装 **MySQL 官方原版的 MySQL-server/mysql-client**，而非系统默认的 MariaDB。

> 运行命令：

```bash
sudo dpkg -i /opt/software/mysql-apt-config_0.8.36-1_all.deb
```

- `dpkg` → Debian 系系统的底层包管理器，专门处理本地 `.deb` 格式的安装包；
- `-i` → `--install` 的简写，核心作用：**安装本地的 deb 包文件**；

![image-20260116120819044](images/image-20260116120819044.png)

![image-20260116120329474](images/image-20260116120329474.png)

![image-20260116120444793](images/image-20260116120444793.png)

### 2.1.4 从MySQL APT 源更新包信息

```bash
sudo apt update
```

### 2.1.5 安装Mysql服务

```bash
sudo apt install mysql-server
```

![image-20260116122029796](images/image-20260116122029796.png)

注意：用`apt install mysql-server`安装 MySQL，文件会被分散到这些固定路径：

- 可执行文件 → `/usr/bin`、`/usr/sbin`
- 配置文件 → `/etc/mysql`
- 数据库文件 → `/var/lib/mysql`
- 库文件 → `/usr/lib/x86_64-linux-gnu/mysql`

Linux 的 FHS 文件系统标准，就是让软件的「可执行、配置、数据、库」文件分类存放，保证系统稳定性，这也是包管理器的设计初衷。

### 2.1.6 设置root用户密码

#### 情况一：

没有先安装mysql-apt-xxx.deb，直接执行`sudo apt install mysql-server`安装过程中，弹出如下对话框，输入密码。此时实际安装的是Ubuntu系统自带的 MariaDB。

![image-20260116134913506](images/image-20260116134913506.png)

当出现Use Strong Password Encryption (RECOMMENDED)直接选ok就行

![image-20260116135305291](images/image-20260116135305291.png)

![image-20260116135429569](images/image-20260116135429569.png)

#### 情况二：

Oracle 官方原版 MySQL8.0彻底**取消了 `apt install` 过程中的「交互式密码设置弹窗」**，这是 Oracle 官方的刻意设计，目的是提升安全性。MySQL 会为 `root@localhost` 自动生成一个**临时随机密码**，并把这个密码写入到 **MySQL 的错误日志文件** 中，或直接是空密码。不再让用户手动设置简单密码，从根源避免弱密码风险。

通过如下命令查看是否有root临时密码：

```bash
sudo grep 'root@localhost' /var/log/mysql/error.log
```

- 有临时密码：![image-20260116140828875](images/image-20260116140828875.png)
- 无临时密码：![image-20260116140853609](images/image-20260116140853609.png)



### 2.1.7 查看MySQL服务状态

#### 1、查看MySQL服务状态

```bash
sudo systemctl status mysql
```

![image-20260116151258183](images/image-20260116151258183.png)



#### 2、使用systemctl查看报错（wsl问题）

```
atguigu@LAPTOP-AG8KORH9:~$sudo systemctl status mysql（报错）
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

解决办法：WSL2 启用 systemd

步骤如下（全程在 Ubuntu 终端执行）：

步骤1：编辑 WSL 配置文件，开启 systemd

```bash
sudo vi /etc/wsl.conf
```

步骤2：写入以下配置（直接复制粘贴，覆盖原有内容即可）

```bash
[boot]
systemd=true
```

步骤3：保存退出 vi（按`ESC`，再输入`:wq!`，回车）

步骤4：关闭 WSL 并重启（关键步骤，必须执行）

**打开 Windows 的「管理员 PowerShell」**，执行以下命令关闭所有 WSL 发行版：

```powershell
wsl --shutdown
```

然后重新打开 Ubuntu 终端即可。



### 2.1.8 修改root用户密码

#### 第一步：安全模式启动MySQL

```bash
# 1.停止MySQL
sudo systemctl stop mysql

# 2.创建socket目录（确保存在）
sudo mkdir -p /var/run/mysqld
sudo chown mysql:mysql /var/run/mysqld

# 3.启动安全模式
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

![image-20260116144721151](images/image-20260116144721151.png)

#### 第二步：修改root用户密码

🔴注意：新开一个终端窗口

```bash
# 4. 新开一个终端窗口，直接免密登录mysql（无需输入密码，回车即可）
mysql -uroot
```

登录后，执行如下SQL语句

```sql
# 5. 使用mysql系统库
use mysql

# 6.刷新权限表
FLUSH PRIVILEGES;

# 7.修改root密码（MySQL 8.0方法）
ALTER USER 'root'@'localhost' IDENTIFIED BY '你的新密码';
#ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

-- 如果上述失败（ERROR 1524 (HY000): Plugin 'auth_socket' is not loaded），尝试传统方法。
UPDATE mysql.user 
SET authentication_string='', 
    plugin='mysql_native_password'
WHERE user='root' AND host='localhost';

-- 然后设置密码
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的新密码';
#ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456';

# 8.刷新权限表
FLUSH PRIVILEGES;

#退出
EXIT;
```

![image-20260116144814078](images/image-20260116144814078.png)

#### 第三步：重新启动

```bash
# 重新启动mysql
sudo systemctl start mysql
```

如果卡住请执行如下命令，再重启：

```bash
# 按 Ctrl+C 中断当前卡住的命令

# 强制停止MySQL服务
sudo systemctl stop mysql

# 确保所有MySQL进程都停止
sudo pkill -9 mysql
sudo pkill -9 mysqld
sudo pkill -9 mysqld_safe
```

## 2.2 连接Linux的MySQL服务

### 2.2.1 Linux本地登录

```bash
mysql -uroot -p
Enter password:输入密码
```

![image-20260116154626878](images/image-20260116154626878.png)

### 2.2.2 授权其它客户端服务器的权限

在虚拟机上打开终端，登录mysql

```bash
mysql -uroot -p
Enter password:输入密码
```

执行如下SQL语句：

```sql
# 更新用户表
update mysql.user set host='%' where user='root';
# 刷新权限表
FLUSH PRIVILEGES;
```

![image-20260116154644450](images/image-20260116154644450.png)

### 2.2.3 在windows上连接虚拟机的MySQL

```cmd
mysql -h虚拟机主机IP地址 -uroot -p
Enter password:输入密码
```

![image-20260116154831297](images/image-20260116154831297.png)



## 2.3 重新安装MySQL

```bash
# 停止服务
sudo systemctl stop mysql
sudo systemctl stop mysqld

# 确保进程停止
sudo pkill -9 mysql
sudo pkill -9 mysqld

# 备份配置和数据
# sudo cp -r /etc/mysql /etc/mysql_backup
# sudo cp -r /var/lib/mysql /var/lib/mysql_backup

# 重新安装MySQL（Ubuntu/Debian）
sudo apt-get purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-*
sudo rm -rf /etc/mysql /var/lib/mysql
sudo apt-get autoremove
sudo apt-get autoclean

# 重新安装
sudo apt-get update
sudo apt-get install mysql-server

# 启动服务
sudo systemctl start mysql
```



## 2.4 卸载MySQL

```shell
#!/bin/bash
# MySQL完全卸载脚本（Ubuntu/Debian）

echo "=== 开始卸载MySQL ==="

# 备份提醒
echo "警告：这将删除所有MySQL数据！"
read -p "是否已备份重要数据？(y/n): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "请先备份数据再执行卸载！"
    exit 1
fi

# 停止服务
echo "停止MySQL服务..."
sudo systemctl stop mysql 2>/dev/null
sudo systemctl stop mysqld 2>/dev/null
sudo pkill -9 mysql 2>/dev/null
sudo pkill -9 mysqld 2>/dev/null

# 卸载软件包
echo "卸载MySQL软件包..."
sudo apt-get remove --purge mysql-server mysql-client mysql-common mysql-community-server mysql-community-client -y
sudo apt-get autoremove --purge -y
sudo apt-get autoclean -y

# 删除目录
echo "删除MySQL文件和目录..."
sudo rm -rf /etc/mysql /etc/my.cnf /etc/my.cnf.d
sudo rm -rf /var/lib/mysql /var/lib/mysql-files /var/lib/mysql-keyring
sudo rm -rf /var/log/mysql /var/log/mysqld.log
sudo rm -rf /tmp/mysql* /tmp/.mysql*
sudo rm -rf /var/run/mysqld /run/mysqld
sudo rm -rf /usr/lib/mysql /usr/share/mysql /usr/share/doc/mysql*

# 清理用户
echo "清理MySQL用户..."
sudo userdel -r mysql 2>/dev/null
sudo groupdel mysql 2>/dev/null

# 最终清理
echo "最终清理..."
sudo apt-get update
sudo apt-get clean

echo "=== MySQL卸载完成 ==="
echo "建议重启系统: sudo reboot"
```













