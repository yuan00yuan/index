+++
title = "利用Amazon Connect + Lex 快速构建自己的呼叫中心 & 智能语音机器人"
date = 2019-09-05T14:35:39+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++


## 实验概览

此实验搭建了一个智能电话订车系统，通过语音交互可以获取用户需要订车的城市，取车以及还车时间，对车型的要求，以及驾驶人的年龄，并且将这些信息通过lambda上传到dynamoDB当中. 

## 架构图

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/connect_architecture.png)

## demo效果

可直接拨打+1 443-692-8678测试，触发词"reserve"  "reserver a car","book a car"等。


## 实现步骤

### 1. 搭建呼叫中心，实现可以人工call in & call out的基本功能   

(1) 创建实例，设置基本选项   
  
	打开Amazon Connect，添加实例，设置访问URL，管理用用户密码, 选择功能(call in/ out/both)，通话存储的位置等。   

(2) 构建呼叫中心   

	点击进入实例（需要管理员密码）。在dashboard当中开始设置。最基本的选项是第一个Claim a phone number，可以选择你的call center number的国家，connect会自动帮你生成一个号码(暂不支持中国+86)。设置完毕后，这个电话就可以使用了，可以进行正常的拨出、拨入，人工对话。   

	除此之外，你可以定义你的call center的运营时间（比如周一9:00AM-18:00PM, 周六10:00AM-14:00PM,周末不上班）等。其他信息可以暂时不进行设置。   


### 2. 使用Lex创建智能语音机器人

(1) 创建并定义Bot   
    
    打开Lex，创建bots，Lex提供多个模板可供demo选择。这里我们选择基于模板BookTrip。COPPA设置为no。

    此时testbots上有两个intents（意图）。代表着用户可以利用此bots做两种触发，一种是订车，一种是订住宿。两者只是定义时的逻辑和数据字段不同，因此此例当中，我们只用到car reservation。

    在Interts的设置上，有以下参数：    

    * Sample utterances：唤醒词，可自行定义或添加。        

    * SLOTS字段：希望Lex帮你获取的信息。默认情况下Lex会按照顺序询问，也可以自己定义priority。       

    * Fulfillment：定义拿到这些参数之后，下一步希望做什么。可以选择直接把这些信息返回给客户端，或者是利用lambda做进一步的存储、处理、分析等。

    * Response：执行完毕后，可以利用这些response做进一步的唤醒。注意，如果这里的fulfillment是lambda，且lambda有return message，将忽略这里的response的设置。 此lab中，我们将用lambda作为car reservation的实现。

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/lex.png)

此外左侧tab还包括：   

* error handling：当获取语音输入失败时，自定义语音，以及重复最多尝试次数。   

* slot types: 可以给用户提示，一些选择的example。

   
（2） 创建lambda函数：存储进dynamoDB

在与Lex同一个region创建lambda函数connect_using_lex。我们将利用这个函数将car reservation环节所得参数存储到dynamoDB当中。请确保lambda的IAM role具有往ddb写入的权限，且提前在ddb当中创建了表connect_lex_car_rent

基于python 2.7的实例代码如下。   

```
import json
import boto3
import uuid

#替换成自己区域的endpoint
dynamodb = boto3.resource('dynamodb', region_name='us-east-1', endpoint_url="http://dynamodb.us-east-1.amazonaws.com")

def lambda_handler(event, context):
    print(event['currentIntent']['name'])
    PickUpCity = event['currentIntent']['slots']['PickUpCity']
    PickUpDate = event['currentIntent']['slots']['PickUpDate']
    ReturnDate = event['currentIntent']['slots']['ReturnDate']
    DriverAge  = event['currentIntent']['slots']['DriverAge']
    CarType    = event['currentIntent']['slots']['CarType']
    

	#dynamoDB table setting 
    table = dynamodb.Table('connect_lex_car_rent')
    item={
		'id':str(uuid.uuid4()),
		'PickUpCity':PickUpCity,
		'PickUpDate':PickUpDate,
		'ReturnDate':ReturnDate,
		'DriverAge':DriverAge,
		'CarType':CarType
        
    }

    resp= table.put_item(Item=item)
    
    #定义自己的response
    msg = "Your reservation has been completed, the pickup city is " + PickUpCity
    
    return {
      "sessionAttributes": {},
      "dialogAction": {
        "type": "Close",
        "fulfillmentState": "Fulfilled",
        "message": {
          "contentType": "PlainText",
          "content": msg
        }
        }
    }

   
```

（3） 在lex当中指定lambda函数，完成构建   

BookCar Intent的fufillment为此lambda函数，BookHotel不变。保存此intent的编写并build。在build成功后，即可在右侧界面进行文字或者语音的交互测试。且可以看到下方的return当中，lex可以实时的获取这些参数。当测试没有问题后，Publish the bots。   
   
![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/testbots.png)


### 3. 将Connect与Lex进行集成

回到Amazon Connect，创建自己的contact flow （create contact flow）。实际上是一个逻辑图，所有的逻辑构建都可以拖拽连线完成。左侧的tab提供了丰富的模块选择，他们分别可以提供不同的功能，比如播放，获取输入，存储电话录音等等。 （注：与lex的交互暂不支持语音存储）     

示例如图。 在该示例当中，核心元素块包括：play prompt播放语音或者背景音乐，get customer input获取用户语音或者按键输入，disconnect挂断。

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/contact-flow.png)

* **（1）**： 电话接通后，会自动播放欢迎语音（play prompt），询问客户需要什么帮助，等待客户回答（get customer input设置为lex）。

* **（2）**： 如果客户的描述符合Sample utterances（如reserve，reserve a car等），则成功触发Lex的smart robot。

* **（3）**： 如无问题(default分支)，则播放感谢语音（play prompt），挂断（disconnect）。

* **（4）**： 如lex识别语音失败或者客户迟迟没有响应（可设置timeout），则触发另外一个分支error。重新获取用户输入。

* **（5）**： 等待用户按键响应。按1位重新预定，按0直接退出。 按键是通过DTMF设置的   

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/get-customer-input-dtmf.png)

完成自己的逻辑设计后，save & publish.   

### 4. 绑定电话号码

自定义完contact flow之后，需要将此flow与你的电话号码进行绑定。在dashboard- view phone number当中可以完成此操作。

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/bindphone.png)


## 监控 & 获取指标

Amazon connect还提供了完善的指标记录。比如可以查询来电数量，来电录音等。在dashboard当中均可以找到。如图，可以查询某一天的来电状况。

![](https://s3.ap-northeast-2.amazonaws.com/salander/quickstarter/Amazon+Connect/metics.png)


## 参考链接

https://aws.amazon.com/cn/blogs/contact-center/amazon-connect-with-amazon-lex-press-or-say-input/   

https://aws.amazon.com/cn/blogs/aws/new-amazon-connect-and-amazon-lex-integration/   


