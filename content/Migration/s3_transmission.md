+++
title = "基于Amazon ECS的国内外文件同步系统解决方案"
date = 2019-09-09T10:26:56+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

很多跨国企业都会有数据同步的需求，这些大型文件主要是存储在S3桶上，通过console或者命令行的形式从本地上传至S3中。本文描述了一种基于容器化的，在Amazon ECS平台上搭建的、快速同步的且基于负载进行扩容和收缩的、通用的解决方案。
### 解决方案概述
该方案如下图所示。
 ![图片1](http://cdn.quickstart.org.cn/assets/s3_transmission/archi.png)
在该方案中，我们会启动以下资源：
1. 一个global的S3桶：用于global作为数据传输的源
2. 4个Lambda函数： 分别用于文件分片，容器的扩张与收缩，发送传输完成信号
3. 1个sqs队列：用于存储传输所用参数
4. 2个cloud watch警报： 监控sqs内消息数量，分别用于需要收缩和扩张容器
5. 一个ECS集群，一个服务和若干个task：用于搭载容器
6. 两个dynamoDB 表： T 和C。表T用于记录传输的uplaodID和分段，传输情况，表C用于记录每个片段的唯一标记值-etag。
7. 具有相应的权限的角色。

[![Image link global](http://cdn.quickstart.org.cn/assets/GlobalRegion.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=ecsStack&templateURL=https://s3-us-west-2.amazonaws.com/chinalabs/templates/s3Transmission/ecs.yaml)
[![Image link global](http://cdn.quickstart.org.cn/assets/GlobalRegion.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=lamStackS&templateURL=https://s3-us-west-2.amazonaws.com/chinalabs/templates/s3Transmission/lambda.yaml)

**在运行cloudformation之前需要您准备好：**
国内：一个S3桶，用户的访问密钥 AccessId 和AccessKey
国外：一个VPC和其中的一个公有子网， EC2密钥对，用于接收信息的Email地址
### 步骤：
1. 用户在global bucket中成功上传文件。
2. 触发lambda函数。Lambda函数会判断新文件的大小判断用哪种方式传输，当小于5M的时候会选择直接进行下载上传，当大于5M的时候会进行分段上传。
3. 当确定用哪种方式进行传输之后会把相应的信息存储至sqs中，包括文件名称，uploadID, 分段情况等。
4. 同时Lambda会将传输情况备份至dynamoDB表C中,表1用于记载传输情况。
5. 容器运行在ecs集群中， SQS内的信息被ecs中的servces和task消耗，
6. 我们为SQS设定了cloudwatch监控，用于根据负载实现容器的扩张和缩小。
7. 我们创建了警报A 用于判断是否需要根据情况扩张和缩小容器的数量，当sqs内消息数量较大时会触发警报A
8. 警报A会触发发送消息给SNS
9. SNS接到消息之后会触发LambdaA
10. LambdaA会调整容器的数量到10个，实现容器的扩张
11. 当sqs内消息数量较小时会触发警报B
12. 警报B会触发发送消息给SNS
13. SNS接到消息之后会触发LambdaB
14. LambdaB会调整容器的数量到1个，实现容器的收缩
15. 容器会将下载的片段传至国内S3桶。
16. 容器更改dynamo DB表C中的数据，更新已完成的片段数量
17. 容器更改dynamo DB 表P，插入新的数据存入etag和uplaod id partnumber等信息。
18. 当表C中已完成的片段数量和公有片段数量一致的时候会触发Lambda函数
19. 函数搜索同一个uploadID的所有的片段的etag 并按partnumber排序，发送给国内的S3桶。S3桶在核对所有的etag之后进行片段组装，出现在国内的S3桶上。
### 架构部署
1. 登录 AWS 管理控制台并通过以下网址打开 AWS CloudFormation 控制台：https://console.aws.amazon.com/cloudformation。
2. 如果这是新的 AWS CloudFormation 账户，请单击“Create New Stack”。否则，请单击“Create Stack”。
3. 在模板部分，选择指定 Amazon S3 模板 URL
4. 在 Specify Details 部分的 Name 字段中，输入堆栈名称。堆栈名中不得含有空格。
5. 在 Specify Parameters (指定参数) 页上，您将看到模板的 Parameters 部分中的参数。 请按实际情况填写。
6. 单击 Next (下一步)。
7. 在这种情况下，我们不会添加任何标签。单击“Next”。作为密钥值对的标签可帮助您识别堆栈。
8. 审查堆栈信息。如果满意该设置，则单击“Create ”。

