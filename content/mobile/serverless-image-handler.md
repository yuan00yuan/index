+++
title = "S3图片处理"
date = 2019-09-09T10:27:31+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++

如果您使用的是AWS Global Region, 您可以停止阅读本文，转向使用[Serverless Image Hanlder](https://docs.aws.amazon.com/solutions/latest/serverless-image-handler/deployment.html)提供的CloudFormation时间自动部署。

**本文提到的部署方式，仅适用于AWS China**; 关于能够实现那些图片处理功能，请参考[文档](https://docs.aws.amazon.com/solutions/latest/serverless-image-handler/welcome.html)

## 修改源代码，配置参数
[点击此处下载源代码](https://s3.cn-northwest-1.amazonaws.com.cn/aws-quickstart/quickstart-serverless-image-handler/ServerlessImageHandler.zip)

1. 修改`image_handler/thumbor.conf`
  * `TC_AWS_REGION` 定义为 `cn-northwest-1` 或者 `cn-north-1`
  * `TC_AWS_ENDPOINT` 定义为`https://s3.cn-northwest-1.amazonaws.com.cn` 和`https://s3.cn-north-1.amazonaws.com.cn`
  * `TC_AWS_LOADER_BUCKET` 定义为源S3的bucket名称

2. 修改`tc_aws/__init__.py`
  * `TC_AWS_REGION` 定义为 `cn-northwest-1`或者`cn-north-1`
  * `TC_AWS_LOADER_BUCKET` 定义为源S3的bucket名称
  * `TC_AWS_ENDPOINT` 定义为`https://s3.cn-northwest-1.amazonaws.com.cn` 或者 `https://s3.cn-north-1.amazonaws.com.cn`

> 本文的源代码与global region的源代码还存在以下几处修改，在本文的源代码中已经更改，用户无需再次修改

1. 修改`image_handler/thumbor.conf`
  * `ALLOW_UNSAFE_URL` 定义为 True（提供的代码已经更改）

2. 修改`thumbor/thumbor.conf`
  * `ALLOW_UNSAFE_URL` 改成 True（提供的代码已经更改）

3. 修改`image_handler/lambda_function.py`

```
if str(os.environ.get('ENABLE_CORS')).upper() == "YES":
    api_response['headers']['Access-Control-Allow-Origin'] = os.environ.get('CORS_ORIGIN')
```
改成

```
api_response['headers']['Access-Control-Allow-Origin'] = os.environ.get('*')
```


## 上传代码至S3

* 打包源文件（工程文件必须位于压缩包一级目录）
* 上传到与 Lambda, API Gateway 在同一个region的S3

## 配置 IAM Role
创建一个供Lambda使用的 IAM Role, 并且包括如下两条policy

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws-cn:logs:*:*:*"
        }
    ]
}
```
以上policy允许Lambda将日志写到CloudWatch Logs

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws-cn:s3:::<bucket-name>/*",
                "arn:aws-cn:s3:::<bucket-name>"
            ],
            "Effect": "Allow"
        }
    ]
}
```
将`<bucket-name>`替换成源S3 bucket 名称。

## 创建Lambda

1. 创建Lambda函数，选择 Python 2.7 执行环境
2. 选择刚才创建的IAM Role
3. 在 Function Code 部分，选择**upload a file from Amazon S3**
4. 在 **Amazon S3 link URL**，输入压缩包的S3的URL链接
5. 在 **Handler**，输入`image_handler/lambda_function.lambda_handler`
6. 在 **Basic settings**, 将 **Memory** 调整到 1536MB 或以上
7. 在 **Basic settings**，将 **Timeout** 调整到 10 sec
8. 点击保存

## 创建API Gateway

1. 在API Gateway页面，点击 **Create API**, 输入 **API name**
2. 在**Resouces**页面，点击**Actions**, 选择 **Create Resource**
3. 勾选**Configure as proxy resource**
4. 勾选**Enable API Gateway CORS**，并选择 **Create Resource**
5. 选择 **Lambda Function Proxy**
6. 选择正确的Lambda region
7. 输入Lambda Function的名称，并选择**Save**
8. 在弹出的确认窗口选择**OK**
9. 在左侧导航栏，选择 **Settings**
10. 选择**Add Binary Media Type**，并输入`*/*`
11. 选择**Save Changes**
12. 在左侧导航栏，选择**Resources**，点击**Actions**并选择**Deploy API**
13. 在**Deployment stage** 选择**New Stage**，并输入名称，选择**Deploy**
14. 拷贝**Invoke URL**
15. [可选]在左侧导航栏选择**Stages**, 并点击刚才创建的**Stage**
16. [可选]在**Cache Settings**选择**Enable API cache**，选择**Cache Capacity**
17. [可选]在**Cache time-to-live (TTL)**输入TTL,并保存

开启 API Gateway Cache 会减少 Lambda的执行次数，从而提供相同requet请求时的响应时间。

## 测试自动切图功能

1. 浏览器输入 `https://<api-gateway-invoke-url>/fit-in/300x400/<s3-object-key>`，例如`https://fwgzof7dee.execute-api.cn-northwest-1.amazonaws.com.cn/dev/fit-in/300x400/demo/flower.jpg`获得300x400的切图
2. 浏览器输入 `https://<api-gateway-invoke-url>/filters:blur(7)/<s3-object-key>`，例如`https://fwgzof7dee.execute-api.cn-northwest-1.amazonaws.com.cn/dev/filters:blur(7)/demo/flower.jpg`获得模糊图片

支持的Filter，请参考[Serverless Imager Handler Appendix A: Supported Filters](https://docs.aws.amazon.com/solutions/latest/serverless-image-handler/appendix-a.html)

