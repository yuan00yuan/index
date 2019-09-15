+++
title = "从Aurora迁移到Redshift并利用Quicksight做数据可视化分析"
date = 2019-09-09T10:27:36+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

# 实验介绍

利用terraform模板启动DMS任务

## 架构图

![](dms.png)

## 实验前提

1. 确保您的本机已经安装 Terraform  （https://www.terraform.io/）
2. 提前创建好EC2的秘钥对
3. 事先用AWSCLI，aws configure配置好了凭证以有权启动这些资源

## 步骤


步骤1： 替换terraform变量
替换variable.tf里面的参数, 包括：VPC，DB subnet,EC2 subnet，EC2 key pair, region & 相应地区的AMI ID。 

补充说明: 默认在us-west-2，因启动脚本用了yum install & wget等命令, 如果要换区域，请尽量添加Amazon Linux AMI确保这些命令可以正常运行   


步骤2： 启动terrform模板

```
terraform init
terraform apply
```

成功后会创建好以下相关资源 ,且bastion EC2已通过启动脚本(mysql_init.tpl)加载好了Aurora的实验数据。  	
记下这些output 

```
Outputs Example:
Aurora database name = lab
Aurora endpoint = tf-xxxxxxx.us-west-2.rds.amazonaws.com
Aurora master password = xxxx
Aurora master username = root
DMS Aurora Source ID = pdaxxx
DMS RedShift Target ID = ehjxxxx
DMS instanace name = vipxxxx
EC2 Bastion Endpoint = 
RedShift Endpoint = xxxxxx.us-west-2.redshift.amazonaws.com
RedShift database name = lab
RedShift master password  = 3GMxxxx
Redshift master username = root
Quicksight security group = sg-xxxx
```

步骤3： 创建DMS任务   

新建一个DMS任务，指定好source & target & replication instance & mapping table （schema name: lab）  
当status为Load complete时，DMS任务执行完毕，检查table loaded这一栏应为1，full load rows为41188.
   
   
   
步骤4： Quicksight数据可视化   
   
右上角切换Quicksight区域到指定区域如Oregon, 测试从Aurora或者redshift加载数据。  
因为Aurora和redshift均在私有子网当中，因此首先需要给Quicksight加一个VPC connection。    
   

1. 添加VPC connection   

	右上角选择用户名---下拉tab:manage quicksight --- 左侧tab:manage VPC connections --add VPC connection   
	选择VPC ID和output当中的Quicksight security group，以创建VPC连接   

	注:   
	（1）如没有mange VPC connections的选项，需要先开启enterprise版本  
	（2）这里Quicksight的端口为全开，因此aurora和redshift均可以使用这一安全组做连接。实际生产请开启自己需要的端口。 


2. 数据可视化   

	选择Aurora或redshift实例为数据源，选择刚才创建的vpc connection. database name:lab。输入用户名密码（在output当中）  

	validate确保connectivity没问题后，即可创建。   

	左侧可以看到age, job, marital这几栏。任意拖拽数据栏生成自己的dashboard   
   

   
步骤5:  销毁资源

```
terraform destroy
```

Note: 由于一些依赖关系，destroy命令可能会报错，这时手动删除报错资源再运行此命令销毁其他资源即可。





