# 防火墙 Firewalld

Firewalld 是 Linux 上的防火墙管理软件，仅仅是管理，防火墙的实现依然通过系统内核的 netfilter 来实现，与 iptables 类似，他们的作用都是用于维护规则，只是使用方式不同而已。相比而言，firewalld 可以动态修改单条规则，而不需要像 iptables 那样，在修改了规则后必须得全部刷新才可以生效。 

firewalld 将配置储存在 /usr/lib/firewalld/ 和 /etc/firewalld/ 中，firewalld 的配置方法主要有三种：

 - firewall-config 图形化管理工具
 - firewall-cmd 命令行工具
 - xml 文件，直接修改配置

**firewalld 中有一个重要的概念：区域管理**

通过将网络划分成不同的区域，制定出不同区域之间的访问控制策略来控制不同程序区域间传送的数据流。例如，互联网是不可信任的区域，而内部网络是高度信任的区域。网络安全模型可以在安装，初次启动和首次建立网络连接时选择初始化。该模型描述了主机所连接的整个网络环境的可信级别，并定义了新连接的处理方式。有如下几种不同的初始化区域，默认区域是public：

- 阻塞区域（block）：任何传入的网络数据包都将被阻止。

- 工作区域（work）：相信网络上的其他计算机，不会损害你的计算机。

- 家庭区域（home）：相信网络上的其他计算机，不会损害你的计算机。

- 公共区域（public）：不相信网络上的任何计算机，只有选择接受传入的网络连接。

- 隔离区域（DMZ）：隔离区域也称为非军事区域，内外网络之间增加的一层网络，起到缓冲作用。对于隔离区域，只有选择接受传入的网络连接。

- 信任区域（trusted）：所有的网络连接都可以接受。

- 丢弃区域（drop）：任何传入的网络连接都被拒绝。

- 内部区域（internal）：信任网络上的其他计算机，不会损害你的计算机。只有选择接受传入的网络连接。

- 外部区域（external）：不相信网络上的其他计算机，不会损害你的计算机。只有选择接受传入的网络连接。

firewalld默认提供了九个zone配置文件：block.xml、dmz.xml、drop.xml、external.xml、 home.xml、internal.xml、public.xml、trusted.xml、work.xml，他们都保存在 /usr/lib/firewalld/zones/ 目录下。

firewalld 服务基本管理

##### 服务管理

```bash
systemctl start firewalld.service # 启动服务
systemctl stop firewalld.service # 关闭服务
systemctl restart firewalld.service # 重启服务
systemctl status firewalld.service # 查看服务状态

# 关闭状态
Active: inactive (dead) since Wed 2018-10-03 18:21:01 CST; 31s ago
# 启动状态
Active: active (running) since Wed 2018-10-03 18:21:50 CST; 3s ago
```

##### 启动设置

```bash
systemctl enable firewalld.service # 设置开机启动
systemctl disable firewalld.service # 禁用开机启动
systemctl is-enabled firewalld.service # 查看服务是否开机启动
systemctl list-unit-files | grep enabled # 查看已启动的服务列表
systemctl --failed # 查看启动失败的服务列表
```

#### firewalld-cmd 配置防火墙

`firewall-cmd`命令需要`firewalld`进程处于运行状态。我们可以使用`systemctl status/start/stop/restart firewalld`来控制这个守护进程。



##### 命令信息

```bash
firewall-cmd --version # 查看版本
firewall-cmd --state # 显示状态
firewall-cmd --help # 查看帮助信息
```

##### 接口所属区域管理

```bash
firewall-cmd --get-zones # 显示支持的区域列表	
firewall-cmd --get-active-zones # 查看当前区域信息
firewall-cmd --get-zone-of-interface=eth0 # 查看指定接口所属区域
firewall-cmd --permanent --zone=public --add-interface=eth0  # 将接口添加到区域，默认接口都在public
firewall-cmd [--zone=<zone>] --remove-interface=eth0	# 从区域中删除一个接口
firewall-cmd [--zone=<zone>] --query-interface=eth0 # 查询区域中是否包含某接口
firewall-cmd --set-default-zone=public	# 设置默认接口区域，立即生效无需重启
firewall-cmd --zone=public --list-all # 显示所有公共区域（public）全部启用的特性，省略 zone 参数则显示默认区域的信息	
firewall-cmd --list-all-zones # 列出全部启用的区域的特性
firewall-cmd --permanent --zone=internal --change-interface=eth0 # 永久修改网络接口enp03s为内部区域（internal）
firewall-cmd --permanent --zone=internal --add-service=http # 添加HTTP服务到内部区域（internal）
```

##### 包管理

```bash
firewall-cmd --panic-on # 拒绝所有包
firewall-cmd --panic-off # 取消拒绝状态
firewall-cmd --query-panic # 查看是否拒绝
```

##### 重新加载火墙规则

```bash
firewall-cmd --reload # 无需断开连接，firewalld 特性之一动态添加规则
firewall-cmd --complete-reload # 类似重启服务，需要断开连接
```

##### 服务管理

```bash
firewall-cmd --get-services # 显示服务列表 
firewall-cmd --list-services # 显示当前服务
firewall-cmd --zone=work --add-service=http # 允许服务 permanent 参数表示永久生效，没有此参数重启后失效, zone 表示添加的区域
firewall-cmd --zone=work --remove-service=http # 禁止服务
firewall-cmd --enable service=smtp --timeout=60 # 临时允许某服务通过一段时间，单位为秒
firewall-cmd --permanent --zone=internal --add-service=http # 添加HTTP服务到内部区域（internal）
```

##### 添加自定义服务

```xml
<!-- 添加新文件 8000.xml 到 /etc/firewalld/services/目录 -->
<?xml version="1.0" encoding="utf-8"?>
<service>
    <short>Service Name</short>
    <description>description</description>
    <port protocol="tcp" port="8000"/>
</service>

<!-- 编辑 public.xml 文件,加入相应的Server /etc/firewalld/zones/public.xml -->
<?xml version="1.0" encoding="utf-8"?>
<zone>
    <short>Public</short>
    <description>xxxxx</description>
    <service name="dhcpv6-client"/>
    <service name="ssh"/>
    <!-- 新增此行，端口号与文件 /etc/firewalld/services/8000.xml 文件名相同 -->
    <service name="8000"/>
</zone>
```



##### 端口管理

```bash
firewall-cmd --permanent --zone=public --add-port=8080/tcp # permanent 参数表示永久生效，没有此参数重启后失效, zone 表示添加的区域
firewall-cmd --permanent --zone=public --add-port=8080-8090/tcp # 打开范围内的所有端口
firewall-cmd --permanent --zone= public --remove-port=8080/tcp	# 关闭端口
firewall-cmd --zone=public --query-port=8080/tcp # 查看端口状态
firewall-cmd --zone=public --list-ports	# 查看所有打开的端口
```

##### 伪装 IP

```bash
firewall-cmd --query-masquerade # 检查是否允许伪装IP
firewall-cmd --add-masquerade   # 允许防火墙伪装IP
firewall-cmd --remove-masquerade# 禁止防火墙伪装IP
```

##### 端口转发

端口转发可以隐藏端口和流量分发。在配置时，需要注意源端口和目标端口都必须开放监听，并且必须允许防火墙伪装IP，否则配置失败。

```bash
firewall-cmd --add-service=http # 允许 http 协议通过防火墙
firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8000 # 将80端口的流量转发至8000，如果不指定 ip 的话就默认为本机
firewall-cmd --permanent --add-forward-port=proto=80:proto=tcp:toaddr=192.168.1.123 # 将80端口的流量转发至192.168.1.123，指定了 ip 却没指定端口，则默认使用来源相同的端口
firewall-cmd --permanent --add-forward-port=proto=80:proto=tcp:toaddr=192.168.1.123:toport=8000 # 将80端口的流量转发至192.168.1.123的8000端口	
```

