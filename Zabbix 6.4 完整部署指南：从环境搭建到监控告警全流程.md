# Zabbix 6.4 完整部署指南：从环境搭建到监控告警全流程

在Linux服务器运维工作中，监控系统是保障服务稳定运行的核心工具。Zabbix作为一款开源、企业级的监控解决方案，支持对服务器、网络设备、应用程序等进行全方位监控，还能通过邮件、钉钉等方式推送告警，广泛应用于中小规模企业的运维场景。

本文将带大家从零开始部署Zabbix 6.4监控系统，涵盖「服务端部署（LAMP/LNMP环境）、数据库配置、Zabbix初始化、客户端部署、监控模板配置、告警设置」全流程，步骤详细且附带问题排查方案，新手也能轻松上手。

## 一、部署前准备

### 1. 环境说明

本次部署采用「CentOS 7.9」系统，Zabbix版本为6.4（最新稳定版），架构为「服务端+客户端」：

- 服务端：负责数据采集、存储、展示及告警，需要部署Zabbix Server、Web前端、数据库（MySQL/MariaDB）

- 客户端：部署在被监控主机上，通过Zabbix Agent采集主机资源数据并上报给服务端

服务器配置建议：服务端至少2核4G内存（监控主机数<100），客户端无特殊要求（1核1G即可）。

### 2. 依赖工具安装

先安装部署所需的基础工具，关闭防火墙和SELinux（生产环境可按需配置规则，新手建议先关闭）：

```bash

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭SELinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 安装基础依赖
yum install -y wget vim net-tools gcc
```

## 二、服务端部署（核心步骤）

Zabbix 6.4服务端依赖「Web服务（Apache/Nginx）+ 数据库（MySQL 8.0/MariaDB 10.5+）+ Zabbix Server」，本文采用「LAMP环境（Apache+MariaDB）」部署，兼容性更优。

### 1. 安装并配置MariaDB数据库

Zabbix需要数据库存储监控数据，推荐使用MariaDB（MySQL的分支，开源免费）：

```bash

# 1. 安装MariaDB 10.5（CentOS 7默认版本较低，需配置官方源）
cat > /etc/yum.repos.d/MariaDB.repo << EOF
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.5/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF

# 2. 安装MariaDB服务
yum install -y MariaDB-server MariaDB-client

# 3. 启动MariaDB并设置开机自启
systemctl start mariadb
systemctl enable mariadb

# 4. 初始化数据库（设置root密码，删除空用户和测试数据库）
mysql_secure_installation
# 执行后按提示操作：
# - 输入当前root密码（默认空，直接回车）
# - 输入Y设置root密码（建议设为复杂密码，如Zabbix@123456）
# - 后续所有提示都输入Y（删除匿名用户、禁止root远程登录、删除test库、刷新权限）
```

创建Zabbix专用数据库和用户，授权访问：

```bash

# 登录MariaDB
mysql -u root -p
# 输入刚才设置的root密码

# 2. 创建数据库（编码必须为utf8mb4，支持中文）
CREATE DATABASE zabbix character set utf8mb4 collate utf8mb4_bin;

# 3. 创建Zabbix用户并授权（用户zabbix，密码Zabbix@123456，允许本地访问）
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'Zabbix@123456';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;

# 4. 退出数据库
exit
```

### 2. 安装Zabbix Server和Web前端

Zabbix官方提供了YUM源，直接配置后安装即可：

```bash

# 1. 配置Zabbix 6.4 YUM源
rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/7/x86_64/zabbix-release-6.4-1.el7.noarch.rpm
yum clean all
yum makecache fast

# 2. 安装Zabbix Server、Web前端（Apache版本）、Agent
yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-agent
```

### 3. 初始化Zabbix数据库

Zabbix官方提供了数据库初始化脚本，直接导入即可完成表结构和初始数据的创建：

```bash

# 导入初始化脚本（脚本路径为/usr/share/zabbix-sql-scripts/mysql/server.sql.gz）
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
# 输入Zabbix数据库用户的密码（即刚才设置的Zabbix@123456）
```

注意：导入过程可能需要1-2分钟，耐心等待即可，不要中断执行。

### 4. 配置Zabbix Server

修改Zabbix Server配置文件，指定数据库连接信息：

```bash

# 编辑配置文件
vim /etc/zabbix/zabbix_server.conf

# 找到以下配置项，修改为实际数据库信息
DBName=zabbix          # 数据库名（已创建）
DBUser=zabbix          # 数据库用户（已创建）
DBPassword=Zabbix@123456  # 数据库密码（刚才设置的）

# 其他核心配置（默认即可，新手无需修改）
ListenPort=10051       # Zabbix Server监听端口
LogFile=/var/log/zabbix/zabbix_server.log  # 日志路径
```

### 5. 配置Zabbix Web前端（Apache）

Zabbix Web前端依赖PHP环境，官方已通过zabbix-apache-conf包配置好Apache和PHP，只需修改PHP时区即可：

```bash

# 编辑PHP配置文件
vim /etc/httpd/conf.d/zabbix.conf

# 找到date.timezone配置项，修改为Asia/Shanghai（中国时区）
php_value date.timezone Asia/Shanghai
```

### 6. 启动服务端所有服务

```bash

# 启动Zabbix Server、Zabbix Agent、Apache
systemctl start zabbix-server zabbix-agent httpd

# 设置开机自启
systemctl enable zabbix-server zabbix-agent httpd

# 检查服务状态（确保所有服务都是active状态）
systemctl status zabbix-server zabbix-agent httpd
```

若Zabbix Server启动失败，可查看日志排查问题：`tail -f /var/log/zabbix/zabbix_server.log`

## 三、Zabbix Web前端初始化配置

服务端启动后，通过浏览器访问Web前端，完成初始化配置：

### 1. 访问Web界面

浏览器输入地址：`http://服务端IP/zabbix`，进入Zabbix初始化向导：

- 步骤1：点击「Next step」，进入环境检查页面（确保所有检查项都是“OK”，若有异常需修复对应依赖）

- 步骤2：数据库配置，输入刚才创建的Zabbix数据库信息（数据库类型选MySQL，密码为Zabbix@123456），点击「Next step」

- 步骤3：Zabbix Server配置（默认即可，Server名称可自定义，如“Zabbix监控系统”），点击「Next step」

- 步骤4：确认配置信息，点击「Next step」，完成初始化

- 步骤5：点击「Finish」，进入登录页面

### 2. 登录Zabbix后台

默认管理员账号：`Admin`，默认密码：`zabbix`（登录后建议立即修改密码）。

首次登录后，系统会自动跳转到Zabbix控制台，此时服务端部署完成。

## 四、客户端部署（被监控主机）

客户端需安装Zabbix Agent，通过Agent采集主机CPU、内存、磁盘等资源数据，上报给服务端。

### 1. 客户端环境准备

被监控主机需关闭防火墙/开放10050端口（Agent默认端口），并配置服务端IP（确保客户端能ping通服务端）。

### 2. 安装Zabbix Agent

```bash

# 1. 配置Zabbix YUM源（和服务端一致）
rpm -Uvh https://repo.zabbix.com/zabbix/6.4/rhel/7/x86_64/zabbix-release-6.4-1.el7.noarch.rpm
yum clean all
yum makecache fast

# 2. 安装Zabbix Agent
yum install -y zabbix-agent
```

### 3. 配置Zabbix Agent

```bash

# 编辑Agent配置文件
vim /etc/zabbix/zabbix_agentd.conf

# 核心配置项修改（重点！）
Server=服务端IP        # 指定Zabbix Server的IP（允许该IP采集数据）
ServerActive=服务端IP:10051  # 主动上报数据的服务端地址和端口
Hostname=Client-01     # 客户端主机名（需和服务端添加主机时的名称一致）

# 其他默认配置（无需修改）
ListenPort=10050       # Agent监听端口
LogFile=/var/log/zabbix/zabbix_agentd.log  # 日志路径
```

### 4. 启动Zabbix Agent

```bash

# 启动Agent并设置开机自启
systemctl start zabbix-agent
systemctl enable zabbix-agent

# 检查Agent状态
systemctl status zabbix-agent

# 验证端口是否监听（10050端口）
netstat -tnlp | grep 10050
```

## 五、服务端添加被监控主机

客户端部署完成后，需在服务端Web界面添加主机，才能开始监控：

### 1. 添加主机

1. 登录Zabbix后台，点击左侧「配置」→「主机」→「创建主机」

2. 填写主机信息：
        

    - 主机名称：需和客户端配置文件中的Hostname一致（如Client-01）

    - 可见名称：自定义（如“测试客户端-192.168.1.100”）

    - 群组：选择“Linux servers”（或自定义群组）

    - 接口：点击「添加」，选择「Agent」，IP地址填客户端IP，端口10050

3. 点击「模板」→「选择」，搜索“Linux by Zabbix agent”（Zabbix官方Linux监控模板），添加到右侧

4. 点击「添加」，完成主机添加

### 2. 验证监控数据

添加完成后，等待5-10分钟（Agent默认1分钟上报一次数据），点击左侧「监测」→「最新数据」，选择刚添加的主机，即可看到CPU使用率、内存使用、磁盘空间等监控数据。

若未获取到数据，可排查：① 客户端Agent是否正常运行；② 服务端和客户端网络是否通畅（telnet 客户端IP 10050）；③ 主机名称是否一致。

## 六、配置告警（邮件/钉钉）

监控的核心是告警，当资源使用率超标时（如CPU>80%、磁盘>90%），Zabbix需自动推送告警。下面以「邮件告警」为例，讲解配置流程：

### 1. 配置邮件服务器

1. 点击左侧「管理」→「报警媒介类型」→「创建媒体类型」

2. 填写配置：
        

    - 名称：自定义（如“QQ邮件告警”）

    - 类型：选择“Email”

    - SMTP服务器：smtp.qq.com（QQ邮箱SMTP服务器）

    - SMTP服务器端口：465（SSL加密端口）

    - SMTP HELO：qq.com

    - SMTP邮箱：你的QQ邮箱（如123456@qq.com）

    - 认证：选择“用户名和密码”

    - 用户名：你的QQ邮箱账号

    - 密码：QQ邮箱授权码（不是登录密码，需在QQ邮箱设置中开启SMTP并获取）

    - 连接安全：选择“SSL/TLS”

3. 点击「测试」，输入接收告警的邮箱，发送测试邮件，验证配置是否正确

4. 点击「添加」，完成邮件媒介配置

### 2. 配置告警接收人

1. 点击左侧「管理」→「用户」，编辑管理员账号（Admin）

2. 点击「报警媒介」→「添加」，选择刚才创建的邮件媒介，输入接收告警的邮箱，点击「更新」

3. 点击「权限」，确保用户有对应的主机监控权限

### 3. 配置告警触发器

使用官方模板的默认触发器（如CPU>80%、磁盘>90%）即可，也可自定义触发器：

1. 点击左侧「配置」→「模板」，编辑“Linux by Zabbix agent”模板

2. 点击「触发器」，可查看或修改默认触发器（如修改磁盘使用率阈值为90%）

3. 自定义触发器：点击「创建触发器」，设置触发条件（如“系统负载>2”）、严重级别（如“警告”），点击「添加」

配置完成后，当监控指标超标时，Zabbix会自动发送告警邮件到指定邮箱。

## 七、常见问题排查

### 1. Zabbix Server启动失败，日志提示“Can't connect to MySQL server”

解决方案：① 检查MariaDB服务是否正常运行；② 确认zabbix_server.conf中的数据库密码是否正确；③ 检查Zabbix用户是否有数据库访问权限。

### 2. 服务端无法获取客户端监控数据

解决方案：① 客户端Agent服务是否正常；② 防火墙是否开放10050端口（客户端）和10051端口（服务端）；③ 客户端配置文件中的Server和ServerActive是否正确；④ 服务端添加主机时的Hostname是否和客户端一致。

### 3. 告警邮件发送失败

解决方案：① 检查SMTP服务器配置是否正确；② 确认邮箱授权码是否有效；③ 查看Zabbix日志（/var/log/zabbix/zabbix_server.log）中的告警相关错误信息。

## 八、总结

本文完整讲解了Zabbix 6.4监控系统的部署流程，从服务端环境搭建、数据库配置，到客户端部署、监控主机添加，再到告警配置，覆盖了运维实战中的核心需求。通过Zabbix，我们可以实现对Linux服务器的全方位监控，及时发现并处理资源过载、服务异常等问题，保障业务稳定运行。

后续可根据实际需求扩展功能：① 监控网络设备（交换机、路由器）；② 监控应用程序（Nginx、MySQL、Redis）；③ 配置钉钉/企业微信告警；④ 自定义监控模板和报表。




> （注：文档部分内容可能由 AI 生成）