+++
title = "基于CodePipeline ECS的CICD解决方案"
date = 2019-09-09T10:26:39+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

## 综述：
通过本文章，您可以通过AWS CodePipeline，AWS CodeBuild，AWS CodeCommit, AWS CloudFormation 来实现基于Amazon ECS的CICD方案。开发人员在AWS CodeCommit中提交的新版本代码，会自动触发代码获取，打包镜像，上传镜像仓库，创建新版本的任务定义和服务。
## 所需组件
- [Code Commit](https://docs.aws.amazon.com/zh_cn/codecommit/latest/userguide/welcome.html)：AWS CodeCommit 是由 Amazon Web Services 托管的版本控制服务，可让您在云中私下存储和管理资产（如文档、源代码和二进制文件）。
- Docker：CodeBuild使用Docker container 运行构建过程中需要的应用程序。
- Amazon EC2：作为ECS的容器宿主机集群。
- Amazon VPC：服务所在的网络。
- [Amazon ECS](http://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/Welcome.html)：AWS托管的容器编排服务。
- [Amazon ECR](http://docs.aws.amazon.com/zh_cn/AmazonECR/latest/userguide/what-is-ecr.html)：AWS 托管的容器镜像仓库。
- [AWS CodePipeline](http://docs.aws.amazon.com/zh_cn/codepipeline/latest/userguide/welcome.html)：AWS 托管的持续集成持续交付服务，可以快速可靠的更新应用程序和服务，集成支持GitHub，Jenkins等主流开源工具。
- [AWS CodeBuild](http://docs.aws.amazon.com/zh_cn/codebuild/latest/userguide/welcome.html)：AWS 托管的构建服务，用于打包代码生成可部署的软件包。
- [AWS CloudFormation](http://docs.aws.amazon.com/zh_cn/AWSCloudFormation/latest/UserGuide/Welcome.html)：批量创建和管理AWS资源的自动化脚本。

## 步骤
  
   ![图片1](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/cicd/codepipeline_ecs/flowchart.png)
   
### 流程
1.	开发者将一个新版本的代码工程提交到CodeCommit或者github  触发codepipeline 流程
2.      Pipeline的Source阶段，检测到指定CodeCommit的repo/ github 有新版本的更新 
3.      从CodeCommit / github 上拉取工程代码
4.	Pipeline的Build阶段，AWS CodeBuild将新版本的代码按照 buildspec 文件打包为jar包
5.      将打包好的jar包上传到S3
6.      上传到s3的jar包会作为下一步build的source
7.      Pipeline的Build阶段，上一部build结束之后会触发第二个构建步骤，本步骤将jar包打包成Docker镜像
8.      将Docker镜像上传到ECR镜像仓库
9.      ECR中的镜像会在下一步中作为容器镜像
10.	Pipeline的Deploy阶段，AWS CodePipeline触发AWS CloudFormation，其定义了Amazon ECS的Task definition和service
11.     AWS CloudFormation创建新版本的Task definition关联到新版本的Docker镜像，并更新Service。Amazon ECS从Amazon ECR中取到新版本的Docker镜像，创建新的 Task definition和服务，完成服务的更新
	
## 搭建
### 基础设施
#### 创建VPC，子网
1.	打开 Amazon VPC 控制台 https://console.aws.amazon.com/vpc/。 
2.	在控制面板上，选择 Create VPC (创建 VPC)。 
 ![图片2](http://cdn.quickstart.org.cn/assets/CICD/vpc-1.png)
3.	选择第一个选项，即 VPC with a Single Public Subnet，然后选择‘选择’。 
 ![图片3](http://cdn.quickstart.org.cn/assets/CICD/vpc-2.png)
4.	保留其余默认设置，然后选择 Create VPC (创建 VPC)。 
#### 安全组
创建 WebServerSG 安全组
1.	打开 Amazon VPC 控制台 https://console.aws.amazon.com/vpc/。 
2.	在导航窗格中，选择 Security Groups。 
3.	选择 Create Security Group。 
4.	提供安全组的名称和描述。在本主题中，使用名称 WebServerSG 作为示例。从 VPC 菜单中选择您 VPC 的 ID，然后选择 Yes, Create。 
5.	选择您刚刚创建的 WebServerSG 安全组。详细信息窗格内会显示此安全组的信息，以及可供您使用入站规则和出站规则的选项卡。
6.	在 Inbound Rules 选项卡上，选择 Edit，然后执行以下操作：
    - 从 Type (类型) 列表中选择 HTTP，然后在 0.0.0.0/0Source (源) 字段中输入 。 
    - 选择 Add another rule，从 Type 列表中选择 HTTPS，然后在 Source 字段中输入 0.0.0.0/0。     
    - 选择 Add another rule，然后从 Type 列表中选择 SSH (对于 Linux) 或 RDP (对于 Windows)。在 Source (源) 字段中输入您网络的公有 IP 地址范围。(如果不知道此地址范围，则可使用 0.0.0.0/0 作测试用途；在生产环境下，您将仅授权特定 IP 地址或地址范围访问您的实例。) 
    -  选择 Add another rule，然后从 Type 列表中选择 ALL traffic。在 Source 字段中，输入 WebServerSG 安全组的 ID。 
7.	选择 Save

#### Internet Gateway
创建 Internet 网关并将其附加到 VPC
1.	打开 Amazon VPC 控制台 https://console.aws.amazon.com/vpc/。 
2.	在导航窗格中，选择 Internet Gateways (Internet 网关)，然后选择 Create internet gateway (创建 Internet 网关)。 
 ![图片4](http://cdn.quickstart.org.cn/assets/CICD/ing-1.png)
3.	选择刚刚创建的 Internet 网关，然后选择 Actions, Attach to VPC
 ![图片5](http://cdn.quickstart.org.cn/assets/CICD/ing-2.png)
4.	从列表中选择 VPC，然后选择 Attach (附加)。 
 ![图片6](http://cdn.quickstart.org.cn/assets/CICD/ing-3.png)
#### 路由表
将Internet Gateway Attach到VPC上，路由表配置0.0.0.0/0指向Internet Gateway，并关联子网。
 ![图片7](http://cdn.quickstart.org.cn/assets/CICD/rtr.png)
之后的EC2宿主机集群，负载均衡器等都使用在这个网络里。
#### 负载均衡器
1. 创建ALB应用负载均衡器，监听80端口
  ![图片8](http://cdn.quickstart.org.cn/assets/CICD/alb-1.png)
![图片9](http://cdn.quickstart.org.cn/assets/CICD/alb-2.png)
2. 选择对应的子网
![图片10](http://cdn.quickstart.org.cn/assets/CICD/alb-3.png)
#### 新建安全组，端口80，  
![图片11](http://cdn.quickstart.org.cn/assets/CICD/sg.png)
#### 新建目标组
 ![图片12](http://cdn.quickstart.org.cn/assets/CICD/tg.png)
	注册目标此时不选择，ECS创建服务时会注册集群和对应端口进来。
下一步审核后创建。
#### ECS
##### ECS宿主机集群
在ECS的界面下，创建集群
 
 ![图片13](http://cdn.quickstart.org.cn/assets/CICD/ecs-1.png)
  ![图片14](http://cdn.quickstart.org.cn/assets/CICD/ecs-2.png)
实例配置保持默认或根据情况自行选择。
联网配置，选择创建好的VPC，子网，创建Role允许宿主机上的ECS代理调用ECS服务的API。
  ![图片15](http://cdn.quickstart.org.cn/assets/CICD/ecs-3.png)
  ![图片16](http://cdn.quickstart.org.cn/assets/CICD/ecs-4.png)
  ![图片17](http://cdn.quickstart.org.cn/assets/CICD/ecs-5.png)
创建后画面下面会显示集群信息
  ![图片18](http://cdn.quickstart.org.cn/assets/CICD/ecs-6.png)
集群一览会显示
  ![图片19](http://cdn.quickstart.org.cn/assets/CICD/ecs-7.png)
修改ECS宿主机集群的安全组，inbound源设置为建好的应用负载均衡器的安全组ID
  ![图片20](http://cdn.quickstart.org.cn/assets/CICD/ecs-8.png)
##### ECR镜像仓库
创建一个用于Build阶段上传存放软件工程Docker镜像的镜像仓库
  ![图片21](http://cdn.quickstart.org.cn/assets/CICD/ecs-9.png)
ECS界面下，创建存储库，创建好后如下
  ![图片22](http://cdn.quickstart.org.cn/assets/CICD/ecs-10.png)
#### S3桶创建
创建一个S3桶用来存放Deploy阶段CloudFormation使用的脚本模版，创建桶时选择和以上服务同一Region，并且打开桶的版本控制。
  ![图片23](http://cdn.quickstart.org.cn/assets/CICD/s3.png)
将CloudFormation模版压缩zip后上传到桶中。
将模版文件service.yaml放在templates文件夹后压缩为templates.zip。
url
### Pipeline的搭建
分为Source，Build以及Deploy三阶段：
1. Source阶段设置CodeCommit上的软件工程位置，并设置Deploy阶段会使用的CloudFormation脚本模版来更新ECS服务，
2. Build阶段使用AWS CodeBuild来打包软件工程为jar包并上传至S3
3. Build阶段使用AWS CodeBuild来下载最新的jar包并打包成Docker镜像并上传到ECR，
4. Deploy阶段使用Source阶段引入的CloudFormation脚本，找到对应的宿主机集群，负载均衡器，以及上传到ECR的Docker镜像等对象，更新服务。
5. AWS CodePipeline创建后的展示图是这样的，串起了整个CICD流程
  ![图片24](http://cdn.quickstart.org.cn/assets/CICD/pipe-1.png)
在AWS CodePipeline界面点击创建管道Pipeline，可以看到画面左侧一个基本流程，从源，到build阶段生成jar，到build阶段生成docker到部署Deploy。实际应用中用户可以随实际需要，或随着CICD流程的由简入繁在创建后编辑加入新的阶段或操作。
   ![图片25](http://cdn.quickstart.org.cn/assets/CICD/pipe-2.png)
 
点击下一步。
  ![图片26](http://cdn.quickstart.org.cn/assets/CICD/pipe-3.png)
#### Source阶段配置
源提供商下拉菜单选择CodeCommit，选择存储库名称和分支名称，来允许AWS CodePipeline从CodeCommit上获取软件工程源内容， 点击下一步
 
此次实例使用的Demo软件工程可以从以下链接Fork：
https://github.com/.....
#### Build阶段配置
AWS CodePipeline在Build阶段支持包括AWS CodeBuild，Jenkins在内的引擎，此方案选用AWS 托管的CodeBuild服务
   ![图片27](http://cdn.quickstart.org.cn/assets/CICD/pipe-4.png)
选择新建构建项目
   ![图片28](http://cdn.quickstart.org.cn/assets/CICD/pipe-5.png)
选择AWS CodeBuild托管的镜像，支持Ubuntu系统，运行时支持包括Java，Python，Go语言，Node.js，Docker在内的众多选择，此次方案使用Java。
   ![图片29](http://cdn.quickstart.org.cn/assets/CICD/pipe-6.png)
构建规范这里选择使用buildspec.yml，里面预定了AWS CodeBuild在Build生命周期中要执行的动作，如 
  ![图片30](http://cdn.quickstart.org.cn/assets/CICD/pipe-7.png)
Buildspec.yml放在GitHub软件工程源代码目录中

选择Role
   ![图片31](http://cdn.quickstart.org.cn/assets/CICD/pipe-8.png)
新建一个Role，这个Role允许AWS CodeBuild来调用相关的AWS服务，此方案中需要调用包括S3，ECR，CloudWatch在内的服务。
* 默认创建的Role不具备对ECR的权限，需要在保存构建项目后，到IAM找到新创建的Role，编辑添加对ECR的权限否则后面Pipeline执行到Build时会报错。
保存构建项目。 
    ![图片32](http://cdn.quickstart.org.cn/assets/CICD/pipe-9.png)
   ![图片33](http://cdn.quickstart.org.cn/assets/CICD/pipe-10.png)
打开codebuild编辑已有项目
选择构建放置的位置  类型选择S3 填入相应的存储桶名称
![图片](http://cdn.quickstart.org.cn/assets/CICD/adds3.png)
点击下一步。

#### Deploy
AWS CodePipeline部署阶段支持包括AWS CloudFormation，AWS CodeDeploy，AWS Elastic Beanstalk在内的服务提供商，此方案选用AWS CloudFormation来部署ECS容器服务。
    ![图片34](http://cdn.quickstart.org.cn/assets/CICD/pipe-11.png)
这里暂时选择无部署，等Pipeline创建好后，编辑引入Deploy的CloudFormation模版源，再进行配置。
点击下一步。
##### 角色
配置AWS CodePipeline对AWS服务的调用权限，包括S3，AWS CodeBuild，AWS CloudFormation，IAM等。点击创建角色到IAM界面选择相对应的策略创建。
    ![图片34](http://cdn.quickstart.org.cn/assets/CICD/pipe-12.png)
创建好后画面回到Pipeline，IAM创建好的Role已经显示在里面。
    ![图片35](http://cdn.quickstart.org.cn/assets/CICD/pipe-13.png)
点击下一步。
#### 审核后创建管道
管道创建好后会自动运行，现有的从GitHub软件工程源代码抓取工程，打包Docker镜像并推送到ECR上。
#### 添加Deploy阶段CloudFormation需要的模版源
点击编辑，点击Source阶段右上角的画笔图标
    ![图片36](http://cdn.quickstart.org.cn/assets/CICD/pipe-14.png)
可以看到AWS CodePipeline的编辑界面在南北纵向和东西横向都可以添加
     ![图片37](http://cdn.quickstart.org.cn/assets/CICD/pipe-15.png)
在GitHub这个Source右侧，点击添加操作，选择源，操作名称Template，选择S3，输入创建好的S3桶的地址
     ![图片38](http://cdn.quickstart.org.cn/assets/CICD/pipe-16.png)
画面往下拉，注意在输出项目这里，输入Template。
Pipeline中各阶段的传递需要制定南北向的输入输出，即Source阶段S3源的输出Template，在Deploy阶段用输入Template来衔接。
 ![图片39](http://cdn.quickstart.org.cn/assets/CICD/pipe-17.png)
点击更新。
#### Build的构建dockers
点击Build阶段下面的添加阶段，画面右侧选择构建
选择新建构建项目
  ![图片40](http://cdn.quickstart.org.cn/assets/CICD/pipe-18.png)
选择AWS CodeBuild托管的镜像，支持Ubuntu系统，运行时支持包括Java，Python，Go语言，Node.js，Docker在内的众多选择，此次方案使用Docker。
   ![图片41](http://cdn.quickstart.org.cn/assets/CICD/pipe-19.png)
构建规范这里选择使用构建命令，里面预定了AWS CodeBuild在Build生命周期中要执行的动作，如
   ![图片42](http://cdn.quickstart.org.cn/assets/CICD/pipe-20.png)
复制粘贴命令到构建命令中并修改相应参数
```
version: 0.2
phases:
  pre_build:
    commands:
      - $(aws ecr get-login --no-include-email)
      - TAG="$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | head -c 8)"
      - aws s3 cp s3地址  ./
  build:
    commands:
      - docker build -t 替换创建好的ECR镜像仓库的URI:${TAG} .
  post_build:
    commands:
      - docker push "替换创建好的ECR镜像仓库的URI:${TAG}"
      - printf '{"tag":"%s"}' $TAG > build.json
artifacts:
  files: build.json
```
选择Role
   ![图片43](http://cdn.quickstart.org.cn/assets/CICD/pipe-21.png)
新建一个Role，这个Role允许AWS CodeBuild来调用相关的AWS服务，此方案中需要调用包括S3，ECR，CloudWatch在内的服务。
*默认创建的Role不具备对ECR的权限，需要在保存构建项目后，到IAM找到新创建的Role，编辑添加对ECR的权限否则后面Pipeline执行到Build时会报错。
保存构建项目。 
    ![图片44](http://cdn.quickstart.org.cn/assets/CICD/pipe-22.png)
    ![图片45](http://cdn.quickstart.org.cn/assets/CICD/pipe-23.png)
    ![图片46](http://cdn.quickstart.org.cn/assets/CICD/pipe-24.png)
点击下一步。


#### Cloudformation
点击Build阶段下面的添加阶段，画面右侧选择部署，选择AWS CloudFormation
    ![图片47](http://cdn.quickstart.org.cn/assets/CICD/pipe-25.png)
操作模式选择创建或更新堆栈，输入创建的堆栈名称，模版这里输入Template::templates/service.yaml，也就是对应的输入是S3源桶中templates.zip里的service.yaml文件。功能选择CAPABILITY_NAMED_IAM。
  ![图片47](http://cdn.quickstart.org.cn/assets/CICD/pipe-26.png)
同样需要创建一个Role，允许AWS CloudFormation调用包括IAM，ECS，ECR在内的AWS服务。
  ![图片48](http://cdn.quickstart.org.cn/assets/CICD/pipe-27.png)
在IAM界面创建好后选择Role。
高级这里点开
  ![图片49](http://cdn.quickstart.org.cn/assets/CICD/pipe-28.png)
在参数覆盖这里输入CloudFormation需要传入的参数，其中的固定参数也可以在S3的service.yaml中直接定义。
```
{

  "Tag" : { "Fn::GetParam" : [ "MyAppBuild", "build.json", "tag" ] },

  "DesiredCount": "2",

  "Cluster": "XXXXXXX",

  "TargetGroup": "XXXXX",

  "Repository": "XXXXX"

}
```
- Tag是Build阶段传出的Docker镜像Tag使用的值，传入CloudFormation中用于建立Task Definition的Container时从ECR拉取对应版本的Docker镜像。
- DesiredCount，即想要在ECS的Service中建立的Task的数量。
- Cluster，即建立好的宿主机集群的名称。
- TargetGroup，即建立好的宿主机集群的应用负载均衡器的目标组的ARN。
- Repository，即建立好的ECR的镜像仓库名称。
 
输入项目这里输入Build阶段和S3模版源的输出。
     ![图片50](http://cdn.quickstart.org.cn/assets/CICD/pipe-29.png)
点击添加操作。
保存管道更改。

