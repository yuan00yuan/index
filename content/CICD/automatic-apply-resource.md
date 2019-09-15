+++
title = "自动化脚本：监测 & 自动启动可用资源"
date = 2019-09-09T10:26:39+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

## 场景概览

某些时候某可用区的资源暂时不足，启动时会启动失败。如果客户有一些资源要求一定要在特定可用区部署特定机型，需要经常去刷新资源。   
此脚本通过持续申请资源并且一旦有空闲资源，以预留容量的方式锁定资源来解决上述问题。   
示例代码演示了在宁夏cn-northwest-1a可用区申请r5.2xlarge资源。如果失败会持续执行，如果成功自动按预留容量启动。  
点击此链接了解预留容量：https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html

## 前提条件
 
#### 1. 安装以及配置AWSCLI   
安装：https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-install.html    
配置：https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-configure.html      
```
aws configure
```
#### 2. 安装AWS官方python SDK BOTO3
```
sudo easy_install pip 
pip install boto3
```
#### 3. 执行前需要修改参数：实例类型（如c5.2xlarge），平台/操作系统(如Linux, windows，可添加多个),  可用区。
具体参数说明：https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.create_capacity_reservation

## 具体实现

注：以下基于python2.7

```

from botocore.exceptions import ClientError
import boto3
import json

client = boto3.client('ec2',region_name='cn-northwest-1')


while 1:
    print('enter the loop')
    try:
        #revise your own instance type & operating system & availability zone here!!
        response = client.create_capacity_reservation(InstanceType='r5.2xlarge',InstancePlatform='Linux/UNIX',AvailabilityZone='cn-northwest-1a',InstanceCount=1,EndDateType='unlimited')
        print('[Success]check the detailed info below or in the console')
        print(response)
        break
    except ClientError as e:
        #print(e)
        #Launch failed. Reason: Exceed limit 
        if 'InstanceLimitExceeded' in str(e):
            print('[Failed] Reached instance Limit. Please submit request for higher limits')
            break
        #Launch failed. Reason: No resource available.
        elif 'Insufficient capacity' in str(e):
            print('[Failed] Insufficient capacity right now, making an another request...')
            continue
        #Other errors. 
        else:
            print(e)
            break
```

运行此上脚本分三种情况：
#### 1. 执行成功，会成功预留此容量，程序运行结束。
****注意：在不使用的情况下未取消预留容量也会持续扣费。如不再使用，可随时通过控制台或SDK取消。***
```

#How to cancel reservation capacity? Via console or below API
response = client.cancel_capacity_reservation(
    CapacityReservationId='xxxxxxxxxxx'
)

```
#### 2. 启动失败：当前无可用资源 Insufficient capacity
此可用区当前无可用资源。此时程序不会退出，会一直维持while循环，直到成功或者手动停止此程序。
 
#### 3. 启动失败：已达到实例上限 InstanceLimitExceeded
每个账户下会有instance limit, 如果已经达到此上限，无法启动成功，此时需要提case给后台申请提高此限制。   
此时程序直接退出。


## 参考链接

https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html#EC2.Client.create_capacity_reservation   

https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html



