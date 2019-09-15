+++
title = "Aurora VS MySQL 压测"
date = 2019-09-09T10:11:17+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

## 实验目的
> 本实验大约耗时**35分钟**

使用 [sysbench](https://github.com/akopytov/sysbench) 对Amazon Aurora与RDS MySQL进行基准测试, 对比二者的读写性能.

以下是本次测试的架构

![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/architecture.jpg)

**涉及组件**
- Aurora
- RDS
- EC2

## 实验步骤

### 配置 VPC (可选)

新建 VPC 安全组，具体步骤参考[适用于 Amazon VPC 的 IPv4 入门](https://docs.aws.amazon.com/zh_cn/AmazonVPC/latest/UserGuide/getting-started-ipv4.html#getting-started-create-security-group)

将安全组的**入站规则**设置为

- **Type**: ALL Traffice
- **Protocol**: ALL
- **Port Range**: ALL
- **Source**: 选择 **Custom IP**，然后键入 `0.0.0.0/0`。

> **重要**
>
> 除演示之外，建议不要使用 0.0.0.0/0，因为它允许从 Internet 上的任何计算机进行访问。在实际环境中，您需要根据自己的网络设置创建入站规则。

### 启动Aurora实例

1. 登录 AWS 管理控制台 并通过以下网址打开 Amazon RDS 控制台：<https://console.aws.amazon.com/rds/>。 

2. 在导航窗格中，选择**实例**。 

3. 选择**启动数据库实例**。**启动数据库实例向导**在**选择引擎**页面打开。

   ![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/aurora-1.png)

4. 选择 **Amazon Aurora**, 并将**版本**选择为**与 MySQL 5.6 兼容**，然后选择**下一步**。

5. 在**指定数据库详细信息**页面上，指定数据库实例信息。选择下列值，然后选择 **下一步**。
   ![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/aurora-2.png)

   - **数据库实例类**: **db.r4.xlarge**
   - **多可用区部署**: **是**
   - **数据库实例标识符**: 例如`my-aurora`
   - **主用户名**: 例如`root`
   - **主密码**: 用户自定义

6. 在**配置高级设置**页面上，提供 RDS 启动 MySQL 数据库实例所需的其他信息。选择下列值，然后选择 **下一步**。

   - **Virtual Private Cloud (VPC)**：选择您在**配置 VPC**这一步骤中创建的安全组所对应的 VPC
   - **公开可用性**：否
   - **可用区**: 根据您的实际区域来更改, 但请保证之后的MySQL与EC2都在这一区域中
   - **VPC安全组**：创建新的VPC安全组
   - **数据库集群标识符**: `my-mysql-cluster`
   - **数据库名称**：`dbname`
   - **备份保留期**：1 天
   - **加密**: **禁用加密**
   - **备份保留期**: **1天**
   - **监控**: **禁用增强监控**
   - **其他请选择默认**

### 启动MySQL实例

1. 登录AWS管理控制台,并通过以下网址打开Amazon RDS控制台：<https://console.aws.amazon.com/rds/>。 

2. 在导航窗格中，选择**实例**。 

3. 选择**启动数据库实例**。**启动数据库实例向导**在**选择引擎**页面打开。

4. 选择 **MySQL**，然后选择**下一步**。

5. **选择使用案例**页面询问您是否计划使用所创建的数据库实例进行生产。选择 **生产**，然后选择 **下一步** 。
![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/mysql-1.png)  

6. 在**指定数据库详细信息**页面上，指定数据库实例信息。选择下列值，然后选择 **下一步**。 

   - **数据库引擎版本**: **MySQL 5.6.41** 
   - **数据库实例类**：**db.r4.xlarge**
   - **多可用区部署**：**是**
   - **存储类型**：**预置IOPS(SSD)**
   - **分配的存储空间**：**100**GiB
   - **预置 IOPS**: `1000`
   - **数据库实例标识符**：键入 `my-mysql`。 
   - **主用户名**：键入 `root`。 
   - **主密码**和**确认密码**：用户自定义
   - **其他设置保持默认**

![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/mysql-2.png)

1. 在**配置高级设置**页面上，提供 RDS 启动 MySQL 数据库实例所需的其他信息。选择下列值，然后选择 **下一步**。

   - **Virtual Private Cloud (VPC)**：选择Aurora使用的安全组
   - **公开可用性**：否
   - **可用区**: 与之前 Aurora 的可用区相同
   - **VPC安全组**：选择现有 VPC 安全组，并且选择**配置 VPC**这一步骤中创建的安全组
   - **数据库名称**：键入`dbname`
   - **备份保留期**：1 天
   - **其他请选择默认**
   
![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/mysql-3.png)  


*启动Aurora Multi-AZ部署方式，可以在控制台看到2个数据库实例；启动RDS MySQL Multi-AZ部署方式，在控制台只能看到一个主实例*

### 启动压测机EC2

1. 启动实例

   - 打开 Amazon EC2 控制台 <https://console.aws.amazon.com/ec2/>。选择您要在其中创建EC2实例的区域。这里请保证与之前创建Aurora,MySQL 的区域相同。 

   - 从控制台控制面板中，选择 **启动实例**。

   - **Choose an Amazon Machine Image (AMI)** 页面显示一组称为 *Amazon 系统映像 (AMI)* 的基本配置，作为您的实例的模板。选择 Amazon Linux AMI 2 的 HVM 版本 AMI。 

   - 在**选择实例类型** 页面上，您可以选择实例的硬件配置。**实例的数量**输入`4`并选择 `r4.xlarge` 类型

   - 在**配置实例详细信息**页面上，自动分配公有 IP 选择**启用**，并保证选择子网和刚才的数据库在同一个可用区，点击**下一步**
   
   ![](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/ec2-1.png)

   - 在**添加存储**页面上, 点击**下一步**
   
   - 在**添加标签**页面上，点击**下一步**

   - 在**配置安全组**页面选择**选择一个现有的安全组**，或者创建一个新的安全组，

   - 在**审核**页面选择**启动**

   - 当系统提示提供密钥时，选择 **选择现有的密钥对**，然后选择合适的密钥对。若没有创建密钥对，请参考[创建密钥对](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html#create-a-key-pair)

     准备好后，选中确认复选框，然后选择 **启动实例**。  


### 安装Sysbench

请参考[使用 SSH 连接到 Linux 实例](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

**在连接到 EC2 实例后**，依次输入以下命令，在实例中安装`sysbench`, 您需要在4台EC2上分别执行如下命令

下载`sysbench`
```shell
sudo yum -y install bzr automake libtool mysql mysql-devel
bzr branch lp:sysbench 
```
编译和安装`sysbench`
```shell
cd sysbench
./autogen.sh
./configure
make
cd sysbench
```

### 进行Sysbench测试

* 准备数据

只需要在一台EC2上执行以下命令，为`sysbench`测试准备数据；您在为MySQL准备数据的同时，可以在另外一台机器上为Aurora准备数据。

请根据您的实际情况替换下面命令中的`mysql-host`, `mysql-user`, `mysql-password`,`mysql-db`参数

```shell
./sysbench --test=tests/db/oltp.lua --mysql-host=<rds-mysql-instancehost-name> \
--mysql-port=3306 --mysql-user=<db-username> --mysql-password=<db-password> \
--mysql-db=<db-name> --mysql-table-engine=innodb --oltp-table-size=25000 \
--oltp-tables-count=250 --db-driver=mysql prepare
```

* 执行read heavy测试

> 请在4台EC2上同时执行以下命令

```shell
./sysbench --test=tests/db/oltp.lua --mysql-host=<rds-aurora-instancehost-name> \
--oltp-tables-count=250 --mysql-user=<db-username> --mysql-password=<db-password> \
--mysql-port=3306 --db-driver=mysql --oltp-tablesize=25000 \
--mysql-db=<db-name> --max-requests=0 --oltp_simple_ranges=0 --oltp-distinct-ranges=0 \
--oltp-sum-ranges=0 --oltp-order-ranges=0 --max-time=300 \
--oltp-read-only=on --num-threads=300 run
```

以下是MySQL测试结果，我们需要将以下4台主机的测试结果相加才是我们的最终结果

![mysql测试结果](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/msyql-read.png)

以下是Aurora的测试结果

![Aurora测试结果](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/aurora-read.png)



* 执行write heavy测试

> 请在4台EC2上同时执行以下命令

```shell
./sysbench --test=tests/db/oltp.lua --mysql-host=<rds-aurora-instancehost-name> \
--oltp-tables-count=250 --mysql-user=<db-username> --mysql-password=<db-password> \
--mysql-port=3306 --db-driver=mysql --oltp-tablesize=25000 \
--mysql-db=<db-name> --max-requests=0 --max-time=300 --oltp_simple_ranges=0 \
--oltp-distinct-ranges=0 --oltp-sum-ranges=0 --oltporder-ranges=0 \
--oltp-point-selects=0 --num-threads=450 --randtype=uniform run
```

以下是MySQL的测试结果

![MySQL测试结果](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/mysql-write.png)

以下是Aurora测试结果

![Aurora测试结果](http://cdn.quickstart.org.cn/assets/aurora-vs-mysql/aurora-write.png)


由于机型的差异，选择更大的机型，Aurora与MySQL的性能差异表现得更加明显。


## 更多阅读

[Amazon Aurora Performance Assessment](https://d0.awsstatic.com/product-marketing/Aurora/RDS_Aurora_Performance_Assessment_Benchmarking_v1-2.pdf)

[知己知彼-对Aurora进行压力测试](https://amazonaws-china.com/cn/blogs/china/aurora-test/)

[Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://www.allthingsdistributed.com/files/p1041-verbitski.pdf)


