+++
title = "基于jenkins的java应用程序CICD解决方案"
date = 2019-09-09T10:26:39+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

## 概述
本文概述了用jenkins完成java项目的CICD流程的所需步骤，其中涉及到codeBuild和codeDeploy的相关配置在 [cicd jar codepipe](https://github.com/yuan00yuan/quickstart-guide/blob/master/cicd%20jar%20codepipe.md)中详细说明
## 步骤
1. 创建jenkins服务器
2. 创建jenkins项目
3. 配置source
4. 配置codeBuild
5. 配置codeDeploy

### 创建jenkins
1. 创建Amazon EC2实例，选择实例类型和添加存储。
   ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-1.png)
2. 在“高级详细信息”里面输入启动脚本
   ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-2.png)

```
#!/bin/bash
yum -y update
yum install -y ruby
yum install -y aws-cli
yum install –y git
cd /home/ec2-user
wget https://bucket-name.s3.amazonaws.com/latest/install
chmod +x ./install
./install auto

```
3. bucket name对应列表为
    ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-3.png)
4.  EC2启动成功后，使用SSH到该EC2，使用如下命令检验Agent是否工作正常。

```
sudo service codedeploy-agent status
Result: The AWS CodeDeploy agent is running as PID 3523

```
- ### 使用Github托管源代码，并配置webhook自动触发
1. 首先进入自己的Github地址，点击https://github.com/settings/tokens，生成GitHub token，这个token用于jenkins访问GitHub。

    ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-4.png)
2. 为需要做CI/CD的GitHub创建hook，实现代码更新自动通知Jenkins，Payload URL设置Jenkins Server的地址，默认Jenkins监听8080端口。记录下生成的token字符串，比如： bf6adc27311a39ad0b5c9a63xxxxxxxxxxxxxx
3. 创建一个新的repository
   ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-5.png)
4. 创建本次环境所需要的Git仓库，比如名为AWS-BJS-CodeDeploy-CICD-Jenkins。点击“Settings”配置webhooks。
   ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-6.png)
5. 点击“Add Webhooks”
6. 在Payload URL，输入http://EC2公网IP地址/github-wekhook/，如下图所示：
    ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-7.png)
### 部署Jenkins，并安装CodeDeploy插件
1. 安装如何脚本安装Jenkins，默认Java的环境是1.7的，可以先升级到Java 1.8版本。

```
sudo -s
java –version
yum install java-1.8.0
yum remove java-1.7.0-openjdk
wget -O /etc/yum.repos.d/jenkins.repo http://jenkins-ci.org/redhat/jenkins.repo
rpm --import http://pkg.jenkins-ci.org/redhat/jenkins-ci.org.key
yum install jenkins
chkconfig jenkins on
service jenkins start
//查看Jenkins默认密码
cat /var/lib/jenkins/secrets/initialAdminPassword

```
2. 在浏览器输入输入EC2的公网IP地址（最好绑定一个弹性EIP），比如54.223.215.xx:8080，然后出现如下界面，输入上面得到的默认密码。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-8.png)
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-9.png)
3. 创建用户名和密码，就到如下界面，这个时候Jenkins就可以进行配置了。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-10.png)
4. 进入系统配置
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-11.png)
5. 输入Jenkins URL，点击“Add”添加Jenkins
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-12.png)
6. 输入Github获取的Access Token，点击“添加”。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-13.png)
7. 点击“Test Connection”，没有报错说明配置成功。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-14.png)
8. 添加管理插件
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-15.png)
9. 添加AWS CodeDeploy的插件，点击“Install without restart”
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-16.png)
10. 新建一Jenkins个项目，点击“Create a new project”
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-17.png)
11. 配置Github项目的地址，源代码管理选择Git方式。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-18.png)
12. 触发构建，选择Github hook trigger for GITScm polling

     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-19.png)
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-20.png)
13. 选择“添加构建步骤”

    选择“AWS cloud build”插件
    
    ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-21.png)
14. 其余保持空白

    点击“添加构建步骤”
    
    选择执行 shell
    
    
```
aws s3 cp s3://yuan0928/target/test_springboot.jar ./   //您在codeBuild中写的文件输出的位置
mkdir target
mv test_springboot.jar target/

```

   这三行脚本的意思是把上一步骤生成的jar包添加到环境中，方便后续的部署工作。
   
   ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-22.png)
    
15. 选择“Post-build Actions”，输入CodeDeploy相关信息，区域选择您所在的code deploy的region
认证方式可以输入AWS Access Key和Security Key，如果是生产环境建议使用临时的credentials。
     ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-23.png)
      ![图片1](http://cdn.quickstart.org.cn/assets/cicd-jar-jenkins/jar-jenkins-24.png)
16. 点击“应用” “保存”

