+++
title = "利用API Gateway访问DynamoDB"
date = 2019-09-09T10:11:17+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

本文介绍如何利用API Gateway作为proxy直接对dynamoDB进行访问，将示例get和post两种方法。   
通过这种方式，无需构建serverside，无需提供credentials，可直接利用标准的https API对ddb内的数据进行读取或者写入操作。


## 步骤

### 1. 创建dynamoDB table  

指定表名称，主键即可。

### 2. 构建API Get请求

#### (1) 选择Scan模式：返回DDB中所有结果 

新建资源，增加get方法。

![](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/database/api-gateway-proxy-for-ddb/get-method.png)

进入Integration Request, 选择DynamoDB服务，action为Scan。

![](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/database/api-gateway-proxy-for-ddb/get-intergration-request.png)

定义mapping template。

![](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/database/api-gateway-proxy-for-ddb/get-mapping-table.png)

#### (2) 选择Query: 查询某项值

新建资源/{year}, 增加get方法。Intergration request的其他选项不变，action改为Query。mapping table更改为如下：

```
{
    "TableName": "xxxx",
    "KeyConditionExpression": "account_type = :v1",   
    "ExpressionAttributeValues": {
        ":v1": {
            "S": "$input.params('account_type')"
        }
    }
}
```

#### (3) 测试     
选择test测试API。对于query选项，path {year}输入需要查询的value；对于scan无需带参。测试完成后无问题部署即可。    


### 3. 构建API Post请求

配置如下
![](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/database/api-gateway-proxy-for-ddb/post-request.png)

mapping table配置如下：

```
{ 
    "TableName": "xxx",
    "Item": {
	"account_type": {
            "S": "$input.path('$.account_type')"
            },
        "balance": {
            "S": "$input.path('$.balance')"
            }
        }
}
```

test时，request body输入数据即可。如

```
{ 
	"account_type" : "savings",
	"balance" : 1950.21
}
```

### 4. 完成部署
选择deploy method完成API部署


## 参考连接    

https://docs.aws.amazon.com/zh_cn/apigateway/latest/developerguide/api-gateway-create-api-step-by-step.html    
https://aws.amazon.com/cn/blogs/compute/using-amazon-api-gateway-as-a-proxy-for-dynamodb/   


   

  
   

   





