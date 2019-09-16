+++
title = "Deep Learning AMI下Tensorflow环境的安装与切换"
date = 2019-09-05T14:35:39+08:00
weight = 5
chapter = false
pre = "<b>X. </b>"
+++


在AWS Deep Learning AMI的环境下，如何利用conda管理不同环境，配置所需要的依赖，以及创建新的隔离环境。  
文档来源：尹主任

## 教程

### 0.启动EC2实例，选择Deep Learning AMI


### 1.切换环境
ML AMI自带多套机器学习的环境，如MXNet(+Keras2) with Python3, TensorFlow(+Keras2) with Python3等等。可以利用source activate的命令方便的做切换。如：
```

#切换到MXNet with python3.6

source activate mxnet_p36  

```

### 2.安装新的conda环境

```
 # 通过conda安装环境，并配置对应python版本 
 $ conda create -n tf_gpu14_p35 python=3.5.2 

 # 进入环境
 $ source activate tf_gpu14_p35

 # 进⼊入环境后，可以查看已经安装的包
 $ pip list installed

 #安装tensorflow-gpu
 $ pip install tensorflow-gpu==1.4

 # optional 安装其他的画图相关⼯工具或任意工具
 $ pip install tensorflow==1.12 jupyter matplotlib pandas seaborn numpy
```


### 3.安装notebook&把conda环境加⼊入notebook中

```
# Install jupyter notebook
$ conda install jupyter
$ conda install ipykernel

# Add your conda env to jupyter notebook kernels (此处特别注意，加载的时候要在对应的虚拟环境加载，否则会出现找不不到依赖包的问题)
$ python -m ipykernel install --user --name tf_gpu14_p35 --display-name "my kernel (tensorflow_gpu14_p35)”

# 在base目录Launch Jupyter Notebook 
$ jupyter notebook
```

### 5.通过conda可以设置出多个隔离的交互和训练环境

在对应的训练环境中可以创建对应的ipynb。如果从第三⽅下载的ipynb，必须notebook instance要有对应的环境才可以;比如如下，引入了了第三⽅方的xxx.ipynb，但是其存在于特定的环境conda_mxnet_p36中，但是本机不存在就会报错;

![](https://lab798.s3.cn-north-1.amazonaws.com.cn/legacy/ai/tensorflow/error.png)

### 6.如何查看已经安装的conda env & 如何从notebook中删除conda env 查看列列表

```
jupyter kernelspec list

#   删除特定环境
jupyter kernelspec uninstall unwanted-kernel

```

### 7.指定jupyter加载的ipynb的⽬目录

修改配置⽂文件:
```
$vim jupyter /home/ubuntu/.jupyter/jupyter_notebook_config.py 
```
修改加载⽬目录
```
$c.NotebookApp.notebook_dir = u'/home/ubuntu/TensorFlow-Examples'
```

## 参考链接

###Tensorflow安装

基于ubuntu16.04安装: https://medium.com/@dichen_5479/setup-tensorflow-gpu-with-aws-ec2-on-ubuntu-16-04-in-10-minutes- 7ee64e47a66a   

基于ubuntu16.04(DeepLearning AMI) https://medium.com/@dichen_5479/launch-an-aws-ec2-instance-with-gpu-for-deep-learning-in-10minutes- 685e5c025ec6


### Jupyter Notebook
如何从外部访问notebook https://jupyter-notebook.readthedocs.io/en/stable/public_server.html#notebook-server-security


### Sagemaker for tensorflow
https://github.com/aws/sagemaker-python-sdk/blob/master/src/sagemaker/tensorflow/README.rst
                



