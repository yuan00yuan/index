+++
title = "Guide: S3FS快速部署"
date = 2019-09-09T10:04:24+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

您可以启动Quick Start，将 S3fs 部署到 AWS 账户中。完成部署需要约 5 分钟。请查看下述实施详细信息，按照此指南后面部分提供的分步说明进行操作。

[![Image link china](http://cdn.quickstart.org.cn/assets/ChinaRegion.png)](https://console.amazonaws.cn/cloudformation/home?region=cn-north-1#/stacks/new?stackName=S3FS&templateURL=https://s3.cn-northwest-1.amazonaws.com.cn/aws-quickstart/quickstart-s3fs/template/s3fs-revised.template) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[![Image link global](http://cdn.quickstart.org.cn/assets/GlobalRegion.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=S3FS&templateURL=https://s3-us-west-2.amazonaws.com/chinalabs/s3fs-fixed.template)


## 前提条件        

需要在目标区域，提前创建好一个可用的S3桶用于挂载。cloudformation不会自动生成此通。
且需注意：最终定义的EBS卷的容量需要大于S3桶的总大小。可通过下列方法调用查看桶的大小.
![](http://cdn.quickstart.org.cn/assets/s3fs/get-total-size.png)

## 实现步骤      

****步骤一：启动资源****      

- 在您的 AWS 账户中启动 AWS CloudFormation 模板。

![](http://cdn.quickstart.org.cn/assets/s3fs/01.png)

- 请确保您的账户有一个VPC和私网，如 [Amazon VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_DHCP_Options.html)中所述。
- 检查导航栏右上角显示的所在区域，根据需要进行更改。
- 在 **Select Template** 页面上，保留模板 URL 的默认设置，然后选择 **Next** 。

![](http://cdn.quickstart.org.cn/assets/s3fs/02.png)

- 在 **Specify Details** 页面上，填写堆栈名称及相关参数，根据需要进行更改（见下表）。完成后选择 **Next** 进入下一步。   
**部署S3FS的配置参数**   


| 参数标签 | 参数名称 | 默认值 | 说明 |
| --- | --- | --- | --- |
| VPC | VpcId | _需要输入_ | 将部署S3FS实例的 VPC的ID (例如，vpc-0343606e)。 |
| 子网 | SubnetId | _需要输入_ | 要部署S3FS实例的 VPC 中现有子网的 ID (例如，subnet-a0246dcd)。 |
| 密钥名称 | KeyName | _需要输入_ | 公有/私有密钥对，使您能够在实例启动后安全地与它连接。|
| S3 存储桶名称 | BucketName | _需要输入_ | S3FS实例希望挂载的S3 存储桶(需提前创建好) |
| 实例类型 | InstanceType | _可选_ | 选择您S3FS实例的实例类型。|
| EBS卷容量 | VolumeSize | _需要输入_ | 选择您实例EBS卷的容量（大于S3 bucket大小）|
| 安全组 | S3FSSecurityGroup | _需要输入_ | 选择您的安全实例(需要保证在选择的VPC下)|
| 目录 | S3FSDirectory | _需要输入_ | 输入需要与您S3相连的实例的目录。（例如，/mnt/s3）|

截图如下：

![](http://cdn.quickstart.org.cn/assets/s3fs/03.png)

- 在 **Options** 页面上，您可以为堆栈中的资源 [指定标签](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-resource-tags.html) (键/值对) 并 [设置高级选项](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-add-tags.html)。在完成此操作后，选择 **Next** 。
- 在 **Review** 页面上，查看并确认模板设置。选择 **Capabilities** 下的复选框，以确认模板将创建 IAM 资源。

![](http://cdn.quickstart.org.cn/assets/s3fs/04.png)

- 选择 **Create** 以部署堆栈。
- 监控堆栈的状态。当状态为 **CREATE\_COMPLETE** 时 (如图4所示)，表示群集已准备就绪。在output当中，可以看到用于ssh连接的EC2地址

![](http://cdn.quickstart.org.cn/assets/s3fs/05.png)

****步骤二 连接到实例****
- 当 AWS CloudFormation 模板成功创建堆栈后，您需要通过SSH连接AWS账户中已安装的S3fs实例。
- 进入目录(例： **/mnt/s3mnt** )，可发现实例已经与您的S3桶相连。

![](http://cdn.quickstart.org.cn/assets/s3fs/06.png)


##	限制	
利用S3fs可以方便的把S3存储桶挂载在用户本地操作系统目录中，但是由于S3fs实际上是依托于Amazon S3服务提供的目录访问接口，所以不能简单的把S3fs挂载的目录和本地操作系统目录等同使用。用户使用S3f3挂载S3存储桶和直接访问S3服务有类似的使用场景。适用于对不同大小文件对象的一次保存（上传），多次读取（下载）。不适用于对已保存文件经常做随机修改，因为每次在本地修改并保存文件内容都会导致S3fs上传新的文件到Amazon S3去替换原来的文件。从访问性能上来说，通过操作系统目录方式间接访问Amazon S3存储服务的性能不如直接使用SDK或CLI接口访问效率高。以本地配置文件方式保存访问密钥的安全性也不如使用EC2 IAM角色方式高。
关于S3fs使用时候需要注意的更多细节，请参考下面s3fs官网内容：  
通常S3不能提供与本地文件系统相同的性能或语义。进一步来说：
*   随机写入或追加到文件需要重写整个文件
*   元数据操作比如列出目录会因为网络延迟原因导致性能较差
*	最终一致性设计可能临时导致过期数据
*	没有对文件或目录的原子重命名功能
*	挂载相同存储桶的多个客户端之间没有相互协调机制
*	不支持硬链接

