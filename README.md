# [AI训练营]paddleclas实现图像分类baseline

## 一、项目背景
暑期参加了百度AI达人创造营活动，大作业完成自己的项目，由于自己能力有限，知识储备量还是不那么足够，只能根据大佬的教程完成基本操作。
本项目选择利用PaddleClas实现图像分类，进行五大食物分类。
> [课程链接：飞桨领航团AI达人创造营](https://aistudio.baidu.com/aistudio/course/introduce/24607?directly=1&shared=1)    

<hr>
<center><font size="3px" color="green">暑    期    充    电    季</font></center>
<hr>
<center><font size="2px" color="red">百度飞桨领航团全新推出“AI达人创造营”</font></center>
<center><font size="2px" color="green"><strong>十位飞桨开发者技术专家（PPDE）</strong>手把手教大家完成项目从idea思考到部署落地的全流程实战</font></center>
<center><font >最终让每位参与者都有一个可以给自己简历加分的项目</font></center>
<center><font >7月26日-8月16日，每晚 19:00-21:00 直播讲解、十位飞桨开发者技术专家（PPDE）手把手教助你成为AI达人</font></center>   
<center><font size="2px" color="blue">报名后，请加入课程 QQ 群861942585，QQ群用于直播提醒、实时答疑、交流互动等</font></center>  

<hr>

### 1.1 项目实现简介
正如标题，采用paddleclas套件实现分类[30分钟玩转PaddleClas（尝鲜版）](https://gitee.com/paddlepaddle/PaddleClas/blob/release/2.2/docs/zh_CN/tutorials/quick_start_new_user.md)    
查看套件，可以知道    
实现分类，仅仅需要我们将数据集提取为如下这种格式的txt文件即可（当然我们并不需要更大的训练集）    
![](https://ai-studio-static-online.cdn.bcebos.com/012e5e2e7a204f77ab2ad9759860978d6b19d55946dd4ee2adea6a4e9736d121)

## 二、数据集介绍

<font size="3px" color="red">本次数据集有五个可供大家选择。分别是：</font>   
1. 猫12分类
2. 垃圾40分类
3. 场景5分类
4. 食物5分类
5. 蝴蝶20分类<br>

我们在这里选择食物五大类进行图像分类，图像五大类分别包括：

|<center> 食物 1<center/> | <center> 食物 2<center/>| <center> 食物 3<center/>  | <center> 食物 4<center/>  | <center> 食物 5<center/>  |
| -------- | -------- | -------- |-------- | -------- |
| beef_carpaccio| baby_back_ribs|beef_tartare| apple_pie| baklava|




```python
# 先导入库
from sklearn.utils import shuffle
import os
import pandas as pd
import numpy as np
from PIL import Image
import paddle
import paddle.nn as nn
import random
```


```python
# 忽略（垃圾）警告信息
# 在python中运行代码经常会遇到的情况是——代码可以正常运行但是会提示警告，有时特别讨厌。
# 那么如何来控制警告输出呢？其实很简单，python通过调用warnings模块中定义的warn()函数来发出警告。我们可以通过警告过滤器进行控制是否发出警告消息。
import warnings
warnings.filterwarnings("ignore")
```

### 2.1 解压数据集，查看数据的结构


```python
# 项目挂载的数据集先解压出来，待解压完毕，刷新后可发现左侧文件夹根目录出现五个zip
!unzip -oq /home/aistudio/data/data103736/五种图像分类数据集.zip
```

左侧可以看到如图所示五个zip    
![](https://ai-studio-static-online.cdn.bcebos.com/f8bc5b21a0ba49b4b78b6e7b18ac0341dfb14cf545b14c83b1f597b6ee8109bb)



```python
# 我们选择食物分类为例进行介绍，因为分类大多数情况下是不存在标签文件的，猫分类已经有了标签，省去了数据处理的操作。
# 解压完毕左侧出现文件夹，即为需要分类的文件
!unzip -oq /home/aistudio/食物5分类.zip
```

左侧可以看到如图所示五个zip    
![](https://ai-studio-static-online.cdn.bcebos.com/0649387a1d1945b48cc6139f366bddc86104776b26414ee792a30b951dc708cb)



```python
# 查看结构，正为一个类别下有一系列对应的图片
!tree foods/
```

    5 directories, 5000 files

**五类食物图片**  
1. beef_carpaccio
2. baby_back_ribs
3. beef_tartare
4. apple_pie
5. baklava    

具体结构如下：
```
foods/
├── apple_pie
│   ├── 1005649.jpg
│   ├── 1011328.jpg
│   ├── 101251.jpg
```

### 2.2 拿到总的训练数据txt


```python
import os
# -*- coding: utf-8 -*-
# 根据官方paddleclas的提示，我们需要把图像变为两个txt文件
# train_list.txt（训练集）
# val_list.txt（验证集）
# 先把路径搞定 比如：foods/beef_carpaccio/855780.jpg ,读取到并写入txt 

# 根据左侧生成的文件夹名字来写根目录
dirpath = "foods"
# 先得到总的txt后续再进行划分，因为要划分出验证集，所以要先打乱，因为原本是有序的
def get_all_txt():
    all_list = []
    i = 0 # 标记总文件数量
    j = 0 # 标记文件类别
    for root,dirs,files in os.walk(dirpath): # 分别代表根目录、文件夹、文件
        for file in files:
            i = i + 1 
            # 文件中每行格式： 图像相对路径      图像的label_id（数字类别）（注意：中间有空格）。              
            imgpath = os.path.join(root,file)
            all_list.append(imgpath+" "+str(j)+"\n")

        j = j + 1

    allstr = ''.join(all_list)
    f = open('all_list.txt','w',encoding='utf-8')
    f.write(allstr)
    return all_list , i
all_list,all_lenth = get_all_txt()
print(all_lenth)
```

    5000


### 2.3 数据打乱


```python
# 把数据打乱
all_list = shuffle(all_list)
allstr = ''.join(all_list)
f = open('all_list.txt','w',encoding='utf-8')
f.write(allstr)
print("打乱成功，并重新写入文本")
```

    打乱成功，并重新写入文本


### 2.4 数据划分


```python
# 按照比例划分数据集 食品的数据有5000张图片，不算大数据，一般9:1即可
train_size = int(all_lenth * 0.9)
train_list = all_list[:train_size]
val_list = all_list[train_size:]

print(len(train_list))
print(len(val_list))
```

    4500
    500



```python
# 运行cell，生成训练集txt 
train_txt = ''.join(train_list)
f_train = open('train_list.txt','w',encoding='utf-8')
f_train.write(train_txt)
f_train.close()
print("train_list.txt 生成成功！")

# 运行cell，生成验证集txt
val_txt = ''.join(val_list)
f_val = open('val_list.txt','w',encoding='utf-8')
f_val.write(val_txt)
f_val.close()
print("val_list.txt 生成成功！")
```

    train_list.txt 生成成功！
    val_list.txt 生成成功！


## 三、安装paddleclas

数据集核实完搞定成功的前提下，可以准备更改原文档的参数进行实现自己的图片分类了！

这里采用paddleclas的2.2版本，好用！


```python
# 先把paddleclas安装上再说
# 安装paddleclas以及相关三方包(好像studio自带的已经够用了，无需安装了)
!git clone https://gitee.com/paddlepaddle/PaddleClas.git -b release/2.2
# 我这里安装相关包时，花了30几分钟还有错误提示，不管他即可
#!pip install --upgrade -r PaddleClas/requirements.txt -i https://mirror.baidu.com/pypi/simple
```

    Cloning into 'PaddleClas'...
    remote: Enumerating objects: 538, done.[K
    remote: Counting objects: 100% (538/538), done.[K
    remote: Compressing objects: 100% (323/323), done.[K
    remote: Total 15290 (delta 347), reused 349 (delta 210), pack-reused 14752[K
    Receiving objects: 100% (15290/15290), 113.56 MiB | 8.71 MiB/s, done.
    Resolving deltas: 100% (10239/10239), done.
    Checking connectivity... done.



```python
#因为后续paddleclas的命令需要在PaddleClas目录下，所以进入PaddleClas根目录，执行此命令
%cd PaddleClas
!ls
```

    /home/aistudio/PaddleClas
    dataset  hubconf.py   MANIFEST.in    README_ch.md  requirements.txt
    deploy	 __init__.py  paddleclas.py  README_en.md  setup.py
    docs	 LICENSE      ppcls	     README.md	   tools



```python
# 将图片移动到paddleclas下面的数据集里面
# 至于为什么现在移动，也是我的一点小技巧，防止之前移动的话，生成的txt的路径是全路径，反而需要去掉路径的一部分
!mv ../foods/ dataset/
```


```python
# 挪动文件到对应目录
!mv ../all_list.txt dataset/foods
!mv ../train_list.txt dataset/foods
!mv ../val_list.txt dataset/foods
```

### 3.1 修改配置文件
#### 3.1.1
主要是以下几点：分类数、图片总量、训练和验证的路径、图像尺寸、数据预处理、训练和预测的num_workers: 0

路径如下：
>PaddleClas/ppcls/configs/quick_start/new_user/ShuffleNetV2_x0_25.yaml

<font size="3px" color="red">（主要的参数如下：）</font>

```
# global configs
Global:
  checkpoints: null
  pretrained_model: null
  output_dir: ./output/
  # 使用GPU训练
  device: gpu
  # 每几个轮次保存一次
  save_interval: 1 
  eval_during_train: True
  # 每几个轮次验证一次
  eval_interval: 1 
  # 训练轮次
  epochs: 20 
  print_batch_step: 1
  use_visualdl: True #开启可视化（目前平台不可用）
  # used for static mode and model export
  # 图像大小
  image_shape: [3, 224, 224] 
  save_inference_dir: ./inference
  # training model under @to_static
  to_static: False

# model architecture
Arch:
  # 采用的网络
  name: ResNet50
  # 类别数 多了个0类 0-5 0无用 
  class_num: 6 
 
# loss function config for traing/eval process
Loss:
  Train:

    - CELoss: 
        weight: 1.0
  Eval:
    - CELoss:
        weight: 1.0


Optimizer:
  name: Momentum
  momentum: 0.9
  lr:
    name: Piecewise
    learning_rate: 0.015
    decay_epochs: [30, 60, 90]
    values: [0.1, 0.01, 0.001, 0.0001]
  regularizer:
    name: 'L2'
    coeff: 0.0005


# data loader for train and eval
DataLoader:
  Train:
    dataset:
      name: ImageNetDataset
      # 根路径
      image_root: ./dataset/
      # 前面自己生产得到的训练集文本路径
      cls_label_path: ./dataset/foods/train_list.txt
      # 数据预处理
      transform_ops:
        - DecodeImage:
            to_rgb: True
            channel_first: False
        - ResizeImage:
            resize_short: 256
        - CropImage:
            size: 224
        - RandFlipImage:
            flip_code: 1
        - NormalizeImage:
            scale: 1.0/255.0
            mean: [0.485, 0.456, 0.406]
            std: [0.229, 0.224, 0.225]
            order: ''

    sampler:
      name: DistributedBatchSampler
      batch_size: 128
      drop_last: False
      shuffle: True
    loader:
      num_workers: 0
      use_shared_memory: True

  Eval:
    dataset: 
      name: ImageNetDataset
      # 根路径
      image_root: ./dataset/
      # 前面自己生产得到的验证集文本路径
      cls_label_path: ./dataset/foods/val_list.txt
      # 数据预处理
      transform_ops:
        - DecodeImage:
            to_rgb: True
            channel_first: False
        - ResizeImage:
            resize_short: 256
        - CropImage:
            size: 224
        - NormalizeImage:
            scale: 1.0/255.0
            mean: [0.485, 0.456, 0.406]
            std: [0.229, 0.224, 0.225]
            order: ''
    sampler:
      name: DistributedBatchSampler
      batch_size: 128
      drop_last: False
      shuffle: True
    loader:
      num_workers: 0
      use_shared_memory: True

Infer:
  infer_imgs: ./dataset/foods/beef_carpaccio/855780.jpg
  batch_size: 10
  transforms:
    - DecodeImage:
        to_rgb: True
        channel_first: False
    - ResizeImage:
        resize_short: 256
    - CropImage:
        size: 224
    - NormalizeImage:
        scale: 1.0/255.0
        mean: [0.485, 0.456, 0.406]
        std: [0.229, 0.224, 0.225]
        order: ''
    - ToCHWImage:
  PostProcess:
    name: Topk
    # 输出的可能性最高的前topk个
    topk: 5
    # 标签文件 需要自己新建文件
    class_id_map_file: ./dataset/label_list.txt

Metric:
  Train:
    - TopkAcc:
        topk: [1, 5]
  Eval:
    - TopkAcc:
        topk: [1, 5]
```
#### 3.1.2 标签文件
这个是在预测时生成对照的依据，在上个文件有提到这个
```
# 标签文件 需要自己新建文件
    class_id_map_file: dataset/label_list.txt
```
按照对应的进行编写：   

![](https://ai-studio-static-online.cdn.bcebos.com/0e40a0afaa824ba9b70778aa7931a3baf2a421bcb81b4b0f83632da4e4ddc0ef)  

如食品分类(要对照之前的txt的类别确认无误) 
```
1 beef_carpaccio
2 baby_back_ribs
3 beef_tartare
4 apple_pie
5 baklava
```
![](https://ai-studio-static-online.cdn.bcebos.com/6b3a4a244ed34517bcf4be5fcc629ab10d081eb0af7c4532a795583d19939a82)




![](https://ai-studio-static-online.cdn.bcebos.com/8ee69dbad8bf4e0f9c743645a4438cc035446d052d4b409a88c1cc25b58dcf83)

### 3.2 开始训练


```python
# 提示，运行过程中可能存在坏图的情况，但是不用担心，训练过程不受影响。
# 仅供参考，我只跑了五轮，准确率很低
!python3 tools/train.py \
    -c ./ppcls/configs/quick_start/new_user/ShuffleNetV2_x0_25.yaml
```

    /home/aistudio/PaddleClas/ppcls/arch/backbone/model_zoo/vision_transformer.py:15: DeprecationWarning: Using or importing the ABCs from 'collections' instead of from 'collections.abc' is deprecated, and in 3.8 it will stop working
      from collections import Callable
    [2021/08/15 15:51:32] root INFO: 
    ===========================================================
    ==        PaddleClas is powered by PaddlePaddle !        ==
    ===========================================================
    ==                                                       ==
    ==   For more info please go to the following website.   ==
    ==                                                       ==
    ==       https://github.com/PaddlePaddle/PaddleClas      ==
    ===========================================================
    
    [2021/08/15 15:51:32] root INFO: Arch : 
    [2021/08/15 15:51:32] root INFO:     class_num : 6
    [2021/08/15 15:51:32] root INFO:     name : ResNet50
    [2021/08/15 15:51:32] root INFO: DataLoader : 
    [2021/08/15 15:51:32] root INFO:     Eval : 
    [2021/08/15 15:51:32] root INFO:         dataset : 
    [2021/08/15 15:51:32] root INFO:             cls_label_path : ./dataset/foods/val_list.txt
    [2021/08/15 15:51:32] root INFO:             image_root : ./dataset/
    [2021/08/15 15:51:32] root INFO:             name : ImageNetDataset
    [2021/08/15 15:51:32] root INFO:             transform_ops : 
    [2021/08/15 15:51:32] root INFO:                 DecodeImage : 
    [2021/08/15 15:51:32] root INFO:                     channel_first : False
    [2021/08/15 15:51:32] root INFO:                     to_rgb : True
    [2021/08/15 15:51:32] root INFO:                 ResizeImage : 
    [2021/08/15 15:51:32] root INFO:                     resize_short : 256
    [2021/08/15 15:51:32] root INFO:                 CropImage : 
    [2021/08/15 15:51:32] root INFO:                     size : 224
    [2021/08/15 15:51:32] root INFO:                 NormalizeImage : 
    [2021/08/15 15:51:32] root INFO:                     mean : [0.485, 0.456, 0.406]
    [2021/08/15 15:51:32] root INFO:                     order : 
    [2021/08/15 15:51:32] root INFO:                     scale : 1.0/255.0
    [2021/08/15 15:51:32] root INFO:                     std : [0.229, 0.224, 0.225]
    [2021/08/15 15:51:32] root INFO:         loader : 
    [2021/08/15 15:51:32] root INFO:             num_workers : 0
    [2021/08/15 15:51:32] root INFO:             use_shared_memory : True
    [2021/08/15 15:51:32] root INFO:         sampler : 
    [2021/08/15 15:51:32] root INFO:             batch_size : 128
    [2021/08/15 15:51:32] root INFO:             drop_last : False
    [2021/08/15 15:51:32] root INFO:             name : DistributedBatchSampler
    [2021/08/15 15:51:32] root INFO:             shuffle : True
    [2021/08/15 15:51:32] root INFO:     Train : 
    [2021/08/15 15:51:32] root INFO:         dataset : 
    [2021/08/15 15:51:32] root INFO:             cls_label_path : ./dataset/foods/train_list.txt
    [2021/08/15 15:51:32] root INFO:             image_root : ./dataset/
    [2021/08/15 15:51:32] root INFO:             name : ImageNetDataset
    [2021/08/15 15:51:32] root INFO:             transform_ops : 
    [2021/08/15 15:51:32] root INFO:                 DecodeImage : 
    [2021/08/15 15:51:32] root INFO:                     channel_first : False
    [2021/08/15 15:51:32] root INFO:                     to_rgb : True
    [2021/08/15 15:51:32] root INFO:                 ResizeImage : 
    [2021/08/15 15:51:32] root INFO:                     resize_short : 256
    [2021/08/15 15:51:32] root INFO:                 CropImage : 
    [2021/08/15 15:51:32] root INFO:                     size : 224
    [2021/08/15 15:51:32] root INFO:                 RandFlipImage : 
    [2021/08/15 15:51:32] root INFO:                     flip_code : 1
    [2021/08/15 15:51:32] root INFO:                 NormalizeImage : 
    [2021/08/15 15:51:32] root INFO:                     mean : [0.485, 0.456, 0.406]
    [2021/08/15 15:51:32] root INFO:                     order : 
    [2021/08/15 15:51:32] root INFO:                     scale : 1.0/255.0
    [2021/08/15 15:51:32] root INFO:                     std : [0.229, 0.224, 0.225]
    [2021/08/15 15:51:32] root INFO:         loader : 
    [2021/08/15 15:51:32] root INFO:             num_workers : 0
    [2021/08/15 15:51:32] root INFO:             use_shared_memory : True
    [2021/08/15 15:51:32] root INFO:         sampler : 
    [2021/08/15 15:51:32] root INFO:             batch_size : 128
    [2021/08/15 15:51:32] root INFO:             drop_last : False
    [2021/08/15 15:51:32] root INFO:             name : DistributedBatchSampler
    [2021/08/15 15:51:32] root INFO:             shuffle : True
    [2021/08/15 15:51:32] root INFO: Global : 
    [2021/08/15 15:51:32] root INFO:     checkpoints : None
    [2021/08/15 15:51:32] root INFO:     device : gpu
    [2021/08/15 15:51:32] root INFO:     epochs : 20
    [2021/08/15 15:51:32] root INFO:     eval_during_train : True
    [2021/08/15 15:51:32] root INFO:     eval_interval : 1
    [2021/08/15 15:51:32] root INFO:     image_shape : [3, 224, 224]
    [2021/08/15 15:51:32] root INFO:     output_dir : ./output/
    [2021/08/15 15:51:32] root INFO:     pretrained_model : None
    [2021/08/15 15:51:32] root INFO:     print_batch_step : 1
    [2021/08/15 15:51:32] root INFO:     save_inference_dir : ./inference
    [2021/08/15 15:51:32] root INFO:     save_interval : 1
    [2021/08/15 15:51:32] root INFO:     to_static : False
    [2021/08/15 15:51:32] root INFO:     use_visualdl : True
    [2021/08/15 15:51:32] root INFO: Infer : 
    [2021/08/15 15:51:32] root INFO:     PostProcess : 
    [2021/08/15 15:51:32] root INFO:         class_id_map_file : ./dataset/label_list.txt
    [2021/08/15 15:51:32] root INFO:         name : Topk
    [2021/08/15 15:51:32] root INFO:         topk : 5
    [2021/08/15 15:51:32] root INFO:     batch_size : 10
    [2021/08/15 15:51:32] root INFO:     infer_imgs : ./dataset/foods/beef_carpaccio/855780.jpg
    [2021/08/15 15:51:32] root INFO:     transforms : 
    [2021/08/15 15:51:32] root INFO:         DecodeImage : 
    [2021/08/15 15:51:32] root INFO:             channel_first : False
    [2021/08/15 15:51:32] root INFO:             to_rgb : True
    [2021/08/15 15:51:32] root INFO:         ResizeImage : 
    [2021/08/15 15:51:32] root INFO:             resize_short : 256
    [2021/08/15 15:51:32] root INFO:         CropImage : 
    [2021/08/15 15:51:32] root INFO:             size : 224
    [2021/08/15 15:51:32] root INFO:         NormalizeImage : 
    [2021/08/15 15:51:32] root INFO:             mean : [0.485, 0.456, 0.406]
    [2021/08/15 15:51:32] root INFO:             order : 
    [2021/08/15 15:51:32] root INFO:             scale : 1.0/255.0
    [2021/08/15 15:51:32] root INFO:             std : [0.229, 0.224, 0.225]
    [2021/08/15 15:51:32] root INFO:         ToCHWImage : None
    [2021/08/15 15:51:32] root INFO: Loss : 
    [2021/08/15 15:51:32] root INFO:     Eval : 
    [2021/08/15 15:51:32] root INFO:         CELoss : 
    [2021/08/15 15:51:32] root INFO:             weight : 1.0
    [2021/08/15 15:51:32] root INFO:     Train : 
    [2021/08/15 15:51:32] root INFO:         CELoss : 
    [2021/08/15 15:51:32] root INFO:             weight : 1.0
    [2021/08/15 15:51:32] root INFO: Metric : 
    [2021/08/15 15:51:32] root INFO:     Eval : 
    [2021/08/15 15:51:32] root INFO:         TopkAcc : 
    [2021/08/15 15:51:32] root INFO:             topk : [1, 5]
    [2021/08/15 15:51:32] root INFO:     Train : 
    [2021/08/15 15:51:32] root INFO:         TopkAcc : 
    [2021/08/15 15:51:32] root INFO:             topk : [1, 5]
    [2021/08/15 15:51:32] root INFO: Optimizer : 
    [2021/08/15 15:51:32] root INFO:     lr : 
    [2021/08/15 15:51:32] root INFO:         decay_epochs : [30, 60, 90]
    [2021/08/15 15:51:32] root INFO:         learning_rate : 0.015
    [2021/08/15 15:51:32] root INFO:         name : Piecewise
    [2021/08/15 15:51:32] root INFO:         values : [0.1, 0.01, 0.001, 0.0001]
    [2021/08/15 15:51:32] root INFO:     momentum : 0.9
    [2021/08/15 15:51:32] root INFO:     name : Momentum
    [2021/08/15 15:51:32] root INFO:     regularizer : 
    [2021/08/15 15:51:32] root INFO:         coeff : 0.0005
    [2021/08/15 15:51:32] root INFO:         name : L2
    W0815 15:51:32.553943   599 device_context.cc:404] Please NOTE: device: 0, GPU Compute Capability: 7.0, Driver API Version: 10.1, Runtime API Version: 10.1
    W0815 15:51:32.558450   599 device_context.cc:422] device: 0, cuDNN Version: 7.6.
    [2021/08/15 15:51:37] root INFO: train with paddle 2.1.2 and device CUDAPlace(0)
    {'CELoss': {'weight': 1.0}}
    [2021/08/15 15:51:38] root INFO: [Train][Epoch 1/20][Iter: 0/36]lr: 0.10000, CELoss: 1.96087, loss: 1.96087, top1: 0.08594, top5: 0.81250, batch_cost: 1.20886s, reader_cost: 1.05243, ips: 105.88489 images/sec, eta: 0:14:30
    [2021/08/15 15:51:39] root INFO: [Train][Epoch 1/20][Iter: 1/36]lr: 0.10000, CELoss: 5.71833, loss: 5.71833, top1: 0.14062, top5: 0.90625, batch_cost: 1.02187s, reader_cost: 0.87315, ips: 125.26108 images/sec, eta: 0:12:14
    [2021/08/15 15:51:40] root INFO: [Train][Epoch 1/20][Iter: 2/36]lr: 0.10000, CELoss: 17.05213, loss: 17.05213, top1: 0.15104, top5: 0.88021, batch_cost: 0.99999s, reader_cost: 0.85389, ips: 128.00097 images/sec, eta: 0:11:57
    [2021/08/15 15:51:41] root INFO: [Train][Epoch 1/20][Iter: 3/36]lr: 0.10000, CELoss: 23.93256, loss: 23.93256, top1: 0.17773, top5: 0.87109, batch_cost: 0.98237s, reader_cost: 0.83703, ips: 130.29691 images/sec, eta: 0:11:44
    [2021/08/15 15:51:42] root INFO: [Train][Epoch 1/20][Iter: 4/36]lr: 0.10000, CELoss: 26.75819, loss: 26.75819, top1: 0.19062, top5: 0.85000, batch_cost: 0.96942s, reader_cost: 0.82462, ips: 132.03721 images/sec, eta: 0:11:34
    [2021/08/15 15:51:43] root INFO: [Train][Epoch 1/20][Iter: 5/36]lr: 0.10000, CELoss: 25.86145, loss: 25.86145, top1: 0.18229, top5: 0.82422, batch_cost: 0.94696s, reader_cost: 0.80543, ips: 135.16918 images/sec, eta: 0:11:17
    [2021/08/15 15:51:44] root INFO: [Train][Epoch 1/20][Iter: 6/36]lr: 0.10000, CELoss: 22.83939, loss: 22.83939, top1: 0.18862, top5: 0.82366, batch_cost: 0.92861s, reader_cost: 0.78579, ips: 137.84008 images/sec, eta: 0:11:03
    [2021/08/15 15:51:45] root INFO: [Train][Epoch 1/20][Iter: 7/36]lr: 0.10000, CELoss: 20.46247, loss: 20.46247, top1: 0.19434, top5: 0.81641, batch_cost: 0.92469s, reader_cost: 0.78206, ips: 138.42422 images/sec, eta: 0:10:59
    [2021/08/15 15:51:46] root INFO: [Train][Epoch 1/20][Iter: 8/36]lr: 0.10000, CELoss: 20.46510, loss: 20.46510, top1: 0.19705, top5: 0.82899, batch_cost: 0.93105s, reader_cost: 0.78850, ips: 137.47893 images/sec, eta: 0:11:02
    [2021/08/15 15:51:47] root INFO: [Train][Epoch 1/20][Iter: 9/36]lr: 0.10000, CELoss: 21.83660, loss: 21.83660, top1: 0.18906, top5: 0.82891, batch_cost: 0.93354s, reader_cost: 0.79102, ips: 137.11184 images/sec, eta: 0:11:03
    [2021/08/15 15:51:48] root INFO: [Train][Epoch 1/20][Iter: 10/36]lr: 0.10000, CELoss: 25.14530, loss: 25.14530, top1: 0.18821, top5: 0.84233, batch_cost: 0.93546s, reader_cost: 0.79305, ips: 136.83174 images/sec, eta: 0:11:04
    [2021/08/15 15:51:49] root INFO: [Train][Epoch 1/20][Iter: 11/36]lr: 0.10000, CELoss: 25.03073, loss: 25.03073, top1: 0.18685, top5: 0.85091, batch_cost: 0.93651s, reader_cost: 0.79415, ips: 136.67820 images/sec, eta: 0:11:03
    [2021/08/15 15:51:50] root INFO: [Train][Epoch 1/20][Iter: 12/36]lr: 0.10000, CELoss: 23.57250, loss: 23.57250, top1: 0.18930, top5: 0.86178, batch_cost: 0.93315s, reader_cost: 0.79078, ips: 137.16969 images/sec, eta: 0:11:00
    [2021/08/15 15:51:50] root INFO: [Train][Epoch 1/20][Iter: 13/36]lr: 0.10000, CELoss: 22.07962, loss: 22.07962, top1: 0.19420, top5: 0.87165, batch_cost: 0.93191s, reader_cost: 0.78962, ips: 137.35217 images/sec, eta: 0:10:58
    [2021/08/15 15:51:51] root INFO: [Train][Epoch 1/20][Iter: 14/36]lr: 0.10000, CELoss: 21.22151, loss: 21.22151, top1: 0.19635, top5: 0.88021, batch_cost: 0.93037s, reader_cost: 0.78804, ips: 137.58018 images/sec, eta: 0:10:56
    [2021/08/15 15:51:52] root INFO: [Train][Epoch 1/20][Iter: 15/36]lr: 0.10000, CELoss: 20.33246, loss: 20.33246, top1: 0.19775, top5: 0.88770, batch_cost: 0.92854s, reader_cost: 0.78610, ips: 137.85050 images/sec, eta: 0:10:54
    [2021/08/15 15:51:53] root INFO: [Train][Epoch 1/20][Iter: 16/36]lr: 0.10000, CELoss: 19.38349, loss: 19.38349, top1: 0.20175, top5: 0.89430, batch_cost: 0.92805s, reader_cost: 0.78562, ips: 137.92341 images/sec, eta: 0:10:53
    [2021/08/15 15:51:54] root INFO: [Train][Epoch 1/20][Iter: 17/36]lr: 0.10000, CELoss: 18.57568, loss: 18.57568, top1: 0.20226, top5: 0.90017, batch_cost: 0.92801s, reader_cost: 0.78566, ips: 137.92896 images/sec, eta: 0:10:52
    [2021/08/15 15:51:55] root INFO: [Train][Epoch 1/20][Iter: 18/36]lr: 0.10000, CELoss: 17.89800, loss: 17.89800, top1: 0.20230, top5: 0.90543, batch_cost: 0.92994s, reader_cost: 0.78763, ips: 137.64357 images/sec, eta: 0:10:52
    [2021/08/15 15:51:56] root INFO: [Train][Epoch 1/20][Iter: 19/36]lr: 0.10000, CELoss: 17.42886, loss: 17.42886, top1: 0.20039, top5: 0.91016, batch_cost: 0.93100s, reader_cost: 0.78876, ips: 137.48659 images/sec, eta: 0:10:52
    [2021/08/15 15:51:57] root INFO: [Train][Epoch 1/20][Iter: 20/36]lr: 0.10000, CELoss: 16.68786, loss: 16.68786, top1: 0.20238, top5: 0.91443, batch_cost: 0.93139s, reader_cost: 0.78917, ips: 137.42881 images/sec, eta: 0:10:51
    [2021/08/15 15:51:58] root INFO: [Train][Epoch 1/20][Iter: 21/36]lr: 0.10000, CELoss: 16.03585, loss: 16.03585, top1: 0.20490, top5: 0.91832, batch_cost: 0.93272s, reader_cost: 0.79026, ips: 137.23332 images/sec, eta: 0:10:51
    [2021/08/15 15:51:59] root INFO: [Train][Epoch 1/20][Iter: 22/36]lr: 0.10000, CELoss: 15.62083, loss: 15.62083, top1: 0.20550, top5: 0.92188, batch_cost: 0.93355s, reader_cost: 0.79118, ips: 137.11064 images/sec, eta: 0:10:51
    [2021/08/15 15:52:00] root INFO: [Train][Epoch 1/20][Iter: 23/36]lr: 0.10000, CELoss: 15.19744, loss: 15.19744, top1: 0.20605, top5: 0.92513, batch_cost: 0.93268s, reader_cost: 0.79027, ips: 137.23906 images/sec, eta: 0:10:50
    [2021/08/15 15:52:01] root INFO: [Train][Epoch 1/20][Iter: 24/36]lr: 0.10000, CELoss: 14.67461, loss: 14.67461, top1: 0.20500, top5: 0.92812, batch_cost: 0.93239s, reader_cost: 0.79002, ips: 137.28121 images/sec, eta: 0:10:48
    [2021/08/15 15:52:02] root INFO: [Train][Epoch 1/20][Iter: 25/36]lr: 0.10000, CELoss: 14.23508, loss: 14.23508, top1: 0.20403, top5: 0.93089, batch_cost: 0.93134s, reader_cost: 0.78902, ips: 137.43704 images/sec, eta: 0:10:47
    [2021/08/15 15:52:03] root INFO: [Train][Epoch 1/20][Iter: 26/36]lr: 0.10000, CELoss: 14.15271, loss: 14.15271, top1: 0.20341, top5: 0.93345, batch_cost: 0.93178s, reader_cost: 0.78951, ips: 137.37113 images/sec, eta: 0:10:46
    [2021/08/15 15:52:04] root INFO: [Train][Epoch 1/20][Iter: 27/36]lr: 0.10000, CELoss: 13.74675, loss: 13.74675, top1: 0.20480, top5: 0.93583, batch_cost: 0.93243s, reader_cost: 0.79016, ips: 137.27545 images/sec, eta: 0:10:46
    [2021/08/15 15:52:04] root INFO: [Train][Epoch 1/20][Iter: 28/36]lr: 0.10000, CELoss: 13.33417, loss: 13.33417, top1: 0.20582, top5: 0.93804, batch_cost: 0.93243s, reader_cost: 0.79020, ips: 137.27609 images/sec, eta: 0:10:45
    [2021/08/15 15:52:05] root INFO: [Train][Epoch 1/20][Iter: 29/36]lr: 0.10000, CELoss: 12.94184, loss: 12.94184, top1: 0.20781, top5: 0.94010, batch_cost: 0.93258s, reader_cost: 0.79038, ips: 137.25315 images/sec, eta: 0:10:44
    [2021/08/15 15:52:06] root INFO: [Train][Epoch 1/20][Iter: 30/36]lr: 0.10000, CELoss: 12.59421, loss: 12.59421, top1: 0.20691, top5: 0.94204, batch_cost: 0.93164s, reader_cost: 0.78950, ips: 137.39248 images/sec, eta: 0:10:42
    [2021/08/15 15:52:07] root INFO: [Train][Epoch 1/20][Iter: 31/36]lr: 0.10000, CELoss: 12.32003, loss: 12.32003, top1: 0.20654, top5: 0.94385, batch_cost: 0.93105s, reader_cost: 0.78892, ips: 137.47919 images/sec, eta: 0:10:41
    [2021/08/15 15:52:08] root INFO: [Train][Epoch 1/20][Iter: 32/36]lr: 0.10000, CELoss: 12.09272, loss: 12.09272, top1: 0.20573, top5: 0.94555, batch_cost: 0.93215s, reader_cost: 0.79000, ips: 137.31694 images/sec, eta: 0:10:41
    [2021/08/15 15:52:09] root INFO: [Train][Epoch 1/20][Iter: 33/36]lr: 0.10000, CELoss: 11.79670, loss: 11.79670, top1: 0.20634, top5: 0.94715, batch_cost: 0.93219s, reader_cost: 0.79002, ips: 137.31095 images/sec, eta: 0:10:40
    [2021/08/15 15:52:10] root INFO: [Train][Epoch 1/20][Iter: 34/36]lr: 0.10000, CELoss: 11.58049, loss: 11.58049, top1: 0.20714, top5: 0.94866, batch_cost: 0.93303s, reader_cost: 0.79077, ips: 137.18704 images/sec, eta: 0:10:40
    [2021/08/15 15:52:10] root INFO: [Train][Epoch 1/20][Iter: 35/36]lr: 0.10000, CELoss: 11.59654, loss: 11.59654, top1: 0.20667, top5: 0.94889, batch_cost: 0.91193s, reader_cost: 0.76747, ips: 21.93141 images/sec, eta: 0:10:24
    [2021/08/15 15:52:10] root INFO: [Train][Epoch 1/20][Avg]CELoss: 11.59654, loss: 11.59654, top1: 0.20667, top5: 0.94889
    {'CELoss': {'weight': 1.0}}
    [2021/08/15 15:52:11] root INFO: [Eval][Epoch 1][Iter: 0/4]CELoss: 1714.76697, loss: 1714.76697, top1: 0.17188, top5: 1.00000, batch_cost: 0.97143s, reader_cost: 0.85629, ips: 131.76389 images/sec
    [2021/08/15 15:52:12] root INFO: [Eval][Epoch 1][Iter: 1/4]CELoss: 1692.03906, loss: 1692.03906, top1: 0.19531, top5: 1.00000, batch_cost: 0.93662s, reader_cost: 0.82158, ips: 136.66132 images/sec
    [2021/08/15 15:52:13] root INFO: [Eval][Epoch 1][Iter: 2/4]CELoss: 1708.03687, loss: 1708.03687, top1: 0.21094, top5: 1.00000, batch_cost: 0.93497s, reader_cost: 0.81968, ips: 136.90283 images/sec
    [2021/08/15 15:52:14] root INFO: [Eval][Epoch 1][Iter: 3/4]CELoss: 1239.89478, loss: 1239.89478, top1: 0.24138, top5: 1.00000, batch_cost: 0.90155s, reader_cost: 0.78858, ips: 128.66713 images/sec
    [2021/08/15 15:52:14] root INFO: [Eval][Epoch 1][Avg]CELoss: 1597.05537, loss: 1597.05537, top1: 0.20400, top5: 1.00000
    [2021/08/15 15:52:14] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:52:14] root INFO: [Eval][Epoch 1][best metric: 0.2039999989271164]
    [2021/08/15 15:52:15] root INFO: Already save model in ./output/ResNet50/epoch_1
    [2021/08/15 15:52:15] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:52:16] root INFO: [Train][Epoch 2/20][Iter: 0/36]lr: 0.10000, CELoss: 1.64499, loss: 1.64499, top1: 0.27344, top5: 1.00000, batch_cost: 1.07203s, reader_cost: 0.92764, ips: 119.39980 images/sec, eta: 0:12:13
    [2021/08/15 15:52:17] root INFO: [Train][Epoch 2/20][Iter: 1/36]lr: 0.10000, CELoss: 2.39783, loss: 2.39783, top1: 0.27344, top5: 1.00000, batch_cost: 1.06708s, reader_cost: 0.92276, ips: 119.95401 images/sec, eta: 0:12:08
    [2021/08/15 15:52:18] root INFO: [Train][Epoch 2/20][Iter: 2/36]lr: 0.10000, CELoss: 2.23298, loss: 2.23298, top1: 0.26302, top5: 1.00000, batch_cost: 1.06215s, reader_cost: 0.91793, ips: 120.51011 images/sec, eta: 0:12:04
    [2021/08/15 15:52:19] root INFO: [Train][Epoch 2/20][Iter: 3/36]lr: 0.10000, CELoss: 2.12472, loss: 2.12472, top1: 0.24414, top5: 1.00000, batch_cost: 1.05754s, reader_cost: 0.91337, ips: 121.03611 images/sec, eta: 0:12:00
    [2021/08/15 15:52:20] root INFO: [Train][Epoch 2/20][Iter: 4/36]lr: 0.10000, CELoss: 2.04738, loss: 2.04738, top1: 0.23750, top5: 1.00000, batch_cost: 1.05380s, reader_cost: 0.90968, ips: 121.46461 images/sec, eta: 0:11:56
    [2021/08/15 15:52:21] root INFO: [Train][Epoch 2/20][Iter: 5/36]lr: 0.10000, CELoss: 2.15407, loss: 2.15407, top1: 0.23307, top5: 1.00000, batch_cost: 0.94371s, reader_cost: 0.80121, ips: 135.63539 images/sec, eta: 0:10:40
    [2021/08/15 15:52:22] root INFO: [Train][Epoch 2/20][Iter: 6/36]lr: 0.10000, CELoss: 2.10650, loss: 2.10650, top1: 0.23103, top5: 1.00000, batch_cost: 0.92396s, reader_cost: 0.78223, ips: 138.53486 images/sec, eta: 0:10:26
    [2021/08/15 15:52:23] root INFO: [Train][Epoch 2/20][Iter: 7/36]lr: 0.10000, CELoss: 2.07225, loss: 2.07225, top1: 0.22754, top5: 1.00000, batch_cost: 0.92193s, reader_cost: 0.78028, ips: 138.83874 images/sec, eta: 0:10:24
    [2021/08/15 15:52:24] root INFO: [Train][Epoch 2/20][Iter: 8/36]lr: 0.10000, CELoss: 2.18764, loss: 2.18764, top1: 0.22049, top5: 1.00000, batch_cost: 0.92603s, reader_cost: 0.78434, ips: 138.22492 images/sec, eta: 0:10:25
    [2021/08/15 15:52:25] root INFO: [Train][Epoch 2/20][Iter: 9/36]lr: 0.10000, CELoss: 2.12779, loss: 2.12779, top1: 0.21797, top5: 1.00000, batch_cost: 0.92167s, reader_cost: 0.77999, ips: 138.87838 images/sec, eta: 0:10:22
    [2021/08/15 15:52:26] root INFO: [Train][Epoch 2/20][Iter: 10/36]lr: 0.10000, CELoss: 2.09612, loss: 2.09612, top1: 0.21378, top5: 1.00000, batch_cost: 0.92781s, reader_cost: 0.78591, ips: 137.95958 images/sec, eta: 0:10:25
    [2021/08/15 15:52:27] root INFO: [Train][Epoch 2/20][Iter: 11/36]lr: 0.10000, CELoss: 2.07644, loss: 2.07644, top1: 0.21094, top5: 1.00000, batch_cost: 0.93081s, reader_cost: 0.78904, ips: 137.51532 images/sec, eta: 0:10:26
    [2021/08/15 15:52:27] root INFO: [Train][Epoch 2/20][Iter: 12/36]lr: 0.10000, CELoss: 2.09462, loss: 2.09462, top1: 0.21334, top5: 1.00000, batch_cost: 0.92798s, reader_cost: 0.78624, ips: 137.93328 images/sec, eta: 0:10:23
    [2021/08/15 15:52:28] root INFO: [Train][Epoch 2/20][Iter: 13/36]lr: 0.10000, CELoss: 2.18512, loss: 2.18512, top1: 0.20926, top5: 1.00000, batch_cost: 0.92474s, reader_cost: 0.78303, ips: 138.41680 images/sec, eta: 0:10:20
    [2021/08/15 15:52:29] root INFO: [Train][Epoch 2/20][Iter: 14/36]lr: 0.10000, CELoss: 2.15340, loss: 2.15340, top1: 0.20729, top5: 1.00000, batch_cost: 0.92214s, reader_cost: 0.78048, ips: 138.80809 images/sec, eta: 0:10:17
    [2021/08/15 15:52:30] root INFO: [Train][Epoch 2/20][Iter: 15/36]lr: 0.10000, CELoss: 2.29704, loss: 2.29704, top1: 0.20996, top5: 1.00000, batch_cost: 0.92365s, reader_cost: 0.78193, ips: 138.58009 images/sec, eta: 0:10:17
    [2021/08/15 15:52:31] root INFO: [Train][Epoch 2/20][Iter: 16/36]lr: 0.10000, CELoss: 2.25667, loss: 2.25667, top1: 0.20496, top5: 1.00000, batch_cost: 0.92635s, reader_cost: 0.78463, ips: 138.17626 images/sec, eta: 0:10:18
    [2021/08/15 15:52:32] root INFO: [Train][Epoch 2/20][Iter: 17/36]lr: 0.10000, CELoss: 2.22094, loss: 2.22094, top1: 0.20443, top5: 1.00000, batch_cost: 0.92728s, reader_cost: 0.78557, ips: 138.03859 images/sec, eta: 0:10:18
    [2021/08/15 15:52:33] root INFO: [Train][Epoch 2/20][Iter: 18/36]lr: 0.10000, CELoss: 2.18884, loss: 2.18884, top1: 0.20148, top5: 1.00000, batch_cost: 0.92654s, reader_cost: 0.78488, ips: 138.14810 images/sec, eta: 0:10:17
    [2021/08/15 15:52:34] root INFO: [Train][Epoch 2/20][Iter: 19/36]lr: 0.10000, CELoss: 2.20284, loss: 2.20284, top1: 0.20078, top5: 1.00000, batch_cost: 0.92848s, reader_cost: 0.78686, ips: 137.85957 images/sec, eta: 0:10:17
    [2021/08/15 15:52:35] root INFO: [Train][Epoch 2/20][Iter: 20/36]lr: 0.10000, CELoss: 2.18175, loss: 2.18175, top1: 0.20126, top5: 1.00000, batch_cost: 0.92901s, reader_cost: 0.78744, ips: 137.78082 images/sec, eta: 0:10:16
    [2021/08/15 15:52:36] root INFO: [Train][Epoch 2/20][Iter: 21/36]lr: 0.10000, CELoss: 2.18599, loss: 2.18599, top1: 0.20419, top5: 1.00000, batch_cost: 0.92717s, reader_cost: 0.78563, ips: 138.05460 images/sec, eta: 0:10:14
    [2021/08/15 15:52:37] root INFO: [Train][Epoch 2/20][Iter: 22/36]lr: 0.10000, CELoss: 2.16108, loss: 2.16108, top1: 0.20380, top5: 1.00000, batch_cost: 0.92835s, reader_cost: 0.78677, ips: 137.87932 images/sec, eta: 0:10:14
    [2021/08/15 15:52:38] root INFO: [Train][Epoch 2/20][Iter: 23/36]lr: 0.10000, CELoss: 2.14355, loss: 2.14355, top1: 0.20410, top5: 1.00000, batch_cost: 0.92937s, reader_cost: 0.78780, ips: 137.72764 images/sec, eta: 0:10:14
    [2021/08/15 15:52:39] root INFO: [Train][Epoch 2/20][Iter: 24/36]lr: 0.10000, CELoss: 2.12201, loss: 2.12201, top1: 0.20531, top5: 1.00000, batch_cost: 0.93016s, reader_cost: 0.78865, ips: 137.61144 images/sec, eta: 0:10:13
    [2021/08/15 15:52:40] root INFO: [Train][Epoch 2/20][Iter: 25/36]lr: 0.10000, CELoss: 2.11372, loss: 2.11372, top1: 0.20673, top5: 1.00000, batch_cost: 0.93103s, reader_cost: 0.78957, ips: 137.48190 images/sec, eta: 0:10:13
    [2021/08/15 15:52:41] root INFO: [Train][Epoch 2/20][Iter: 26/36]lr: 0.10000, CELoss: 2.10029, loss: 2.10029, top1: 0.20544, top5: 1.00000, batch_cost: 0.93279s, reader_cost: 0.79134, ips: 137.22206 images/sec, eta: 0:10:13
    [2021/08/15 15:52:42] root INFO: [Train][Epoch 2/20][Iter: 27/36]lr: 0.10000, CELoss: 2.08893, loss: 2.08893, top1: 0.20452, top5: 1.00000, batch_cost: 0.93287s, reader_cost: 0.79141, ips: 137.21169 images/sec, eta: 0:10:12
    [2021/08/15 15:52:42] root INFO: [Train][Epoch 2/20][Iter: 28/36]lr: 0.10000, CELoss: 2.07969, loss: 2.07969, top1: 0.20501, top5: 1.00000, batch_cost: 0.93215s, reader_cost: 0.79069, ips: 137.31635 images/sec, eta: 0:10:11
    [2021/08/15 15:52:43] root INFO: [Train][Epoch 2/20][Iter: 29/36]lr: 0.10000, CELoss: 2.07211, loss: 2.07211, top1: 0.20599, top5: 1.00000, batch_cost: 0.93181s, reader_cost: 0.79033, ips: 137.36676 images/sec, eta: 0:10:10
    [2021/08/15 15:52:44] root INFO: [Train][Epoch 2/20][Iter: 30/36]lr: 0.10000, CELoss: 2.05657, loss: 2.05657, top1: 0.20716, top5: 1.00000, batch_cost: 0.93086s, reader_cost: 0.78936, ips: 137.50778 images/sec, eta: 0:10:08
    [2021/08/15 15:52:45] root INFO: [Train][Epoch 2/20][Iter: 31/36]lr: 0.10000, CELoss: 2.04206, loss: 2.04206, top1: 0.20752, top5: 1.00000, batch_cost: 0.93050s, reader_cost: 0.78897, ips: 137.56031 images/sec, eta: 0:10:07
    [2021/08/15 15:52:46] root INFO: [Train][Epoch 2/20][Iter: 32/36]lr: 0.10000, CELoss: 2.03691, loss: 2.03691, top1: 0.20620, top5: 1.00000, batch_cost: 0.93082s, reader_cost: 0.78932, ips: 137.51283 images/sec, eta: 0:10:06
    [2021/08/15 15:52:47] root INFO: [Train][Epoch 2/20][Iter: 33/36]lr: 0.10000, CELoss: 2.02405, loss: 2.02405, top1: 0.20887, top5: 1.00000, batch_cost: 0.93267s, reader_cost: 0.79119, ips: 137.24033 images/sec, eta: 0:10:07
    [2021/08/15 15:52:48] root INFO: [Train][Epoch 2/20][Iter: 34/36]lr: 0.10000, CELoss: 2.01249, loss: 2.01249, top1: 0.20826, top5: 1.00000, batch_cost: 0.93162s, reader_cost: 0.79013, ips: 137.39474 images/sec, eta: 0:10:05
    [2021/08/15 15:52:48] root INFO: [Train][Epoch 2/20][Iter: 35/36]lr: 0.10000, CELoss: 2.01113, loss: 2.01113, top1: 0.20822, top5: 1.00000, batch_cost: 0.91064s, reader_cost: 0.76621, ips: 21.96260 images/sec, eta: 0:09:51
    [2021/08/15 15:52:48] root INFO: [Train][Epoch 2/20][Avg]CELoss: 2.01113, loss: 2.01113, top1: 0.20822, top5: 1.00000
    [2021/08/15 15:52:49] root INFO: [Eval][Epoch 2][Iter: 0/4]CELoss: 2.25138, loss: 2.25138, top1: 0.21094, top5: 1.00000, batch_cost: 0.98217s, reader_cost: 0.86799, ips: 130.32404 images/sec
    [2021/08/15 15:52:50] root INFO: [Eval][Epoch 2][Iter: 1/4]CELoss: 1.65912, loss: 1.65912, top1: 0.21094, top5: 1.00000, batch_cost: 0.93402s, reader_cost: 0.81967, ips: 137.04185 images/sec
    [2021/08/15 15:52:51] root INFO: [Eval][Epoch 2][Iter: 2/4]CELoss: 1.62295, loss: 1.62295, top1: 0.15625, top5: 1.00000, batch_cost: 0.92033s, reader_cost: 0.80536, ips: 139.08025 images/sec
    [2021/08/15 15:52:52] root INFO: [Eval][Epoch 2][Iter: 3/4]CELoss: 1.60953, loss: 1.60953, top1: 0.25000, top5: 1.00000, batch_cost: 0.89004s, reader_cost: 0.77722, ips: 130.33102 images/sec
    [2021/08/15 15:52:52] root INFO: [Eval][Epoch 2][Avg]CELoss: 1.78998, loss: 1.78998, top1: 0.20600, top5: 1.00000
    [2021/08/15 15:52:53] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:52:53] root INFO: [Eval][Epoch 2][best metric: 0.206]
    [2021/08/15 15:52:53] root INFO: Already save model in ./output/ResNet50/epoch_2
    [2021/08/15 15:52:54] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:52:55] root INFO: [Train][Epoch 3/20][Iter: 0/36]lr: 0.10000, CELoss: 2.20788, loss: 2.20788, top1: 0.19531, top5: 1.00000, batch_cost: 1.09304s, reader_cost: 0.94838, ips: 117.10498 images/sec, eta: 0:11:48
    [2021/08/15 15:52:56] root INFO: [Train][Epoch 3/20][Iter: 1/36]lr: 0.10000, CELoss: 1.95124, loss: 1.95124, top1: 0.17188, top5: 1.00000, batch_cost: 1.08755s, reader_cost: 0.94297, ips: 117.69620 images/sec, eta: 0:11:43
    [2021/08/15 15:52:57] root INFO: [Train][Epoch 3/20][Iter: 2/36]lr: 0.10000, CELoss: 1.91973, loss: 1.91973, top1: 0.17708, top5: 1.00000, batch_cost: 1.08191s, reader_cost: 0.93743, ips: 118.30932 images/sec, eta: 0:11:38
    [2021/08/15 15:52:58] root INFO: [Train][Epoch 3/20][Iter: 3/36]lr: 0.10000, CELoss: 1.88827, loss: 1.88827, top1: 0.18164, top5: 1.00000, batch_cost: 1.07693s, reader_cost: 0.93251, ips: 118.85594 images/sec, eta: 0:11:34
    [2021/08/15 15:52:59] root INFO: [Train][Epoch 3/20][Iter: 4/36]lr: 0.10000, CELoss: 2.02119, loss: 2.02119, top1: 0.18906, top5: 1.00000, batch_cost: 1.07178s, reader_cost: 0.92744, ips: 119.42741 images/sec, eta: 0:11:30
    [2021/08/15 15:53:00] root INFO: [Train][Epoch 3/20][Iter: 5/36]lr: 0.10000, CELoss: 1.95290, loss: 1.95290, top1: 0.18490, top5: 1.00000, batch_cost: 0.89528s, reader_cost: 0.75432, ips: 142.97144 images/sec, eta: 0:09:35
    [2021/08/15 15:53:00] root INFO: [Train][Epoch 3/20][Iter: 6/36]lr: 0.10000, CELoss: 1.92954, loss: 1.92954, top1: 0.18750, top5: 1.00000, batch_cost: 0.91643s, reader_cost: 0.77507, ips: 139.67286 images/sec, eta: 0:09:48
    [2021/08/15 15:53:01] root INFO: [Train][Epoch 3/20][Iter: 7/36]lr: 0.10000, CELoss: 1.92274, loss: 1.92274, top1: 0.18945, top5: 1.00000, batch_cost: 0.91081s, reader_cost: 0.76947, ips: 140.53466 images/sec, eta: 0:09:43
    [2021/08/15 15:53:02] root INFO: [Train][Epoch 3/20][Iter: 8/36]lr: 0.10000, CELoss: 1.88946, loss: 1.88946, top1: 0.19705, top5: 1.00000, batch_cost: 0.91010s, reader_cost: 0.76862, ips: 140.64357 images/sec, eta: 0:09:42
    [2021/08/15 15:53:03] root INFO: [Train][Epoch 3/20][Iter: 9/36]lr: 0.10000, CELoss: 1.87423, loss: 1.87423, top1: 0.19609, top5: 1.00000, batch_cost: 0.90852s, reader_cost: 0.76700, ips: 140.88847 images/sec, eta: 0:09:40
    [2021/08/15 15:53:04] root INFO: [Train][Epoch 3/20][Iter: 10/36]lr: 0.10000, CELoss: 1.87079, loss: 1.87079, top1: 0.19247, top5: 1.00000, batch_cost: 0.90550s, reader_cost: 0.76403, ips: 141.35813 images/sec, eta: 0:09:37
    [2021/08/15 15:53:05] root INFO: [Train][Epoch 3/20][Iter: 11/36]lr: 0.10000, CELoss: 1.86889, loss: 1.86889, top1: 0.19336, top5: 1.00000, batch_cost: 0.90611s, reader_cost: 0.76468, ips: 141.26363 images/sec, eta: 0:09:37
    [2021/08/15 15:53:06] root INFO: [Train][Epoch 3/20][Iter: 12/36]lr: 0.10000, CELoss: 1.85555, loss: 1.85555, top1: 0.19531, top5: 1.00000, batch_cost: 0.90663s, reader_cost: 0.76529, ips: 141.18149 images/sec, eta: 0:09:36
    [2021/08/15 15:53:07] root INFO: [Train][Epoch 3/20][Iter: 13/36]lr: 0.10000, CELoss: 1.84161, loss: 1.84161, top1: 0.19810, top5: 1.00000, batch_cost: 0.90682s, reader_cost: 0.76558, ips: 141.15229 images/sec, eta: 0:09:35
    [2021/08/15 15:53:08] root INFO: [Train][Epoch 3/20][Iter: 14/36]lr: 0.10000, CELoss: 1.83432, loss: 1.83432, top1: 0.19792, top5: 1.00000, batch_cost: 0.91042s, reader_cost: 0.76912, ips: 140.59466 images/sec, eta: 0:09:37
    [2021/08/15 15:53:09] root INFO: [Train][Epoch 3/20][Iter: 15/36]lr: 0.10000, CELoss: 1.83031, loss: 1.83031, top1: 0.19922, top5: 1.00000, batch_cost: 0.91040s, reader_cost: 0.76914, ips: 140.59702 images/sec, eta: 0:09:36
    [2021/08/15 15:53:10] root INFO: [Train][Epoch 3/20][Iter: 16/36]lr: 0.10000, CELoss: 1.83719, loss: 1.83719, top1: 0.19807, top5: 1.00000, batch_cost: 0.90955s, reader_cost: 0.76839, ips: 140.72969 images/sec, eta: 0:09:34
    [2021/08/15 15:53:11] root INFO: [Train][Epoch 3/20][Iter: 17/36]lr: 0.10000, CELoss: 1.83379, loss: 1.83379, top1: 0.19878, top5: 1.00000, batch_cost: 0.91313s, reader_cost: 0.77194, ips: 140.17754 images/sec, eta: 0:09:36
    [2021/08/15 15:53:11] root INFO: [Train][Epoch 3/20][Iter: 18/36]lr: 0.10000, CELoss: 1.82619, loss: 1.82619, top1: 0.19819, top5: 1.00000, batch_cost: 0.91448s, reader_cost: 0.77324, ips: 139.96960 images/sec, eta: 0:09:36
    [2021/08/15 15:53:12] root INFO: [Train][Epoch 3/20][Iter: 19/36]lr: 0.10000, CELoss: 1.82790, loss: 1.82790, top1: 0.19727, top5: 1.00000, batch_cost: 0.91419s, reader_cost: 0.77288, ips: 140.01393 images/sec, eta: 0:09:35
    [2021/08/15 15:53:13] root INFO: [Train][Epoch 3/20][Iter: 20/36]lr: 0.10000, CELoss: 1.81925, loss: 1.81925, top1: 0.19717, top5: 1.00000, batch_cost: 0.91332s, reader_cost: 0.77201, ips: 140.14869 images/sec, eta: 0:09:33
    [2021/08/15 15:53:14] root INFO: [Train][Epoch 3/20][Iter: 21/36]lr: 0.10000, CELoss: 1.81721, loss: 1.81721, top1: 0.19389, top5: 1.00000, batch_cost: 0.91334s, reader_cost: 0.77204, ips: 140.14428 images/sec, eta: 0:09:32
    [2021/08/15 15:53:15] root INFO: [Train][Epoch 3/20][Iter: 22/36]lr: 0.10000, CELoss: 1.81070, loss: 1.81070, top1: 0.19293, top5: 1.00000, batch_cost: 0.91307s, reader_cost: 0.77174, ips: 140.18616 images/sec, eta: 0:09:31
    [2021/08/15 15:53:16] root INFO: [Train][Epoch 3/20][Iter: 23/36]lr: 0.10000, CELoss: 1.81123, loss: 1.81123, top1: 0.19206, top5: 1.00000, batch_cost: 0.91244s, reader_cost: 0.77116, ips: 140.28254 images/sec, eta: 0:09:30
    [2021/08/15 15:53:17] root INFO: [Train][Epoch 3/20][Iter: 24/36]lr: 0.10000, CELoss: 1.80514, loss: 1.80514, top1: 0.19375, top5: 1.00000, batch_cost: 0.91211s, reader_cost: 0.77082, ips: 140.33393 images/sec, eta: 0:09:29
    [2021/08/15 15:53:18] root INFO: [Train][Epoch 3/20][Iter: 25/36]lr: 0.10000, CELoss: 1.80636, loss: 1.80636, top1: 0.19321, top5: 1.00000, batch_cost: 0.91118s, reader_cost: 0.76991, ips: 140.47789 images/sec, eta: 0:09:27
    [2021/08/15 15:53:19] root INFO: [Train][Epoch 3/20][Iter: 26/36]lr: 0.10000, CELoss: 1.79903, loss: 1.79903, top1: 0.19271, top5: 1.00000, batch_cost: 0.90993s, reader_cost: 0.76870, ips: 140.67049 images/sec, eta: 0:09:25
    [2021/08/15 15:53:20] root INFO: [Train][Epoch 3/20][Iter: 27/36]lr: 0.10000, CELoss: 1.79149, loss: 1.79149, top1: 0.19559, top5: 1.00000, batch_cost: 0.90975s, reader_cost: 0.76850, ips: 140.69844 images/sec, eta: 0:09:24
    [2021/08/15 15:53:20] root INFO: [Train][Epoch 3/20][Iter: 28/36]lr: 0.10000, CELoss: 1.78789, loss: 1.78789, top1: 0.19477, top5: 1.00000, batch_cost: 0.90974s, reader_cost: 0.76843, ips: 140.69993 images/sec, eta: 0:09:24
    [2021/08/15 15:53:21] root INFO: [Train][Epoch 3/20][Iter: 29/36]lr: 0.10000, CELoss: 1.78434, loss: 1.78434, top1: 0.19453, top5: 1.00000, batch_cost: 0.90993s, reader_cost: 0.76856, ips: 140.67012 images/sec, eta: 0:09:23
    [2021/08/15 15:53:22] root INFO: [Train][Epoch 3/20][Iter: 30/36]lr: 0.10000, CELoss: 1.78060, loss: 1.78060, top1: 0.19430, top5: 1.00000, batch_cost: 0.91048s, reader_cost: 0.76907, ips: 140.58532 images/sec, eta: 0:09:22
    [2021/08/15 15:53:23] root INFO: [Train][Epoch 3/20][Iter: 31/36]lr: 0.10000, CELoss: 1.77678, loss: 1.77678, top1: 0.19336, top5: 1.00000, batch_cost: 0.91015s, reader_cost: 0.76874, ips: 140.63676 images/sec, eta: 0:09:21
    [2021/08/15 15:53:24] root INFO: [Train][Epoch 3/20][Iter: 32/36]lr: 0.10000, CELoss: 1.77236, loss: 1.77236, top1: 0.19271, top5: 1.00000, batch_cost: 0.91007s, reader_cost: 0.76869, ips: 140.64906 images/sec, eta: 0:09:20
    [2021/08/15 15:53:25] root INFO: [Train][Epoch 3/20][Iter: 33/36]lr: 0.10000, CELoss: 1.76835, loss: 1.76835, top1: 0.19508, top5: 1.00000, batch_cost: 0.91077s, reader_cost: 0.76936, ips: 140.54081 images/sec, eta: 0:09:20
    [2021/08/15 15:53:26] root INFO: [Train][Epoch 3/20][Iter: 34/36]lr: 0.10000, CELoss: 1.76283, loss: 1.76283, top1: 0.19643, top5: 1.00000, batch_cost: 0.91058s, reader_cost: 0.76915, ips: 140.57018 images/sec, eta: 0:09:19
    [2021/08/15 15:53:26] root INFO: [Train][Epoch 3/20][Iter: 35/36]lr: 0.10000, CELoss: 1.76200, loss: 1.76200, top1: 0.19689, top5: 1.00000, batch_cost: 0.89024s, reader_cost: 0.74606, ips: 22.46580 images/sec, eta: 0:09:05
    [2021/08/15 15:53:26] root INFO: [Train][Epoch 3/20][Avg]CELoss: 1.76200, loss: 1.76200, top1: 0.19689, top5: 1.00000
    [2021/08/15 15:53:27] root INFO: [Eval][Epoch 3][Iter: 0/4]CELoss: 1.61704, loss: 1.61704, top1: 0.17188, top5: 1.00000, batch_cost: 0.94381s, reader_cost: 0.82754, ips: 135.61997 images/sec
    [2021/08/15 15:53:28] root INFO: [Eval][Epoch 3][Iter: 1/4]CELoss: 2.08936, loss: 2.08936, top1: 0.25000, top5: 1.00000, batch_cost: 0.91969s, reader_cost: 0.80406, ips: 139.17751 images/sec
    [2021/08/15 15:53:29] root INFO: [Eval][Epoch 3][Iter: 2/4]CELoss: 2.08809, loss: 2.08809, top1: 0.17969, top5: 1.00000, batch_cost: 0.92424s, reader_cost: 0.80881, ips: 138.49289 images/sec
    [2021/08/15 15:53:30] root INFO: [Eval][Epoch 3][Iter: 3/4]CELoss: 1.63187, loss: 1.63187, top1: 0.19828, top5: 1.00000, batch_cost: 0.90396s, reader_cost: 0.79045, ips: 128.32367 images/sec
    [2021/08/15 15:53:30] root INFO: [Eval][Epoch 3][Avg]CELoss: 1.86198, loss: 1.86198, top1: 0.20000, top5: 1.00000
    [2021/08/15 15:53:30] root INFO: [Eval][Epoch 3][best metric: 0.206]
    [2021/08/15 15:53:30] root INFO: Already save model in ./output/ResNet50/epoch_3
    [2021/08/15 15:53:31] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:53:32] root INFO: [Train][Epoch 4/20][Iter: 0/36]lr: 0.10000, CELoss: 1.60495, loss: 1.60495, top1: 0.17188, top5: 1.00000, batch_cost: 1.04598s, reader_cost: 0.90185, ips: 122.37380 images/sec, eta: 0:10:40
    [2021/08/15 15:53:33] root INFO: [Train][Epoch 4/20][Iter: 1/36]lr: 0.10000, CELoss: 1.71818, loss: 1.71818, top1: 0.18750, top5: 1.00000, batch_cost: 1.04262s, reader_cost: 0.89856, ips: 122.76762 images/sec, eta: 0:10:37
    [2021/08/15 15:53:34] root INFO: [Train][Epoch 4/20][Iter: 2/36]lr: 0.10000, CELoss: 1.73675, loss: 1.73675, top1: 0.21875, top5: 1.00000, batch_cost: 1.03950s, reader_cost: 0.89553, ips: 123.13599 images/sec, eta: 0:10:34
    [2021/08/15 15:53:35] root INFO: [Train][Epoch 4/20][Iter: 3/36]lr: 0.10000, CELoss: 1.73487, loss: 1.73487, top1: 0.24805, top5: 1.00000, batch_cost: 1.03616s, reader_cost: 0.89230, ips: 123.53333 images/sec, eta: 0:10:31
    [2021/08/15 15:53:36] root INFO: [Train][Epoch 4/20][Iter: 4/36]lr: 0.10000, CELoss: 1.72428, loss: 1.72428, top1: 0.23750, top5: 1.00000, batch_cost: 1.03263s, reader_cost: 0.88884, ips: 123.95561 images/sec, eta: 0:10:27
    [2021/08/15 15:53:37] root INFO: [Train][Epoch 4/20][Iter: 5/36]lr: 0.10000, CELoss: 1.70642, loss: 1.70642, top1: 0.23177, top5: 1.00000, batch_cost: 0.90090s, reader_cost: 0.75951, ips: 142.07993 images/sec, eta: 0:09:06
    [2021/08/15 15:53:38] root INFO: [Train][Epoch 4/20][Iter: 6/36]lr: 0.10000, CELoss: 1.70627, loss: 1.70627, top1: 0.22433, top5: 1.00000, batch_cost: 0.92236s, reader_cost: 0.78115, ips: 138.77456 images/sec, eta: 0:09:18
    [2021/08/15 15:53:39] root INFO: [Train][Epoch 4/20][Iter: 7/36]lr: 0.10000, CELoss: 1.80558, loss: 1.80558, top1: 0.23047, top5: 1.00000, batch_cost: 0.93041s, reader_cost: 0.78904, ips: 137.57412 images/sec, eta: 0:09:22
    [2021/08/15 15:53:40] root INFO: [Train][Epoch 4/20][Iter: 8/36]lr: 0.10000, CELoss: 1.78196, loss: 1.78196, top1: 0.23003, top5: 1.00000, batch_cost: 0.92978s, reader_cost: 0.78860, ips: 137.66697 images/sec, eta: 0:09:21
    [2021/08/15 15:53:41] root INFO: [Train][Epoch 4/20][Iter: 9/36]lr: 0.10000, CELoss: 1.77155, loss: 1.77155, top1: 0.22422, top5: 1.00000, batch_cost: 0.93312s, reader_cost: 0.79162, ips: 137.17471 images/sec, eta: 0:09:22
    [2021/08/15 15:53:41] root INFO: [Train][Epoch 4/20][Iter: 10/36]lr: 0.10000, CELoss: 1.75618, loss: 1.75618, top1: 0.21946, top5: 1.00000, batch_cost: 0.92894s, reader_cost: 0.78736, ips: 137.79192 images/sec, eta: 0:09:19
    [2021/08/15 15:53:42] root INFO: [Train][Epoch 4/20][Iter: 11/36]lr: 0.10000, CELoss: 1.75293, loss: 1.75293, top1: 0.21810, top5: 1.00000, batch_cost: 0.93089s, reader_cost: 0.78920, ips: 137.50279 images/sec, eta: 0:09:19
    [2021/08/15 15:53:43] root INFO: [Train][Epoch 4/20][Iter: 12/36]lr: 0.10000, CELoss: 1.76134, loss: 1.76134, top1: 0.21695, top5: 1.00000, batch_cost: 0.93164s, reader_cost: 0.78974, ips: 137.39164 images/sec, eta: 0:09:18
    [2021/08/15 15:53:44] root INFO: [Train][Epoch 4/20][Iter: 13/36]lr: 0.10000, CELoss: 1.75667, loss: 1.75667, top1: 0.21317, top5: 1.00000, batch_cost: 0.92802s, reader_cost: 0.78593, ips: 137.92780 images/sec, eta: 0:09:15
    [2021/08/15 15:53:45] root INFO: [Train][Epoch 4/20][Iter: 14/36]lr: 0.10000, CELoss: 1.75008, loss: 1.75008, top1: 0.21563, top5: 1.00000, batch_cost: 0.92965s, reader_cost: 0.78754, ips: 137.68611 images/sec, eta: 0:09:15
    [2021/08/15 15:53:46] root INFO: [Train][Epoch 4/20][Iter: 15/36]lr: 0.10000, CELoss: 1.74126, loss: 1.74126, top1: 0.21729, top5: 1.00000, batch_cost: 0.93081s, reader_cost: 0.78876, ips: 137.51438 images/sec, eta: 0:09:15
    [2021/08/15 15:53:47] root INFO: [Train][Epoch 4/20][Iter: 16/36]lr: 0.10000, CELoss: 1.73462, loss: 1.73462, top1: 0.22289, top5: 1.00000, batch_cost: 0.93183s, reader_cost: 0.78981, ips: 137.36483 images/sec, eta: 0:09:15
    [2021/08/15 15:53:48] root INFO: [Train][Epoch 4/20][Iter: 17/36]lr: 0.10000, CELoss: 1.73947, loss: 1.73947, top1: 0.22266, top5: 1.00000, batch_cost: 0.93239s, reader_cost: 0.79045, ips: 137.28172 images/sec, eta: 0:09:14
    [2021/08/15 15:53:49] root INFO: [Train][Epoch 4/20][Iter: 18/36]lr: 0.10000, CELoss: 1.74679, loss: 1.74679, top1: 0.22245, top5: 1.00000, batch_cost: 0.93277s, reader_cost: 0.79082, ips: 137.22525 images/sec, eta: 0:09:14
    [2021/08/15 15:53:50] root INFO: [Train][Epoch 4/20][Iter: 19/36]lr: 0.10000, CELoss: 1.75110, loss: 1.75110, top1: 0.22266, top5: 1.00000, batch_cost: 0.93168s, reader_cost: 0.78969, ips: 137.38588 images/sec, eta: 0:09:12
    [2021/08/15 15:53:51] root INFO: [Train][Epoch 4/20][Iter: 20/36]lr: 0.10000, CELoss: 1.75073, loss: 1.75073, top1: 0.22061, top5: 1.00000, batch_cost: 0.93001s, reader_cost: 0.78805, ips: 137.63317 images/sec, eta: 0:09:10
    [2021/08/15 15:53:52] root INFO: [Train][Epoch 4/20][Iter: 21/36]lr: 0.10000, CELoss: 1.74697, loss: 1.74697, top1: 0.21982, top5: 1.00000, batch_cost: 0.93055s, reader_cost: 0.78862, ips: 137.55334 images/sec, eta: 0:09:09
    [2021/08/15 15:53:53] root INFO: [Train][Epoch 4/20][Iter: 22/36]lr: 0.10000, CELoss: 1.74182, loss: 1.74182, top1: 0.21739, top5: 1.00000, batch_cost: 0.93086s, reader_cost: 0.78899, ips: 137.50736 images/sec, eta: 0:09:09
    [2021/08/15 15:53:53] root INFO: [Train][Epoch 4/20][Iter: 23/36]lr: 0.10000, CELoss: 1.74032, loss: 1.74032, top1: 0.21745, top5: 1.00000, batch_cost: 0.92875s, reader_cost: 0.78688, ips: 137.81966 images/sec, eta: 0:09:07
    [2021/08/15 15:53:54] root INFO: [Train][Epoch 4/20][Iter: 24/36]lr: 0.10000, CELoss: 1.73580, loss: 1.73580, top1: 0.21594, top5: 1.00000, batch_cost: 0.92681s, reader_cost: 0.78492, ips: 138.10868 images/sec, eta: 0:09:04
    [2021/08/15 15:53:55] root INFO: [Train][Epoch 4/20][Iter: 25/36]lr: 0.10000, CELoss: 1.73111, loss: 1.73111, top1: 0.21695, top5: 1.00000, batch_cost: 0.92706s, reader_cost: 0.78518, ips: 138.07038 images/sec, eta: 0:09:04
    [2021/08/15 15:53:56] root INFO: [Train][Epoch 4/20][Iter: 26/36]lr: 0.10000, CELoss: 1.73483, loss: 1.73483, top1: 0.21875, top5: 1.00000, batch_cost: 0.92611s, reader_cost: 0.78424, ips: 138.21296 images/sec, eta: 0:09:02
    [2021/08/15 15:53:57] root INFO: [Train][Epoch 4/20][Iter: 27/36]lr: 0.10000, CELoss: 1.73385, loss: 1.73385, top1: 0.22238, top5: 1.00000, batch_cost: 0.92633s, reader_cost: 0.78451, ips: 138.17984 images/sec, eta: 0:09:01
    [2021/08/15 15:53:58] root INFO: [Train][Epoch 4/20][Iter: 28/36]lr: 0.10000, CELoss: 1.73827, loss: 1.73827, top1: 0.22495, top5: 1.00000, batch_cost: 0.92593s, reader_cost: 0.78414, ips: 138.23958 images/sec, eta: 0:09:00
    [2021/08/15 15:53:59] root INFO: [Train][Epoch 4/20][Iter: 29/36]lr: 0.10000, CELoss: 1.73481, loss: 1.73481, top1: 0.22604, top5: 1.00000, batch_cost: 0.92451s, reader_cost: 0.78275, ips: 138.45241 images/sec, eta: 0:08:58
    [2021/08/15 15:54:00] root INFO: [Train][Epoch 4/20][Iter: 30/36]lr: 0.10000, CELoss: 1.73226, loss: 1.73226, top1: 0.22631, top5: 1.00000, batch_cost: 0.92346s, reader_cost: 0.78165, ips: 138.60850 images/sec, eta: 0:08:57
    [2021/08/15 15:54:01] root INFO: [Train][Epoch 4/20][Iter: 31/36]lr: 0.10000, CELoss: 1.73304, loss: 1.73304, top1: 0.22632, top5: 1.00000, batch_cost: 0.92319s, reader_cost: 0.78138, ips: 138.64956 images/sec, eta: 0:08:56
    [2021/08/15 15:54:02] root INFO: [Train][Epoch 4/20][Iter: 32/36]lr: 0.10000, CELoss: 1.73368, loss: 1.73368, top1: 0.22467, top5: 1.00000, batch_cost: 0.92335s, reader_cost: 0.78153, ips: 138.62538 images/sec, eta: 0:08:55
    [2021/08/15 15:54:03] root INFO: [Train][Epoch 4/20][Iter: 33/36]lr: 0.10000, CELoss: 1.73438, loss: 1.73438, top1: 0.22312, top5: 1.00000, batch_cost: 0.92273s, reader_cost: 0.78093, ips: 138.71920 images/sec, eta: 0:08:54
    [2021/08/15 15:54:04] root INFO: [Train][Epoch 4/20][Iter: 34/36]lr: 0.10000, CELoss: 1.73027, loss: 1.73027, top1: 0.22321, top5: 1.00000, batch_cost: 0.92303s, reader_cost: 0.78119, ips: 138.67389 images/sec, eta: 0:08:53
    [2021/08/15 15:54:04] root INFO: [Train][Epoch 4/20][Iter: 35/36]lr: 0.10000, CELoss: 1.72982, loss: 1.72982, top1: 0.22333, top5: 1.00000, batch_cost: 0.90231s, reader_cost: 0.75751, ips: 22.16525 images/sec, eta: 0:08:40
    [2021/08/15 15:54:04] root INFO: [Train][Epoch 4/20][Avg]CELoss: 1.72982, loss: 1.72982, top1: 0.22333, top5: 1.00000
    [2021/08/15 15:54:05] root INFO: [Eval][Epoch 4][Iter: 0/4]CELoss: 1.60591, loss: 1.60591, top1: 0.27344, top5: 1.00000, batch_cost: 0.93888s, reader_cost: 0.82370, ips: 136.33200 images/sec
    [2021/08/15 15:54:06] root INFO: [Eval][Epoch 4][Iter: 1/4]CELoss: 1.88130, loss: 1.88130, top1: 0.21094, top5: 1.00000, batch_cost: 0.91158s, reader_cost: 0.79669, ips: 140.41564 images/sec
    [2021/08/15 15:54:07] root INFO: [Eval][Epoch 4][Iter: 2/4]CELoss: 1.58505, loss: 1.58505, top1: 0.35938, top5: 1.00000, batch_cost: 0.90186s, reader_cost: 0.78695, ips: 141.92968 images/sec
    [2021/08/15 15:54:07] root INFO: [Eval][Epoch 4][Iter: 3/4]CELoss: 1.67788, loss: 1.67788, top1: 0.25862, top5: 1.00000, batch_cost: 0.87612s, reader_cost: 0.76322, ips: 132.40226 images/sec
    [2021/08/15 15:54:07] root INFO: [Eval][Epoch 4][Avg]CELoss: 1.68777, loss: 1.68777, top1: 0.27600, top5: 1.00000
    [2021/08/15 15:54:08] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:54:08] root INFO: [Eval][Epoch 4][best metric: 0.27599999761581423]
    [2021/08/15 15:54:09] root INFO: Already save model in ./output/ResNet50/epoch_4
    [2021/08/15 15:54:10] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:54:11] root INFO: [Train][Epoch 5/20][Iter: 0/36]lr: 0.10000, CELoss: 1.63872, loss: 1.63872, top1: 0.19531, top5: 1.00000, batch_cost: 1.08729s, reader_cost: 0.94230, ips: 117.72366 images/sec, eta: 0:10:26
    [2021/08/15 15:54:12] root INFO: [Train][Epoch 5/20][Iter: 1/36]lr: 0.10000, CELoss: 1.68829, loss: 1.68829, top1: 0.19531, top5: 1.00000, batch_cost: 1.08205s, reader_cost: 0.93715, ips: 118.29385 images/sec, eta: 0:10:22
    [2021/08/15 15:54:12] root INFO: [Train][Epoch 5/20][Iter: 2/36]lr: 0.10000, CELoss: 1.67511, loss: 1.67511, top1: 0.21354, top5: 1.00000, batch_cost: 1.07689s, reader_cost: 0.93207, ips: 118.86124 images/sec, eta: 0:10:18
    [2021/08/15 15:54:13] root INFO: [Train][Epoch 5/20][Iter: 3/36]lr: 0.10000, CELoss: 1.65009, loss: 1.65009, top1: 0.23438, top5: 1.00000, batch_cost: 1.07193s, reader_cost: 0.92721, ips: 119.41079 images/sec, eta: 0:10:14
    [2021/08/15 15:54:14] root INFO: [Train][Epoch 5/20][Iter: 4/36]lr: 0.10000, CELoss: 1.66840, loss: 1.66840, top1: 0.22656, top5: 1.00000, batch_cost: 1.06689s, reader_cost: 0.92227, ips: 119.97449 images/sec, eta: 0:10:10
    [2021/08/15 15:54:15] root INFO: [Train][Epoch 5/20][Iter: 5/36]lr: 0.10000, CELoss: 1.65619, loss: 1.65619, top1: 0.22786, top5: 1.00000, batch_cost: 0.90347s, reader_cost: 0.76314, ips: 141.67534 images/sec, eta: 0:08:35
    [2021/08/15 15:54:16] root INFO: [Train][Epoch 5/20][Iter: 6/36]lr: 0.10000, CELoss: 1.65492, loss: 1.65492, top1: 0.22210, top5: 1.00000, batch_cost: 0.90381s, reader_cost: 0.76309, ips: 141.62309 images/sec, eta: 0:08:35
    [2021/08/15 15:54:17] root INFO: [Train][Epoch 5/20][Iter: 7/36]lr: 0.10000, CELoss: 1.66417, loss: 1.66417, top1: 0.22852, top5: 1.00000, batch_cost: 0.90242s, reader_cost: 0.76193, ips: 141.84066 images/sec, eta: 0:08:33
    [2021/08/15 15:54:18] root INFO: [Train][Epoch 5/20][Iter: 8/36]lr: 0.10000, CELoss: 1.65901, loss: 1.65901, top1: 0.22830, top5: 1.00000, batch_cost: 0.90201s, reader_cost: 0.76139, ips: 141.90592 images/sec, eta: 0:08:32
    [2021/08/15 15:54:19] root INFO: [Train][Epoch 5/20][Iter: 9/36]lr: 0.10000, CELoss: 1.65546, loss: 1.65546, top1: 0.22969, top5: 1.00000, batch_cost: 0.90061s, reader_cost: 0.75994, ips: 142.12619 images/sec, eta: 0:08:30
    [2021/08/15 15:54:20] root INFO: [Train][Epoch 5/20][Iter: 10/36]lr: 0.10000, CELoss: 1.64979, loss: 1.64979, top1: 0.23011, top5: 1.00000, batch_cost: 0.90649s, reader_cost: 0.76573, ips: 141.20405 images/sec, eta: 0:08:33
    [2021/08/15 15:54:21] root INFO: [Train][Epoch 5/20][Iter: 11/36]lr: 0.10000, CELoss: 1.64462, loss: 1.64462, top1: 0.23372, top5: 1.00000, batch_cost: 0.90578s, reader_cost: 0.76483, ips: 141.31488 images/sec, eta: 0:08:31
    [2021/08/15 15:54:22] root INFO: [Train][Epoch 5/20][Iter: 12/36]lr: 0.10000, CELoss: 1.64199, loss: 1.64199, top1: 0.23678, top5: 1.00000, batch_cost: 0.90915s, reader_cost: 0.76810, ips: 140.79124 images/sec, eta: 0:08:32
    [2021/08/15 15:54:22] root INFO: [Train][Epoch 5/20][Iter: 13/36]lr: 0.10000, CELoss: 1.63839, loss: 1.63839, top1: 0.23717, top5: 1.00000, batch_cost: 0.90845s, reader_cost: 0.76731, ips: 140.89989 images/sec, eta: 0:08:31
    [2021/08/15 15:54:23] root INFO: [Train][Epoch 5/20][Iter: 14/36]lr: 0.10000, CELoss: 1.63844, loss: 1.63844, top1: 0.23802, top5: 1.00000, batch_cost: 0.90795s, reader_cost: 0.76681, ips: 140.97712 images/sec, eta: 0:08:30
    [2021/08/15 15:54:24] root INFO: [Train][Epoch 5/20][Iter: 15/36]lr: 0.10000, CELoss: 1.63587, loss: 1.63587, top1: 0.24072, top5: 1.00000, batch_cost: 0.90685s, reader_cost: 0.76565, ips: 141.14769 images/sec, eta: 0:08:28
    [2021/08/15 15:54:25] root INFO: [Train][Epoch 5/20][Iter: 16/36]lr: 0.10000, CELoss: 1.63270, loss: 1.63270, top1: 0.24540, top5: 1.00000, batch_cost: 0.90674s, reader_cost: 0.76549, ips: 141.16499 images/sec, eta: 0:08:27
    [2021/08/15 15:54:26] root INFO: [Train][Epoch 5/20][Iter: 17/36]lr: 0.10000, CELoss: 1.64597, loss: 1.64597, top1: 0.24609, top5: 1.00000, batch_cost: 0.90623s, reader_cost: 0.76487, ips: 141.24392 images/sec, eta: 0:08:26
    [2021/08/15 15:54:27] root INFO: [Train][Epoch 5/20][Iter: 18/36]lr: 0.10000, CELoss: 1.64463, loss: 1.64463, top1: 0.24507, top5: 1.00000, batch_cost: 0.90608s, reader_cost: 0.76462, ips: 141.26739 images/sec, eta: 0:08:25
    [2021/08/15 15:54:28] root INFO: [Train][Epoch 5/20][Iter: 19/36]lr: 0.10000, CELoss: 1.64077, loss: 1.64077, top1: 0.24805, top5: 1.00000, batch_cost: 0.90614s, reader_cost: 0.76470, ips: 141.25846 images/sec, eta: 0:08:24
    [2021/08/15 15:54:29] root INFO: [Train][Epoch 5/20][Iter: 20/36]lr: 0.10000, CELoss: 1.64343, loss: 1.64343, top1: 0.24963, top5: 1.00000, batch_cost: 0.90647s, reader_cost: 0.76506, ips: 141.20738 images/sec, eta: 0:08:23
    [2021/08/15 15:54:30] root INFO: [Train][Epoch 5/20][Iter: 21/36]lr: 0.10000, CELoss: 1.64280, loss: 1.64280, top1: 0.25036, top5: 1.00000, batch_cost: 0.90634s, reader_cost: 0.76494, ips: 141.22761 images/sec, eta: 0:08:23
    [2021/08/15 15:54:31] root INFO: [Train][Epoch 5/20][Iter: 22/36]lr: 0.10000, CELoss: 1.64070, loss: 1.64070, top1: 0.24728, top5: 1.00000, batch_cost: 0.90667s, reader_cost: 0.76507, ips: 141.17627 images/sec, eta: 0:08:22
    [2021/08/15 15:54:31] root INFO: [Train][Epoch 5/20][Iter: 23/36]lr: 0.10000, CELoss: 1.63876, loss: 1.63876, top1: 0.24837, top5: 1.00000, batch_cost: 0.90598s, reader_cost: 0.76442, ips: 141.28301 images/sec, eta: 0:08:21
    [2021/08/15 15:54:32] root INFO: [Train][Epoch 5/20][Iter: 24/36]lr: 0.10000, CELoss: 1.63574, loss: 1.63574, top1: 0.24688, top5: 1.00000, batch_cost: 0.90613s, reader_cost: 0.76449, ips: 141.25968 images/sec, eta: 0:08:20
    [2021/08/15 15:54:33] root INFO: [Train][Epoch 5/20][Iter: 25/36]lr: 0.10000, CELoss: 1.63786, loss: 1.63786, top1: 0.24669, top5: 1.00000, batch_cost: 0.90594s, reader_cost: 0.76426, ips: 141.28974 images/sec, eta: 0:08:19
    [2021/08/15 15:54:34] root INFO: [Train][Epoch 5/20][Iter: 26/36]lr: 0.10000, CELoss: 1.63776, loss: 1.63776, top1: 0.24711, top5: 1.00000, batch_cost: 0.90564s, reader_cost: 0.76395, ips: 141.33656 images/sec, eta: 0:08:18
    [2021/08/15 15:54:35] root INFO: [Train][Epoch 5/20][Iter: 27/36]lr: 0.10000, CELoss: 1.63518, loss: 1.63518, top1: 0.24721, top5: 1.00000, batch_cost: 0.90502s, reader_cost: 0.76338, ips: 141.43361 images/sec, eta: 0:08:16
    [2021/08/15 15:54:36] root INFO: [Train][Epoch 5/20][Iter: 28/36]lr: 0.10000, CELoss: 1.63252, loss: 1.63252, top1: 0.24838, top5: 1.00000, batch_cost: 0.90432s, reader_cost: 0.76264, ips: 141.54293 images/sec, eta: 0:08:15
    [2021/08/15 15:54:37] root INFO: [Train][Epoch 5/20][Iter: 29/36]lr: 0.10000, CELoss: 1.62957, loss: 1.62957, top1: 0.24870, top5: 1.00000, batch_cost: 0.90398s, reader_cost: 0.76230, ips: 141.59631 images/sec, eta: 0:08:14
    [2021/08/15 15:54:38] root INFO: [Train][Epoch 5/20][Iter: 30/36]lr: 0.10000, CELoss: 1.62814, loss: 1.62814, top1: 0.24924, top5: 1.00000, batch_cost: 0.90352s, reader_cost: 0.76182, ips: 141.66759 images/sec, eta: 0:08:13
    [2021/08/15 15:54:39] root INFO: [Train][Epoch 5/20][Iter: 31/36]lr: 0.10000, CELoss: 1.62694, loss: 1.62694, top1: 0.24927, top5: 1.00000, batch_cost: 0.90367s, reader_cost: 0.76199, ips: 141.64424 images/sec, eta: 0:08:12
    [2021/08/15 15:54:40] root INFO: [Train][Epoch 5/20][Iter: 32/36]lr: 0.10000, CELoss: 1.62568, loss: 1.62568, top1: 0.25095, top5: 1.00000, batch_cost: 0.90344s, reader_cost: 0.76174, ips: 141.68139 images/sec, eta: 0:08:11
    [2021/08/15 15:54:41] root INFO: [Train][Epoch 5/20][Iter: 33/36]lr: 0.10000, CELoss: 1.62269, loss: 1.62269, top1: 0.25253, top5: 1.00000, batch_cost: 0.90414s, reader_cost: 0.76234, ips: 141.57092 images/sec, eta: 0:08:10
    [2021/08/15 15:54:41] root INFO: [Train][Epoch 5/20][Iter: 34/36]lr: 0.10000, CELoss: 1.62365, loss: 1.62365, top1: 0.25268, top5: 1.00000, batch_cost: 0.90440s, reader_cost: 0.76251, ips: 141.52985 images/sec, eta: 0:08:10
    [2021/08/15 15:54:42] root INFO: [Train][Epoch 5/20][Iter: 35/36]lr: 0.10000, CELoss: 1.62476, loss: 1.62476, top1: 0.25222, top5: 1.00000, batch_cost: 0.88433s, reader_cost: 0.73952, ips: 22.61608 images/sec, eta: 0:07:58
    [2021/08/15 15:54:42] root INFO: [Train][Epoch 5/20][Avg]CELoss: 1.62476, loss: 1.62476, top1: 0.25222, top5: 1.00000
    [2021/08/15 15:54:43] root INFO: [Eval][Epoch 5][Iter: 0/4]CELoss: 1.79548, loss: 1.79548, top1: 0.23438, top5: 1.00000, batch_cost: 0.94585s, reader_cost: 0.83054, ips: 135.32752 images/sec
    [2021/08/15 15:54:44] root INFO: [Eval][Epoch 5][Iter: 1/4]CELoss: 1.59452, loss: 1.59452, top1: 0.30469, top5: 1.00000, batch_cost: 0.91559s, reader_cost: 0.80049, ips: 139.80085 images/sec
    [2021/08/15 15:54:44] root INFO: [Eval][Epoch 5][Iter: 2/4]CELoss: 1.56140, loss: 1.56140, top1: 0.25000, top5: 1.00000, batch_cost: 0.90832s, reader_cost: 0.79333, ips: 140.92007 images/sec
    [2021/08/15 15:54:45] root INFO: [Eval][Epoch 5][Iter: 3/4]CELoss: 1.55800, loss: 1.55800, top1: 0.27586, top5: 1.00000, batch_cost: 0.88277s, reader_cost: 0.76990, ips: 131.40424 images/sec
    [2021/08/15 15:54:45] root INFO: [Eval][Epoch 5][Avg]CELoss: 1.62901, loss: 1.62901, top1: 0.26600, top5: 1.00000
    [2021/08/15 15:54:45] root INFO: [Eval][Epoch 5][best metric: 0.27599999761581423]
    [2021/08/15 15:54:46] root INFO: Already save model in ./output/ResNet50/epoch_5
    [2021/08/15 15:54:46] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:54:47] root INFO: [Train][Epoch 6/20][Iter: 0/36]lr: 0.10000, CELoss: 1.56049, loss: 1.56049, top1: 0.32031, top5: 1.00000, batch_cost: 1.03511s, reader_cost: 0.89042, ips: 123.65791 images/sec, eta: 0:09:18
    [2021/08/15 15:54:48] root INFO: [Train][Epoch 6/20][Iter: 1/36]lr: 0.10000, CELoss: 1.58249, loss: 1.58249, top1: 0.24609, top5: 1.00000, batch_cost: 1.03108s, reader_cost: 0.88648, ips: 124.14137 images/sec, eta: 0:09:15
    [2021/08/15 15:54:49] root INFO: [Train][Epoch 6/20][Iter: 2/36]lr: 0.10000, CELoss: 1.64230, loss: 1.64230, top1: 0.24479, top5: 1.00000, batch_cost: 1.02733s, reader_cost: 0.88280, ips: 124.59502 images/sec, eta: 0:09:12
    [2021/08/15 15:54:50] root INFO: [Train][Epoch 6/20][Iter: 3/36]lr: 0.10000, CELoss: 1.64106, loss: 1.64106, top1: 0.24023, top5: 1.00000, batch_cost: 1.02394s, reader_cost: 0.87947, ips: 125.00736 images/sec, eta: 0:09:09
    [2021/08/15 15:54:51] root INFO: [Train][Epoch 6/20][Iter: 4/36]lr: 0.10000, CELoss: 1.64015, loss: 1.64015, top1: 0.23750, top5: 1.00000, batch_cost: 1.02068s, reader_cost: 0.87629, ips: 125.40647 images/sec, eta: 0:09:07
    [2021/08/15 15:54:52] root INFO: [Train][Epoch 6/20][Iter: 5/36]lr: 0.10000, CELoss: 1.62171, loss: 1.62171, top1: 0.25130, top5: 1.00000, batch_cost: 0.90314s, reader_cost: 0.76155, ips: 141.72785 images/sec, eta: 0:08:03
    [2021/08/15 15:54:53] root INFO: [Train][Epoch 6/20][Iter: 6/36]lr: 0.10000, CELoss: 1.61410, loss: 1.61410, top1: 0.25893, top5: 1.00000, batch_cost: 0.90878s, reader_cost: 0.76733, ips: 140.84843 images/sec, eta: 0:08:05
    [2021/08/15 15:54:54] root INFO: [Train][Epoch 6/20][Iter: 7/36]lr: 0.10000, CELoss: 1.61755, loss: 1.61755, top1: 0.26660, top5: 1.00000, batch_cost: 0.91367s, reader_cost: 0.77255, ips: 140.09496 images/sec, eta: 0:08:06
    [2021/08/15 15:54:55] root INFO: [Train][Epoch 6/20][Iter: 8/36]lr: 0.10000, CELoss: 1.61526, loss: 1.61526, top1: 0.25781, top5: 1.00000, batch_cost: 0.90917s, reader_cost: 0.76821, ips: 140.78845 images/sec, eta: 0:08:03
    [2021/08/15 15:54:56] root INFO: [Train][Epoch 6/20][Iter: 9/36]lr: 0.10000, CELoss: 1.61927, loss: 1.61927, top1: 0.25547, top5: 1.00000, batch_cost: 0.90572s, reader_cost: 0.76490, ips: 141.32450 images/sec, eta: 0:08:00
    [2021/08/15 15:54:56] root INFO: [Train][Epoch 6/20][Iter: 10/36]lr: 0.10000, CELoss: 1.61779, loss: 1.61779, top1: 0.25994, top5: 1.00000, batch_cost: 0.90564s, reader_cost: 0.76490, ips: 141.33696 images/sec, eta: 0:07:59
    [2021/08/15 15:54:57] root INFO: [Train][Epoch 6/20][Iter: 11/36]lr: 0.10000, CELoss: 1.61430, loss: 1.61430, top1: 0.26237, top5: 1.00000, batch_cost: 0.90438s, reader_cost: 0.76359, ips: 141.53348 images/sec, eta: 0:07:58
    [2021/08/15 15:54:58] root INFO: [Train][Epoch 6/20][Iter: 12/36]lr: 0.10000, CELoss: 1.60749, loss: 1.60749, top1: 0.26803, top5: 1.00000, batch_cost: 0.90330s, reader_cost: 0.76250, ips: 141.70233 images/sec, eta: 0:07:56
    [2021/08/15 15:54:59] root INFO: [Train][Epoch 6/20][Iter: 13/36]lr: 0.10000, CELoss: 1.60848, loss: 1.60848, top1: 0.26618, top5: 1.00000, batch_cost: 0.90273s, reader_cost: 0.76192, ips: 141.79139 images/sec, eta: 0:07:55
    [2021/08/15 15:55:00] root INFO: [Train][Epoch 6/20][Iter: 14/36]lr: 0.10000, CELoss: 1.60482, loss: 1.60482, top1: 0.26615, top5: 1.00000, batch_cost: 0.90236s, reader_cost: 0.76147, ips: 141.85102 images/sec, eta: 0:07:54
    [2021/08/15 15:55:01] root INFO: [Train][Epoch 6/20][Iter: 15/36]lr: 0.10000, CELoss: 1.60261, loss: 1.60261, top1: 0.26416, top5: 1.00000, batch_cost: 0.90302s, reader_cost: 0.76203, ips: 141.74639 images/sec, eta: 0:07:54
    [2021/08/15 15:55:02] root INFO: [Train][Epoch 6/20][Iter: 16/36]lr: 0.10000, CELoss: 1.59902, loss: 1.59902, top1: 0.26700, top5: 1.00000, batch_cost: 0.90333s, reader_cost: 0.76212, ips: 141.69724 images/sec, eta: 0:07:53
    [2021/08/15 15:55:03] root INFO: [Train][Epoch 6/20][Iter: 17/36]lr: 0.10000, CELoss: 1.59661, loss: 1.59661, top1: 0.26997, top5: 1.00000, batch_cost: 0.90405s, reader_cost: 0.76270, ips: 141.58486 images/sec, eta: 0:07:52
    [2021/08/15 15:55:04] root INFO: [Train][Epoch 6/20][Iter: 18/36]lr: 0.10000, CELoss: 1.59476, loss: 1.59476, top1: 0.26974, top5: 1.00000, batch_cost: 0.90496s, reader_cost: 0.76365, ips: 141.44260 images/sec, eta: 0:07:52
    [2021/08/15 15:55:05] root INFO: [Train][Epoch 6/20][Iter: 19/36]lr: 0.10000, CELoss: 1.59485, loss: 1.59485, top1: 0.27227, top5: 1.00000, batch_cost: 0.90550s, reader_cost: 0.76423, ips: 141.35794 images/sec, eta: 0:07:51
    [2021/08/15 15:55:06] root INFO: [Train][Epoch 6/20][Iter: 20/36]lr: 0.10000, CELoss: 1.59630, loss: 1.59630, top1: 0.27307, top5: 1.00000, batch_cost: 0.90603s, reader_cost: 0.76481, ips: 141.27501 images/sec, eta: 0:07:51
    [2021/08/15 15:55:06] root INFO: [Train][Epoch 6/20][Iter: 21/36]lr: 0.10000, CELoss: 1.59307, loss: 1.59307, top1: 0.27450, top5: 1.00000, batch_cost: 0.90616s, reader_cost: 0.76497, ips: 141.25609 images/sec, eta: 0:07:50
    [2021/08/15 15:55:07] root INFO: [Train][Epoch 6/20][Iter: 22/36]lr: 0.10000, CELoss: 1.59105, loss: 1.59105, top1: 0.27514, top5: 1.00000, batch_cost: 0.90593s, reader_cost: 0.76473, ips: 141.29129 images/sec, eta: 0:07:49
    [2021/08/15 15:55:08] root INFO: [Train][Epoch 6/20][Iter: 23/36]lr: 0.10000, CELoss: 1.59050, loss: 1.59050, top1: 0.27539, top5: 1.00000, batch_cost: 0.90578s, reader_cost: 0.76451, ips: 141.31410 images/sec, eta: 0:07:48
    [2021/08/15 15:55:09] root INFO: [Train][Epoch 6/20][Iter: 24/36]lr: 0.10000, CELoss: 1.58618, loss: 1.58618, top1: 0.28125, top5: 1.00000, batch_cost: 0.90551s, reader_cost: 0.76424, ips: 141.35755 images/sec, eta: 0:07:47
    [2021/08/15 15:55:10] root INFO: [Train][Epoch 6/20][Iter: 25/36]lr: 0.10000, CELoss: 1.58569, loss: 1.58569, top1: 0.28395, top5: 1.00000, batch_cost: 0.90538s, reader_cost: 0.76405, ips: 141.37639 images/sec, eta: 0:07:46
    [2021/08/15 15:55:11] root INFO: [Train][Epoch 6/20][Iter: 26/36]lr: 0.10000, CELoss: 1.58378, loss: 1.58378, top1: 0.28443, top5: 1.00000, batch_cost: 0.90605s, reader_cost: 0.76468, ips: 141.27228 images/sec, eta: 0:07:45
    [2021/08/15 15:55:12] root INFO: [Train][Epoch 6/20][Iter: 27/36]lr: 0.10000, CELoss: 1.58470, loss: 1.58470, top1: 0.28348, top5: 1.00000, batch_cost: 0.90637s, reader_cost: 0.76499, ips: 141.22326 images/sec, eta: 0:07:44
    [2021/08/15 15:55:13] root INFO: [Train][Epoch 6/20][Iter: 28/36]lr: 0.10000, CELoss: 1.58484, loss: 1.58484, top1: 0.28367, top5: 1.00000, batch_cost: 0.90599s, reader_cost: 0.76460, ips: 141.28258 images/sec, eta: 0:07:43
    [2021/08/15 15:55:14] root INFO: [Train][Epoch 6/20][Iter: 29/36]lr: 0.10000, CELoss: 1.58233, loss: 1.58233, top1: 0.28542, top5: 1.00000, batch_cost: 0.90611s, reader_cost: 0.76472, ips: 141.26279 images/sec, eta: 0:07:43
    [2021/08/15 15:55:15] root INFO: [Train][Epoch 6/20][Iter: 30/36]lr: 0.10000, CELoss: 1.58094, loss: 1.58094, top1: 0.28478, top5: 1.00000, batch_cost: 0.90623s, reader_cost: 0.76482, ips: 141.24467 images/sec, eta: 0:07:42
    [2021/08/15 15:55:16] root INFO: [Train][Epoch 6/20][Iter: 31/36]lr: 0.10000, CELoss: 1.58169, loss: 1.58169, top1: 0.28491, top5: 1.00000, batch_cost: 0.90596s, reader_cost: 0.76453, ips: 141.28713 images/sec, eta: 0:07:41
    [2021/08/15 15:55:16] root INFO: [Train][Epoch 6/20][Iter: 32/36]lr: 0.10000, CELoss: 1.58138, loss: 1.58138, top1: 0.28598, top5: 1.00000, batch_cost: 0.90566s, reader_cost: 0.76427, ips: 141.33405 images/sec, eta: 0:07:40
    [2021/08/15 15:55:17] root INFO: [Train][Epoch 6/20][Iter: 33/36]lr: 0.10000, CELoss: 1.58081, loss: 1.58081, top1: 0.28470, top5: 1.00000, batch_cost: 0.90554s, reader_cost: 0.76416, ips: 141.35271 images/sec, eta: 0:07:39
    [2021/08/15 15:55:18] root INFO: [Train][Epoch 6/20][Iter: 34/36]lr: 0.10000, CELoss: 1.58017, loss: 1.58017, top1: 0.28326, top5: 1.00000, batch_cost: 0.90536s, reader_cost: 0.76402, ips: 141.38004 images/sec, eta: 0:07:38
    [2021/08/15 15:55:18] root INFO: [Train][Epoch 6/20][Iter: 35/36]lr: 0.10000, CELoss: 1.57965, loss: 1.57965, top1: 0.28356, top5: 1.00000, batch_cost: 0.88523s, reader_cost: 0.74112, ips: 22.59296 images/sec, eta: 0:07:27
    [2021/08/15 15:55:18] root INFO: [Train][Epoch 6/20][Avg]CELoss: 1.57965, loss: 1.57965, top1: 0.28356, top5: 1.00000
    [2021/08/15 15:55:19] root INFO: [Eval][Epoch 6][Iter: 0/4]CELoss: 1.53968, loss: 1.53968, top1: 0.31250, top5: 1.00000, batch_cost: 0.93869s, reader_cost: 0.82349, ips: 136.36046 images/sec
    [2021/08/15 15:55:20] root INFO: [Eval][Epoch 6][Iter: 1/4]CELoss: 1.59908, loss: 1.59908, top1: 0.22656, top5: 1.00000, batch_cost: 0.90822s, reader_cost: 0.79290, ips: 140.93536 images/sec
    [2021/08/15 15:55:21] root INFO: [Eval][Epoch 6][Iter: 2/4]CELoss: 1.50905, loss: 1.50905, top1: 0.28906, top5: 1.00000, batch_cost: 0.90010s, reader_cost: 0.78484, ips: 142.20579 images/sec
    [2021/08/15 15:55:22] root INFO: [Eval][Epoch 6][Iter: 3/4]CELoss: 1.43905, loss: 1.43905, top1: 0.38793, top5: 1.00000, batch_cost: 0.87846s, reader_cost: 0.76515, ips: 132.04877 images/sec
    [2021/08/15 15:55:22] root INFO: [Eval][Epoch 6][Avg]CELoss: 1.52370, loss: 1.52370, top1: 0.30200, top5: 1.00000
    [2021/08/15 15:55:23] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:55:23] root INFO: [Eval][Epoch 6][best metric: 0.3020000033378601]
    [2021/08/15 15:55:24] root INFO: Already save model in ./output/ResNet50/epoch_6
    [2021/08/15 15:55:24] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:55:25] root INFO: [Train][Epoch 7/20][Iter: 0/36]lr: 0.10000, CELoss: 1.59387, loss: 1.59387, top1: 0.35938, top5: 1.00000, batch_cost: 1.06856s, reader_cost: 0.92426, ips: 119.78753 images/sec, eta: 0:08:58
    [2021/08/15 15:55:26] root INFO: [Train][Epoch 7/20][Iter: 1/36]lr: 0.10000, CELoss: 1.63195, loss: 1.63195, top1: 0.31641, top5: 1.00000, batch_cost: 1.06363s, reader_cost: 0.91942, ips: 120.34278 images/sec, eta: 0:08:55
    [2021/08/15 15:55:27] root INFO: [Train][Epoch 7/20][Iter: 2/36]lr: 0.10000, CELoss: 1.61211, loss: 1.61211, top1: 0.29167, top5: 1.00000, batch_cost: 1.05904s, reader_cost: 0.91495, ips: 120.86376 images/sec, eta: 0:08:51
    [2021/08/15 15:55:28] root INFO: [Train][Epoch 7/20][Iter: 3/36]lr: 0.10000, CELoss: 1.56845, loss: 1.56845, top1: 0.30469, top5: 1.00000, batch_cost: 1.05449s, reader_cost: 0.91047, ips: 121.38534 images/sec, eta: 0:08:48
    [2021/08/15 15:55:29] root INFO: [Train][Epoch 7/20][Iter: 4/36]lr: 0.10000, CELoss: 1.52463, loss: 1.52463, top1: 0.32969, top5: 1.00000, batch_cost: 1.05014s, reader_cost: 0.90619, ips: 121.88899 images/sec, eta: 0:08:45
    [2021/08/15 15:55:30] root INFO: [Train][Epoch 7/20][Iter: 5/36]lr: 0.10000, CELoss: 1.52442, loss: 1.52442, top1: 0.32552, top5: 1.00000, batch_cost: 0.90892s, reader_cost: 0.76652, ips: 140.82580 images/sec, eta: 0:07:33
    [2021/08/15 15:55:31] root INFO: [Train][Epoch 7/20][Iter: 6/36]lr: 0.10000, CELoss: 1.52827, loss: 1.52827, top1: 0.31473, top5: 1.00000, batch_cost: 0.90815s, reader_cost: 0.76571, ips: 140.94648 images/sec, eta: 0:07:32
    [2021/08/15 15:55:32] root INFO: [Train][Epoch 7/20][Iter: 7/36]lr: 0.10000, CELoss: 1.52278, loss: 1.52278, top1: 0.31445, top5: 1.00000, batch_cost: 0.90516s, reader_cost: 0.76244, ips: 141.41213 images/sec, eta: 0:07:29
    [2021/08/15 15:55:32] root INFO: [Train][Epoch 7/20][Iter: 8/36]lr: 0.10000, CELoss: 1.55081, loss: 1.55081, top1: 0.30816, top5: 1.00000, batch_cost: 0.90335s, reader_cost: 0.76084, ips: 141.69521 images/sec, eta: 0:07:28
    [2021/08/15 15:55:33] root INFO: [Train][Epoch 7/20][Iter: 9/36]lr: 0.10000, CELoss: 1.56604, loss: 1.56604, top1: 0.30703, top5: 1.00000, batch_cost: 0.90417s, reader_cost: 0.76192, ips: 141.56568 images/sec, eta: 0:07:27
    [2021/08/15 15:55:34] root INFO: [Train][Epoch 7/20][Iter: 10/36]lr: 0.10000, CELoss: 1.56647, loss: 1.56647, top1: 0.30185, top5: 1.00000, batch_cost: 0.90409s, reader_cost: 0.76194, ips: 141.57809 images/sec, eta: 0:07:26
    [2021/08/15 15:55:35] root INFO: [Train][Epoch 7/20][Iter: 11/36]lr: 0.10000, CELoss: 1.55453, loss: 1.55453, top1: 0.31315, top5: 1.00000, batch_cost: 0.90584s, reader_cost: 0.76383, ips: 141.30570 images/sec, eta: 0:07:26
    [2021/08/15 15:55:36] root INFO: [Train][Epoch 7/20][Iter: 12/36]lr: 0.10000, CELoss: 1.54916, loss: 1.54916, top1: 0.31971, top5: 1.00000, batch_cost: 0.90695s, reader_cost: 0.76507, ips: 141.13205 images/sec, eta: 0:07:26
    [2021/08/15 15:55:37] root INFO: [Train][Epoch 7/20][Iter: 13/36]lr: 0.10000, CELoss: 1.54937, loss: 1.54937, top1: 0.32031, top5: 1.00000, batch_cost: 0.90768s, reader_cost: 0.76497, ips: 141.01924 images/sec, eta: 0:07:25
    [2021/08/15 15:55:38] root INFO: [Train][Epoch 7/20][Iter: 14/36]lr: 0.10000, CELoss: 1.54798, loss: 1.54798, top1: 0.32135, top5: 1.00000, batch_cost: 0.90711s, reader_cost: 0.76460, ips: 141.10687 images/sec, eta: 0:07:24
    [2021/08/15 15:55:39] root INFO: [Train][Epoch 7/20][Iter: 15/36]lr: 0.10000, CELoss: 1.54057, loss: 1.54057, top1: 0.32666, top5: 1.00000, batch_cost: 0.90587s, reader_cost: 0.76351, ips: 141.30066 images/sec, eta: 0:07:22
    [2021/08/15 15:55:40] root INFO: [Train][Epoch 7/20][Iter: 16/36]lr: 0.10000, CELoss: 1.53825, loss: 1.53825, top1: 0.33180, top5: 1.00000, batch_cost: 0.90525s, reader_cost: 0.76296, ips: 141.39800 images/sec, eta: 0:07:21
    [2021/08/15 15:55:41] root INFO: [Train][Epoch 7/20][Iter: 17/36]lr: 0.10000, CELoss: 1.53356, loss: 1.53356, top1: 0.33247, top5: 1.00000, batch_cost: 0.90642s, reader_cost: 0.76415, ips: 141.21563 images/sec, eta: 0:07:21
    [2021/08/15 15:55:42] root INFO: [Train][Epoch 7/20][Iter: 18/36]lr: 0.10000, CELoss: 1.53049, loss: 1.53049, top1: 0.33059, top5: 1.00000, batch_cost: 0.90590s, reader_cost: 0.76364, ips: 141.29641 images/sec, eta: 0:07:20
    [2021/08/15 15:55:42] root INFO: [Train][Epoch 7/20][Iter: 19/36]lr: 0.10000, CELoss: 1.52536, loss: 1.52536, top1: 0.33320, top5: 1.00000, batch_cost: 0.90582s, reader_cost: 0.76352, ips: 141.30829 images/sec, eta: 0:07:19
    [2021/08/15 15:55:43] root INFO: [Train][Epoch 7/20][Iter: 20/36]lr: 0.10000, CELoss: 1.52432, loss: 1.52432, top1: 0.33557, top5: 1.00000, batch_cost: 0.90498s, reader_cost: 0.76264, ips: 141.43943 images/sec, eta: 0:07:18
    [2021/08/15 15:55:44] root INFO: [Train][Epoch 7/20][Iter: 21/36]lr: 0.10000, CELoss: 1.52430, loss: 1.52430, top1: 0.33736, top5: 1.00000, batch_cost: 0.90524s, reader_cost: 0.76290, ips: 141.39965 images/sec, eta: 0:07:17
    [2021/08/15 15:55:45] root INFO: [Train][Epoch 7/20][Iter: 22/36]lr: 0.10000, CELoss: 1.51899, loss: 1.51899, top1: 0.34137, top5: 1.00000, batch_cost: 0.90507s, reader_cost: 0.76270, ips: 141.42534 images/sec, eta: 0:07:16
    [2021/08/15 15:55:46] root INFO: [Train][Epoch 7/20][Iter: 23/36]lr: 0.10000, CELoss: 1.52017, loss: 1.52017, top1: 0.33984, top5: 1.00000, batch_cost: 0.90492s, reader_cost: 0.76257, ips: 141.44872 images/sec, eta: 0:07:15
    [2021/08/15 15:55:47] root INFO: [Train][Epoch 7/20][Iter: 24/36]lr: 0.10000, CELoss: 1.51912, loss: 1.51912, top1: 0.33875, top5: 1.00000, batch_cost: 0.90507s, reader_cost: 0.76278, ips: 141.42593 images/sec, eta: 0:07:14
    [2021/08/15 15:55:48] root INFO: [Train][Epoch 7/20][Iter: 25/36]lr: 0.10000, CELoss: 1.51604, loss: 1.51604, top1: 0.34014, top5: 1.00000, batch_cost: 0.90500s, reader_cost: 0.76276, ips: 141.43660 images/sec, eta: 0:07:13
    [2021/08/15 15:55:49] root INFO: [Train][Epoch 7/20][Iter: 26/36]lr: 0.10000, CELoss: 1.51292, loss: 1.51292, top1: 0.33767, top5: 1.00000, batch_cost: 0.90481s, reader_cost: 0.76256, ips: 141.46636 images/sec, eta: 0:07:12
    [2021/08/15 15:55:50] root INFO: [Train][Epoch 7/20][Iter: 27/36]lr: 0.10000, CELoss: 1.51033, loss: 1.51033, top1: 0.34235, top5: 1.00000, batch_cost: 0.90492s, reader_cost: 0.76273, ips: 141.44877 images/sec, eta: 0:07:11
    [2021/08/15 15:55:51] root INFO: [Train][Epoch 7/20][Iter: 28/36]lr: 0.10000, CELoss: 1.50942, loss: 1.50942, top1: 0.34267, top5: 1.00000, batch_cost: 0.90476s, reader_cost: 0.76259, ips: 141.47340 images/sec, eta: 0:07:10
    [2021/08/15 15:55:51] root INFO: [Train][Epoch 7/20][Iter: 29/36]lr: 0.10000, CELoss: 1.50642, loss: 1.50642, top1: 0.34557, top5: 1.00000, batch_cost: 0.90490s, reader_cost: 0.76276, ips: 141.45233 images/sec, eta: 0:07:09
    [2021/08/15 15:55:52] root INFO: [Train][Epoch 7/20][Iter: 30/36]lr: 0.10000, CELoss: 1.50852, loss: 1.50852, top1: 0.34551, top5: 1.00000, batch_cost: 0.90495s, reader_cost: 0.76281, ips: 141.44490 images/sec, eta: 0:07:08
    [2021/08/15 15:55:53] root INFO: [Train][Epoch 7/20][Iter: 31/36]lr: 0.10000, CELoss: 1.50615, loss: 1.50615, top1: 0.34570, top5: 1.00000, batch_cost: 0.90488s, reader_cost: 0.76273, ips: 141.45570 images/sec, eta: 0:07:08
    [2021/08/15 15:55:54] root INFO: [Train][Epoch 7/20][Iter: 32/36]lr: 0.10000, CELoss: 1.50529, loss: 1.50529, top1: 0.34683, top5: 1.00000, batch_cost: 0.90473s, reader_cost: 0.76263, ips: 141.47806 images/sec, eta: 0:07:07
    [2021/08/15 15:55:55] root INFO: [Train][Epoch 7/20][Iter: 33/36]lr: 0.10000, CELoss: 1.57651, loss: 1.57651, top1: 0.34513, top5: 1.00000, batch_cost: 0.90499s, reader_cost: 0.76293, ips: 141.43756 images/sec, eta: 0:07:06
    [2021/08/15 15:55:56] root INFO: [Train][Epoch 7/20][Iter: 34/36]lr: 0.10000, CELoss: 1.57700, loss: 1.57700, top1: 0.34063, top5: 1.00000, batch_cost: 0.90495s, reader_cost: 0.76286, ips: 141.44477 images/sec, eta: 0:07:05
    [2021/08/15 15:55:56] root INFO: [Train][Epoch 7/20][Iter: 35/36]lr: 0.10000, CELoss: 1.57727, loss: 1.57727, top1: 0.34022, top5: 1.00000, batch_cost: 0.88476s, reader_cost: 0.73994, ips: 22.60493 images/sec, eta: 0:06:54
    [2021/08/15 15:55:56] root INFO: [Train][Epoch 7/20][Avg]CELoss: 1.57727, loss: 1.57727, top1: 0.34022, top5: 1.00000
    [2021/08/15 15:55:57] root INFO: [Eval][Epoch 7][Iter: 0/4]CELoss: 5.14541, loss: 5.14541, top1: 0.22656, top5: 1.00000, batch_cost: 0.93154s, reader_cost: 0.81712, ips: 137.40673 images/sec
    [2021/08/15 15:55:58] root INFO: [Eval][Epoch 7][Iter: 1/4]CELoss: 9.83385, loss: 9.83385, top1: 0.28906, top5: 1.00000, batch_cost: 0.90829s, reader_cost: 0.79364, ips: 140.92400 images/sec
    [2021/08/15 15:55:59] root INFO: [Eval][Epoch 7][Iter: 2/4]CELoss: 2.61480, loss: 2.61480, top1: 0.24219, top5: 1.00000, batch_cost: 0.89829s, reader_cost: 0.78339, ips: 142.49271 images/sec
    [2021/08/15 15:56:00] root INFO: [Eval][Epoch 7][Iter: 3/4]CELoss: 10.18324, loss: 10.18324, top1: 0.23276, top5: 1.00000, batch_cost: 0.87446s, reader_cost: 0.76167, ips: 132.65338 images/sec
    [2021/08/15 15:56:00] root INFO: [Eval][Epoch 7][Avg]CELoss: 6.86659, loss: 6.86659, top1: 0.24800, top5: 1.00000
    [2021/08/15 15:56:00] root INFO: [Eval][Epoch 7][best metric: 0.3020000033378601]
    [2021/08/15 15:56:00] root INFO: Already save model in ./output/ResNet50/epoch_7
    [2021/08/15 15:56:01] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:56:02] root INFO: [Train][Epoch 8/20][Iter: 0/36]lr: 0.10000, CELoss: 1.70258, loss: 1.70258, top1: 0.17969, top5: 1.00000, batch_cost: 1.03384s, reader_cost: 0.88902, ips: 123.81008 images/sec, eta: 0:08:03
    [2021/08/15 15:56:03] root INFO: [Train][Epoch 8/20][Iter: 1/36]lr: 0.10000, CELoss: 1.70788, loss: 1.70788, top1: 0.19141, top5: 1.00000, batch_cost: 1.02983s, reader_cost: 0.88502, ips: 124.29291 images/sec, eta: 0:08:00
    [2021/08/15 15:56:04] root INFO: [Train][Epoch 8/20][Iter: 2/36]lr: 0.10000, CELoss: 1.67457, loss: 1.67457, top1: 0.21094, top5: 1.00000, batch_cost: 1.02580s, reader_cost: 0.88111, ips: 124.78062 images/sec, eta: 0:07:58
    [2021/08/15 15:56:05] root INFO: [Train][Epoch 8/20][Iter: 3/36]lr: 0.10000, CELoss: 1.66784, loss: 1.66784, top1: 0.21680, top5: 1.00000, batch_cost: 1.02254s, reader_cost: 0.87793, ips: 125.17845 images/sec, eta: 0:07:55
    [2021/08/15 15:56:06] root INFO: [Train][Epoch 8/20][Iter: 4/36]lr: 0.10000, CELoss: 1.66489, loss: 1.66489, top1: 0.21719, top5: 1.00000, batch_cost: 1.01915s, reader_cost: 0.87466, ips: 125.59493 images/sec, eta: 0:07:52
    [2021/08/15 15:56:06] root INFO: [Train][Epoch 8/20][Iter: 5/36]lr: 0.10000, CELoss: 1.70530, loss: 1.70530, top1: 0.21094, top5: 1.00000, batch_cost: 0.89135s, reader_cost: 0.74843, ips: 143.60309 images/sec, eta: 0:06:52
    [2021/08/15 15:56:07] root INFO: [Train][Epoch 8/20][Iter: 6/36]lr: 0.10000, CELoss: 1.69803, loss: 1.69803, top1: 0.21094, top5: 1.00000, batch_cost: 0.89094s, reader_cost: 0.74819, ips: 143.66903 images/sec, eta: 0:06:51
    [2021/08/15 15:56:08] root INFO: [Train][Epoch 8/20][Iter: 7/36]lr: 0.10000, CELoss: 1.69620, loss: 1.69620, top1: 0.20801, top5: 1.00000, batch_cost: 0.89155s, reader_cost: 0.74954, ips: 143.57037 images/sec, eta: 0:06:51
    [2021/08/15 15:56:09] root INFO: [Train][Epoch 8/20][Iter: 8/36]lr: 0.10000, CELoss: 1.70134, loss: 1.70134, top1: 0.21094, top5: 1.00000, batch_cost: 0.89488s, reader_cost: 0.75319, ips: 143.03641 images/sec, eta: 0:06:51
    [2021/08/15 15:56:10] root INFO: [Train][Epoch 8/20][Iter: 9/36]lr: 0.10000, CELoss: 1.71852, loss: 1.71852, top1: 0.20625, top5: 1.00000, batch_cost: 0.89675s, reader_cost: 0.75480, ips: 142.73702 images/sec, eta: 0:06:51
    [2021/08/15 15:56:11] root INFO: [Train][Epoch 8/20][Iter: 10/36]lr: 0.10000, CELoss: 1.72214, loss: 1.72214, top1: 0.20170, top5: 1.00000, batch_cost: 0.89879s, reader_cost: 0.75679, ips: 142.41395 images/sec, eta: 0:06:51
    [2021/08/15 15:56:12] root INFO: [Train][Epoch 8/20][Iter: 11/36]lr: 0.10000, CELoss: 1.71739, loss: 1.71739, top1: 0.20312, top5: 1.00000, batch_cost: 0.89950s, reader_cost: 0.75755, ips: 142.30084 images/sec, eta: 0:06:51
    [2021/08/15 15:56:13] root INFO: [Train][Epoch 8/20][Iter: 12/36]lr: 0.10000, CELoss: 1.71444, loss: 1.71444, top1: 0.20312, top5: 1.00000, batch_cost: 0.90006s, reader_cost: 0.75818, ips: 142.21273 images/sec, eta: 0:06:50
    [2021/08/15 15:56:14] root INFO: [Train][Epoch 8/20][Iter: 13/36]lr: 0.10000, CELoss: 1.71971, loss: 1.71971, top1: 0.20592, top5: 1.00000, batch_cost: 0.90542s, reader_cost: 0.76363, ips: 141.37150 images/sec, eta: 0:06:51
    [2021/08/15 15:56:15] root INFO: [Train][Epoch 8/20][Iter: 14/36]lr: 0.10000, CELoss: 1.71402, loss: 1.71402, top1: 0.20625, top5: 1.00000, batch_cost: 0.90511s, reader_cost: 0.76345, ips: 141.41904 images/sec, eta: 0:06:50
    [2021/08/15 15:56:16] root INFO: [Train][Epoch 8/20][Iter: 15/36]lr: 0.10000, CELoss: 1.72122, loss: 1.72122, top1: 0.20459, top5: 1.00000, batch_cost: 0.90484s, reader_cost: 0.76332, ips: 141.46193 images/sec, eta: 0:06:49
    [2021/08/15 15:56:16] root INFO: [Train][Epoch 8/20][Iter: 16/36]lr: 0.10000, CELoss: 1.71429, loss: 1.71429, top1: 0.20496, top5: 1.00000, batch_cost: 0.90412s, reader_cost: 0.76265, ips: 141.57365 images/sec, eta: 0:06:48
    [2021/08/15 15:56:17] root INFO: [Train][Epoch 8/20][Iter: 17/36]lr: 0.10000, CELoss: 1.70875, loss: 1.70875, top1: 0.20573, top5: 1.00000, batch_cost: 0.90343s, reader_cost: 0.76194, ips: 141.68194 images/sec, eta: 0:06:47
    [2021/08/15 15:56:18] root INFO: [Train][Epoch 8/20][Iter: 18/36]lr: 0.10000, CELoss: 1.70558, loss: 1.70558, top1: 0.20559, top5: 1.00000, batch_cost: 0.90332s, reader_cost: 0.76185, ips: 141.70022 images/sec, eta: 0:06:46
    [2021/08/15 15:56:19] root INFO: [Train][Epoch 8/20][Iter: 19/36]lr: 0.10000, CELoss: 1.70029, loss: 1.70029, top1: 0.20703, top5: 1.00000, batch_cost: 0.90282s, reader_cost: 0.76139, ips: 141.77757 images/sec, eta: 0:06:45
    [2021/08/15 15:56:20] root INFO: [Train][Epoch 8/20][Iter: 20/36]lr: 0.10000, CELoss: 1.70120, loss: 1.70120, top1: 0.20722, top5: 1.00000, batch_cost: 0.90254s, reader_cost: 0.76104, ips: 141.82237 images/sec, eta: 0:06:44
    [2021/08/15 15:56:21] root INFO: [Train][Epoch 8/20][Iter: 21/36]lr: 0.10000, CELoss: 1.69717, loss: 1.69717, top1: 0.20703, top5: 1.00000, batch_cost: 0.90182s, reader_cost: 0.76017, ips: 141.93446 images/sec, eta: 0:06:43
    [2021/08/15 15:56:22] root INFO: [Train][Epoch 8/20][Iter: 22/36]lr: 0.10000, CELoss: 1.69770, loss: 1.69770, top1: 0.20516, top5: 1.00000, batch_cost: 0.90165s, reader_cost: 0.76002, ips: 141.96249 images/sec, eta: 0:06:42
    [2021/08/15 15:56:23] root INFO: [Train][Epoch 8/20][Iter: 23/36]lr: 0.10000, CELoss: 1.69673, loss: 1.69673, top1: 0.20410, top5: 1.00000, batch_cost: 0.90099s, reader_cost: 0.75937, ips: 142.06586 images/sec, eta: 0:06:40
    [2021/08/15 15:56:24] root INFO: [Train][Epoch 8/20][Iter: 24/36]lr: 0.10000, CELoss: 1.70257, loss: 1.70257, top1: 0.20406, top5: 1.00000, batch_cost: 0.90033s, reader_cost: 0.75867, ips: 142.17006 images/sec, eta: 0:06:39
    [2021/08/15 15:56:24] root INFO: [Train][Epoch 8/20][Iter: 25/36]lr: 0.10000, CELoss: 1.69923, loss: 1.69923, top1: 0.20553, top5: 1.00000, batch_cost: 0.90004s, reader_cost: 0.75835, ips: 142.21602 images/sec, eta: 0:06:38
    [2021/08/15 15:56:25] root INFO: [Train][Epoch 8/20][Iter: 26/36]lr: 0.10000, CELoss: 1.69925, loss: 1.69925, top1: 0.20891, top5: 1.00000, batch_cost: 0.90000s, reader_cost: 0.75830, ips: 142.22221 images/sec, eta: 0:06:37
    [2021/08/15 15:56:26] root INFO: [Train][Epoch 8/20][Iter: 27/36]lr: 0.10000, CELoss: 1.69856, loss: 1.69856, top1: 0.20871, top5: 1.00000, batch_cost: 0.90023s, reader_cost: 0.75820, ips: 142.18616 images/sec, eta: 0:06:37
    [2021/08/15 15:56:27] root INFO: [Train][Epoch 8/20][Iter: 28/36]lr: 0.10000, CELoss: 1.69255, loss: 1.69255, top1: 0.21336, top5: 1.00000, batch_cost: 0.90022s, reader_cost: 0.75816, ips: 142.18764 images/sec, eta: 0:06:36
    [2021/08/15 15:56:28] root INFO: [Train][Epoch 8/20][Iter: 29/36]lr: 0.10000, CELoss: 1.68872, loss: 1.68872, top1: 0.21693, top5: 1.00000, batch_cost: 0.89993s, reader_cost: 0.75790, ips: 142.23336 images/sec, eta: 0:06:35
    [2021/08/15 15:56:29] root INFO: [Train][Epoch 8/20][Iter: 30/36]lr: 0.10000, CELoss: 1.68586, loss: 1.68586, top1: 0.22001, top5: 1.00000, batch_cost: 0.89985s, reader_cost: 0.75783, ips: 142.24616 images/sec, eta: 0:06:34
    [2021/08/15 15:56:30] root INFO: [Train][Epoch 8/20][Iter: 31/36]lr: 0.10000, CELoss: 1.68374, loss: 1.68374, top1: 0.22095, top5: 1.00000, batch_cost: 0.90030s, reader_cost: 0.75829, ips: 142.17558 images/sec, eta: 0:06:33
    [2021/08/15 15:56:31] root INFO: [Train][Epoch 8/20][Iter: 32/36]lr: 0.10000, CELoss: 1.68080, loss: 1.68080, top1: 0.22396, top5: 1.00000, batch_cost: 0.90108s, reader_cost: 0.75865, ips: 142.05146 images/sec, eta: 0:06:32
    [2021/08/15 15:56:32] root INFO: [Train][Epoch 8/20][Iter: 33/36]lr: 0.10000, CELoss: 1.67988, loss: 1.67988, top1: 0.22472, top5: 1.00000, batch_cost: 0.90358s, reader_cost: 0.76107, ips: 141.65849 images/sec, eta: 0:06:33
    [2021/08/15 15:56:33] root INFO: [Train][Epoch 8/20][Iter: 34/36]lr: 0.10000, CELoss: 1.68590, loss: 1.68590, top1: 0.22567, top5: 1.00000, batch_cost: 0.90380s, reader_cost: 0.76078, ips: 141.62395 images/sec, eta: 0:06:32
    [2021/08/15 15:56:33] root INFO: [Train][Epoch 8/20][Iter: 35/36]lr: 0.10000, CELoss: 1.68519, loss: 1.68519, top1: 0.22644, top5: 1.00000, batch_cost: 0.88321s, reader_cost: 0.73766, ips: 22.64473 images/sec, eta: 0:06:22
    [2021/08/15 15:56:33] root INFO: [Train][Epoch 8/20][Avg]CELoss: 1.68519, loss: 1.68519, top1: 0.22644, top5: 1.00000
    [2021/08/15 15:56:34] root INFO: [Eval][Epoch 8][Iter: 0/4]CELoss: 1.57986, loss: 1.57986, top1: 0.23438, top5: 1.00000, batch_cost: 0.93558s, reader_cost: 0.81967, ips: 136.81405 images/sec
    [2021/08/15 15:56:35] root INFO: [Eval][Epoch 8][Iter: 1/4]CELoss: 1.75059, loss: 1.75059, top1: 0.23438, top5: 1.00000, batch_cost: 0.90795s, reader_cost: 0.79260, ips: 140.97679 images/sec
    [2021/08/15 15:56:36] root INFO: [Eval][Epoch 8][Iter: 2/4]CELoss: 1.60878, loss: 1.60878, top1: 0.19531, top5: 1.00000, batch_cost: 0.90058s, reader_cost: 0.78540, ips: 142.13012 images/sec
    [2021/08/15 15:56:36] root INFO: [Eval][Epoch 8][Iter: 3/4]CELoss: 1.72979, loss: 1.72979, top1: 0.22414, top5: 1.00000, batch_cost: 0.87932s, reader_cost: 0.76628, ips: 131.92046 images/sec
    [2021/08/15 15:56:36] root INFO: [Eval][Epoch 8][Avg]CELoss: 1.66576, loss: 1.66576, top1: 0.22200, top5: 1.00000
    [2021/08/15 15:56:36] root INFO: [Eval][Epoch 8][best metric: 0.3020000033378601]
    [2021/08/15 15:56:37] root INFO: Already save model in ./output/ResNet50/epoch_8
    [2021/08/15 15:56:38] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:56:39] root INFO: [Train][Epoch 9/20][Iter: 0/36]lr: 0.10000, CELoss: 1.60680, loss: 1.60680, top1: 0.20312, top5: 1.00000, batch_cost: 1.03954s, reader_cost: 0.89413, ips: 123.13173 images/sec, eta: 0:07:29
    [2021/08/15 15:56:40] root INFO: [Train][Epoch 9/20][Iter: 1/36]lr: 0.10000, CELoss: 1.80725, loss: 1.80725, top1: 0.16016, top5: 1.00000, batch_cost: 1.03547s, reader_cost: 0.89013, ips: 123.61519 images/sec, eta: 0:07:26
    [2021/08/15 15:56:41] root INFO: [Train][Epoch 9/20][Iter: 2/36]lr: 0.10000, CELoss: 1.81097, loss: 1.81097, top1: 0.20312, top5: 1.00000, batch_cost: 1.03155s, reader_cost: 0.88628, ips: 124.08533 images/sec, eta: 0:07:23
    [2021/08/15 15:56:42] root INFO: [Train][Epoch 9/20][Iter: 3/36]lr: 0.10000, CELoss: 1.80525, loss: 1.80525, top1: 0.18555, top5: 1.00000, batch_cost: 1.02785s, reader_cost: 0.88268, ips: 124.53203 images/sec, eta: 0:07:20
    [2021/08/15 15:56:42] root INFO: [Train][Epoch 9/20][Iter: 4/36]lr: 0.10000, CELoss: 1.77999, loss: 1.77999, top1: 0.19531, top5: 1.00000, batch_cost: 1.02458s, reader_cost: 0.87948, ips: 124.92939 images/sec, eta: 0:07:18
    [2021/08/15 15:56:43] root INFO: [Train][Epoch 9/20][Iter: 5/36]lr: 0.10000, CELoss: 1.73607, loss: 1.73607, top1: 0.20964, top5: 1.00000, batch_cost: 0.91497s, reader_cost: 0.77112, ips: 139.89502 images/sec, eta: 0:06:30
    [2021/08/15 15:56:44] root INFO: [Train][Epoch 9/20][Iter: 6/36]lr: 0.10000, CELoss: 1.71262, loss: 1.71262, top1: 0.21429, top5: 1.00000, batch_cost: 0.91287s, reader_cost: 0.76973, ips: 140.21713 images/sec, eta: 0:06:28
    [2021/08/15 15:56:45] root INFO: [Train][Epoch 9/20][Iter: 7/36]lr: 0.10000, CELoss: 1.69233, loss: 1.69233, top1: 0.21289, top5: 1.00000, batch_cost: 0.91178s, reader_cost: 0.76877, ips: 140.38398 images/sec, eta: 0:06:27
    [2021/08/15 15:56:46] root INFO: [Train][Epoch 9/20][Iter: 8/36]lr: 0.10000, CELoss: 1.67630, loss: 1.67630, top1: 0.21962, top5: 1.00000, batch_cost: 0.91076s, reader_cost: 0.76805, ips: 140.54203 images/sec, eta: 0:06:26
    [2021/08/15 15:56:47] root INFO: [Train][Epoch 9/20][Iter: 9/36]lr: 0.10000, CELoss: 1.66700, loss: 1.66700, top1: 0.22578, top5: 1.00000, batch_cost: 0.90785s, reader_cost: 0.76541, ips: 140.99178 images/sec, eta: 0:06:24
    [2021/08/15 15:56:48] root INFO: [Train][Epoch 9/20][Iter: 10/36]lr: 0.10000, CELoss: 1.65935, loss: 1.65935, top1: 0.23011, top5: 1.00000, batch_cost: 0.90832s, reader_cost: 0.76589, ips: 140.91986 images/sec, eta: 0:06:23
    [2021/08/15 15:56:49] root INFO: [Train][Epoch 9/20][Iter: 11/36]lr: 0.10000, CELoss: 1.65663, loss: 1.65663, top1: 0.23503, top5: 1.00000, batch_cost: 0.90561s, reader_cost: 0.76326, ips: 141.34158 images/sec, eta: 0:06:21
    [2021/08/15 15:56:50] root INFO: [Train][Epoch 9/20][Iter: 12/36]lr: 0.10000, CELoss: 1.65260, loss: 1.65260, top1: 0.24399, top5: 1.00000, batch_cost: 0.90458s, reader_cost: 0.76224, ips: 141.50286 images/sec, eta: 0:06:19
    [2021/08/15 15:56:51] root INFO: [Train][Epoch 9/20][Iter: 13/36]lr: 0.10000, CELoss: 1.64832, loss: 1.64832, top1: 0.24554, top5: 1.00000, batch_cost: 0.90517s, reader_cost: 0.76269, ips: 141.40942 images/sec, eta: 0:06:19
    [2021/08/15 15:56:52] root INFO: [Train][Epoch 9/20][Iter: 14/36]lr: 0.10000, CELoss: 1.64476, loss: 1.64476, top1: 0.24583, top5: 1.00000, batch_cost: 0.90482s, reader_cost: 0.76224, ips: 141.46517 images/sec, eta: 0:06:18
    [2021/08/15 15:56:52] root INFO: [Train][Epoch 9/20][Iter: 15/36]lr: 0.10000, CELoss: 1.63356, loss: 1.63356, top1: 0.24805, top5: 1.00000, batch_cost: 0.90451s, reader_cost: 0.76195, ips: 141.51271 images/sec, eta: 0:06:17
    [2021/08/15 15:56:53] root INFO: [Train][Epoch 9/20][Iter: 16/36]lr: 0.10000, CELoss: 1.62925, loss: 1.62925, top1: 0.24954, top5: 1.00000, batch_cost: 0.90447s, reader_cost: 0.76178, ips: 141.51893 images/sec, eta: 0:06:16
    [2021/08/15 15:56:54] root INFO: [Train][Epoch 9/20][Iter: 17/36]lr: 0.10000, CELoss: 1.62595, loss: 1.62595, top1: 0.25130, top5: 1.00000, batch_cost: 0.90521s, reader_cost: 0.76248, ips: 141.40381 images/sec, eta: 0:06:15
    [2021/08/15 15:56:55] root INFO: [Train][Epoch 9/20][Iter: 18/36]lr: 0.10000, CELoss: 1.62200, loss: 1.62200, top1: 0.25164, top5: 1.00000, batch_cost: 0.90434s, reader_cost: 0.76172, ips: 141.54042 images/sec, eta: 0:06:14
    [2021/08/15 15:56:56] root INFO: [Train][Epoch 9/20][Iter: 19/36]lr: 0.10000, CELoss: 1.61393, loss: 1.61393, top1: 0.25469, top5: 1.00000, batch_cost: 0.90451s, reader_cost: 0.76191, ips: 141.51330 images/sec, eta: 0:06:13
    [2021/08/15 15:56:57] root INFO: [Train][Epoch 9/20][Iter: 20/36]lr: 0.10000, CELoss: 1.60906, loss: 1.60906, top1: 0.25446, top5: 1.00000, batch_cost: 0.90432s, reader_cost: 0.76180, ips: 141.54211 images/sec, eta: 0:06:12
    [2021/08/15 15:56:58] root INFO: [Train][Epoch 9/20][Iter: 21/36]lr: 0.10000, CELoss: 1.60219, loss: 1.60219, top1: 0.25959, top5: 1.00000, batch_cost: 0.90418s, reader_cost: 0.76173, ips: 141.56413 images/sec, eta: 0:06:11
    [2021/08/15 15:56:59] root INFO: [Train][Epoch 9/20][Iter: 22/36]lr: 0.10000, CELoss: 1.59381, loss: 1.59381, top1: 0.26562, top5: 1.00000, batch_cost: 0.90375s, reader_cost: 0.76134, ips: 141.63155 images/sec, eta: 0:06:10
    [2021/08/15 15:57:00] root INFO: [Train][Epoch 9/20][Iter: 23/36]lr: 0.10000, CELoss: 1.59625, loss: 1.59625, top1: 0.26432, top5: 1.00000, batch_cost: 0.90306s, reader_cost: 0.76069, ips: 141.74102 images/sec, eta: 0:06:09
    [2021/08/15 15:57:01] root INFO: [Train][Epoch 9/20][Iter: 24/36]lr: 0.10000, CELoss: 1.59044, loss: 1.59044, top1: 0.26906, top5: 1.00000, batch_cost: 0.90291s, reader_cost: 0.76054, ips: 141.76333 images/sec, eta: 0:06:08
    [2021/08/15 15:57:01] root INFO: [Train][Epoch 9/20][Iter: 25/36]lr: 0.10000, CELoss: 1.58653, loss: 1.58653, top1: 0.26983, top5: 1.00000, batch_cost: 0.90265s, reader_cost: 0.76032, ips: 141.80399 images/sec, eta: 0:06:07
    [2021/08/15 15:57:02] root INFO: [Train][Epoch 9/20][Iter: 26/36]lr: 0.10000, CELoss: 1.58016, loss: 1.58016, top1: 0.27112, top5: 1.00000, batch_cost: 0.90286s, reader_cost: 0.76055, ips: 141.77237 images/sec, eta: 0:06:06
    [2021/08/15 15:57:03] root INFO: [Train][Epoch 9/20][Iter: 27/36]lr: 0.10000, CELoss: 1.58084, loss: 1.58084, top1: 0.27009, top5: 1.00000, batch_cost: 0.90312s, reader_cost: 0.76086, ips: 141.73016 images/sec, eta: 0:06:05
    [2021/08/15 15:57:04] root INFO: [Train][Epoch 9/20][Iter: 28/36]lr: 0.10000, CELoss: 1.58268, loss: 1.58268, top1: 0.26967, top5: 1.00000, batch_cost: 0.90321s, reader_cost: 0.76099, ips: 141.71715 images/sec, eta: 0:06:04
    [2021/08/15 15:57:05] root INFO: [Train][Epoch 9/20][Iter: 29/36]lr: 0.10000, CELoss: 1.58225, loss: 1.58225, top1: 0.27318, top5: 1.00000, batch_cost: 0.90359s, reader_cost: 0.76141, ips: 141.65767 images/sec, eta: 0:06:04
    [2021/08/15 15:57:06] root INFO: [Train][Epoch 9/20][Iter: 30/36]lr: 0.10000, CELoss: 1.58059, loss: 1.58059, top1: 0.27394, top5: 1.00000, batch_cost: 0.90330s, reader_cost: 0.76116, ips: 141.70289 images/sec, eta: 0:06:03
    [2021/08/15 15:57:07] root INFO: [Train][Epoch 9/20][Iter: 31/36]lr: 0.10000, CELoss: 1.57775, loss: 1.57775, top1: 0.27612, top5: 1.00000, batch_cost: 0.90275s, reader_cost: 0.76061, ips: 141.78859 images/sec, eta: 0:06:02
    [2021/08/15 15:57:08] root INFO: [Train][Epoch 9/20][Iter: 32/36]lr: 0.10000, CELoss: 1.57807, loss: 1.57807, top1: 0.27770, top5: 1.00000, batch_cost: 0.90276s, reader_cost: 0.76065, ips: 141.78762 images/sec, eta: 0:06:01
    [2021/08/15 15:57:09] root INFO: [Train][Epoch 9/20][Iter: 33/36]lr: 0.10000, CELoss: 1.57236, loss: 1.57236, top1: 0.27872, top5: 1.00000, batch_cost: 0.90256s, reader_cost: 0.76048, ips: 141.81820 images/sec, eta: 0:06:00
    [2021/08/15 15:57:10] root INFO: [Train][Epoch 9/20][Iter: 34/36]lr: 0.10000, CELoss: 1.57191, loss: 1.57191, top1: 0.27969, top5: 1.00000, batch_cost: 0.90286s, reader_cost: 0.76081, ips: 141.77115 images/sec, eta: 0:05:59
    [2021/08/15 15:57:10] root INFO: [Train][Epoch 9/20][Iter: 35/36]lr: 0.10000, CELoss: 1.57121, loss: 1.57121, top1: 0.28022, top5: 1.00000, batch_cost: 0.88289s, reader_cost: 0.73796, ips: 22.65286 images/sec, eta: 0:05:50
    [2021/08/15 15:57:10] root INFO: [Train][Epoch 9/20][Avg]CELoss: 1.57121, loss: 1.57121, top1: 0.28022, top5: 1.00000
    [2021/08/15 15:57:11] root INFO: [Eval][Epoch 9][Iter: 0/4]CELoss: 1.53611, loss: 1.53611, top1: 0.32031, top5: 1.00000, batch_cost: 0.95384s, reader_cost: 0.83890, ips: 134.19428 images/sec
    [2021/08/15 15:57:12] root INFO: [Eval][Epoch 9][Iter: 1/4]CELoss: 1.50634, loss: 1.50634, top1: 0.33594, top5: 1.00000, batch_cost: 0.92121s, reader_cost: 0.80600, ips: 138.94770 images/sec
    [2021/08/15 15:57:13] root INFO: [Eval][Epoch 9][Iter: 2/4]CELoss: 2.31824, loss: 2.31824, top1: 0.32812, top5: 1.00000, batch_cost: 0.90820s, reader_cost: 0.79308, ips: 140.93761 images/sec
    [2021/08/15 15:57:13] root INFO: [Eval][Epoch 9][Iter: 3/4]CELoss: 1.57168, loss: 1.57168, top1: 0.24138, top5: 1.00000, batch_cost: 0.88279s, reader_cost: 0.76993, ips: 131.40131 images/sec
    [2021/08/15 15:57:13] root INFO: [Eval][Epoch 9][Avg]CELoss: 1.73697, loss: 1.73697, top1: 0.30800, top5: 1.00000
    [2021/08/15 15:57:14] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:57:14] root INFO: [Eval][Epoch 9][best metric: 0.3079999989271164]
    [2021/08/15 15:57:15] root INFO: Already save model in ./output/ResNet50/epoch_9
    [2021/08/15 15:57:15] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:57:16] root INFO: [Train][Epoch 10/20][Iter: 0/36]lr: 0.10000, CELoss: 1.51716, loss: 1.51716, top1: 0.29688, top5: 1.00000, batch_cost: 1.05789s, reader_cost: 0.91285, ips: 120.99580 images/sec, eta: 0:06:58
    [2021/08/15 15:57:17] root INFO: [Train][Epoch 10/20][Iter: 1/36]lr: 0.10000, CELoss: 1.52328, loss: 1.52328, top1: 0.29688, top5: 1.00000, batch_cost: 1.05303s, reader_cost: 0.90809, ips: 121.55389 images/sec, eta: 0:06:55
    [2021/08/15 15:57:18] root INFO: [Train][Epoch 10/20][Iter: 2/36]lr: 0.10000, CELoss: 1.50213, loss: 1.50213, top1: 0.30729, top5: 1.00000, batch_cost: 1.04844s, reader_cost: 0.90356, ips: 122.08570 images/sec, eta: 0:06:53
    [2021/08/15 15:57:19] root INFO: [Train][Epoch 10/20][Iter: 3/36]lr: 0.10000, CELoss: 1.53759, loss: 1.53759, top1: 0.30273, top5: 1.00000, batch_cost: 1.04447s, reader_cost: 0.89967, ips: 122.54992 images/sec, eta: 0:06:50
    [2021/08/15 15:57:20] root INFO: [Train][Epoch 10/20][Iter: 4/36]lr: 0.10000, CELoss: 1.51233, loss: 1.51233, top1: 0.29844, top5: 1.00000, batch_cost: 1.04033s, reader_cost: 0.89560, ips: 123.03793 images/sec, eta: 0:06:47
    [2021/08/15 15:57:21] root INFO: [Train][Epoch 10/20][Iter: 5/36]lr: 0.10000, CELoss: 1.49596, loss: 1.49596, top1: 0.30078, top5: 1.00000, batch_cost: 0.92328s, reader_cost: 0.77941, ips: 138.63685 images/sec, eta: 0:06:01
    [2021/08/15 15:57:22] root INFO: [Train][Epoch 10/20][Iter: 6/36]lr: 0.10000, CELoss: 1.49607, loss: 1.49607, top1: 0.29688, top5: 1.00000, batch_cost: 0.91768s, reader_cost: 0.77516, ips: 139.48244 images/sec, eta: 0:05:57
    [2021/08/15 15:57:23] root INFO: [Train][Epoch 10/20][Iter: 7/36]lr: 0.10000, CELoss: 1.51332, loss: 1.51332, top1: 0.29980, top5: 1.00000, batch_cost: 0.91483s, reader_cost: 0.77284, ips: 139.91709 images/sec, eta: 0:05:55
    [2021/08/15 15:57:24] root INFO: [Train][Epoch 10/20][Iter: 8/36]lr: 0.10000, CELoss: 1.50680, loss: 1.50680, top1: 0.29948, top5: 1.00000, batch_cost: 0.91230s, reader_cost: 0.77042, ips: 140.30470 images/sec, eta: 0:05:53
    [2021/08/15 15:57:25] root INFO: [Train][Epoch 10/20][Iter: 9/36]lr: 0.10000, CELoss: 1.53553, loss: 1.53553, top1: 0.30078, top5: 1.00000, batch_cost: 0.91067s, reader_cost: 0.76888, ips: 140.55602 images/sec, eta: 0:05:52
    [2021/08/15 15:57:25] root INFO: [Train][Epoch 10/20][Iter: 10/36]lr: 0.10000, CELoss: 1.54892, loss: 1.54892, top1: 0.29403, top5: 1.00000, batch_cost: 0.91115s, reader_cost: 0.76940, ips: 140.48152 images/sec, eta: 0:05:51
    [2021/08/15 15:57:26] root INFO: [Train][Epoch 10/20][Iter: 11/36]lr: 0.10000, CELoss: 1.54597, loss: 1.54597, top1: 0.29427, top5: 1.00000, batch_cost: 0.91217s, reader_cost: 0.77045, ips: 140.32466 images/sec, eta: 0:05:51
    [2021/08/15 15:57:27] root INFO: [Train][Epoch 10/20][Iter: 12/36]lr: 0.10000, CELoss: 1.54186, loss: 1.54186, top1: 0.29567, top5: 1.00000, batch_cost: 0.91193s, reader_cost: 0.77019, ips: 140.36200 images/sec, eta: 0:05:50
    [2021/08/15 15:57:28] root INFO: [Train][Epoch 10/20][Iter: 13/36]lr: 0.10000, CELoss: 1.53987, loss: 1.53987, top1: 0.29911, top5: 1.00000, batch_cost: 0.91097s, reader_cost: 0.76930, ips: 140.50931 images/sec, eta: 0:05:48
    [2021/08/15 15:57:29] root INFO: [Train][Epoch 10/20][Iter: 14/36]lr: 0.10000, CELoss: 1.54256, loss: 1.54256, top1: 0.29948, top5: 1.00000, batch_cost: 0.91018s, reader_cost: 0.76857, ips: 140.63154 images/sec, eta: 0:05:47
    [2021/08/15 15:57:30] root INFO: [Train][Epoch 10/20][Iter: 15/36]lr: 0.10000, CELoss: 1.54281, loss: 1.54281, top1: 0.30225, top5: 1.00000, batch_cost: 0.91024s, reader_cost: 0.76859, ips: 140.62220 images/sec, eta: 0:05:46
    [2021/08/15 15:57:31] root INFO: [Train][Epoch 10/20][Iter: 16/36]lr: 0.10000, CELoss: 1.53086, loss: 1.53086, top1: 0.30607, top5: 1.00000, batch_cost: 0.90965s, reader_cost: 0.76790, ips: 140.71383 images/sec, eta: 0:05:45
    [2021/08/15 15:57:32] root INFO: [Train][Epoch 10/20][Iter: 17/36]lr: 0.10000, CELoss: 1.53379, loss: 1.53379, top1: 0.30512, top5: 1.00000, batch_cost: 0.90974s, reader_cost: 0.76797, ips: 140.70000 images/sec, eta: 0:05:44
    [2021/08/15 15:57:33] root INFO: [Train][Epoch 10/20][Iter: 18/36]lr: 0.10000, CELoss: 1.52920, loss: 1.52920, top1: 0.30757, top5: 1.00000, batch_cost: 0.90912s, reader_cost: 0.76734, ips: 140.79501 images/sec, eta: 0:05:43
    [2021/08/15 15:57:34] root INFO: [Train][Epoch 10/20][Iter: 19/36]lr: 0.10000, CELoss: 1.52598, loss: 1.52598, top1: 0.30977, top5: 1.00000, batch_cost: 0.90849s, reader_cost: 0.76668, ips: 140.89319 images/sec, eta: 0:05:42
    [2021/08/15 15:57:34] root INFO: [Train][Epoch 10/20][Iter: 20/36]lr: 0.10000, CELoss: 1.52127, loss: 1.52127, top1: 0.31138, top5: 1.00000, batch_cost: 0.90817s, reader_cost: 0.76641, ips: 140.94239 images/sec, eta: 0:05:41
    [2021/08/15 15:57:35] root INFO: [Train][Epoch 10/20][Iter: 21/36]lr: 0.10000, CELoss: 1.51539, loss: 1.51539, top1: 0.31037, top5: 1.00000, batch_cost: 0.90807s, reader_cost: 0.76637, ips: 140.95869 images/sec, eta: 0:05:40
    [2021/08/15 15:57:36] root INFO: [Train][Epoch 10/20][Iter: 22/36]lr: 0.10000, CELoss: 1.51599, loss: 1.51599, top1: 0.31216, top5: 1.00000, batch_cost: 0.90759s, reader_cost: 0.76594, ips: 141.03299 images/sec, eta: 0:05:39
    [2021/08/15 15:57:37] root INFO: [Train][Epoch 10/20][Iter: 23/36]lr: 0.10000, CELoss: 1.51200, loss: 1.51200, top1: 0.31283, top5: 1.00000, batch_cost: 0.90689s, reader_cost: 0.76526, ips: 141.14131 images/sec, eta: 0:05:38
    [2021/08/15 15:57:38] root INFO: [Train][Epoch 10/20][Iter: 24/36]lr: 0.10000, CELoss: 1.50536, loss: 1.50536, top1: 0.31781, top5: 1.00000, batch_cost: 0.90634s, reader_cost: 0.76472, ips: 141.22756 images/sec, eta: 0:05:37
    [2021/08/15 15:57:39] root INFO: [Train][Epoch 10/20][Iter: 25/36]lr: 0.10000, CELoss: 1.50561, loss: 1.50561, top1: 0.31611, top5: 1.00000, batch_cost: 0.90535s, reader_cost: 0.76372, ips: 141.38187 images/sec, eta: 0:05:35
    [2021/08/15 15:57:40] root INFO: [Train][Epoch 10/20][Iter: 26/36]lr: 0.10000, CELoss: 1.50788, loss: 1.50788, top1: 0.31424, top5: 1.00000, batch_cost: 0.90567s, reader_cost: 0.76366, ips: 141.33199 images/sec, eta: 0:05:35
    [2021/08/15 15:57:41] root INFO: [Train][Epoch 10/20][Iter: 27/36]lr: 0.10000, CELoss: 1.50893, loss: 1.50893, top1: 0.31529, top5: 1.00000, batch_cost: 0.90586s, reader_cost: 0.76382, ips: 141.30265 images/sec, eta: 0:05:34
    [2021/08/15 15:57:42] root INFO: [Train][Epoch 10/20][Iter: 28/36]lr: 0.10000, CELoss: 1.50588, loss: 1.50588, top1: 0.31492, top5: 1.00000, batch_cost: 0.90594s, reader_cost: 0.76390, ips: 141.28967 images/sec, eta: 0:05:33
    [2021/08/15 15:57:43] root INFO: [Train][Epoch 10/20][Iter: 29/36]lr: 0.10000, CELoss: 1.50126, loss: 1.50126, top1: 0.31615, top5: 1.00000, batch_cost: 0.90585s, reader_cost: 0.76374, ips: 141.30445 images/sec, eta: 0:05:32
    [2021/08/15 15:57:44] root INFO: [Train][Epoch 10/20][Iter: 30/36]lr: 0.10000, CELoss: 1.49785, loss: 1.49785, top1: 0.31804, top5: 1.00000, batch_cost: 0.90580s, reader_cost: 0.76370, ips: 141.31145 images/sec, eta: 0:05:31
    [2021/08/15 15:57:44] root INFO: [Train][Epoch 10/20][Iter: 31/36]lr: 0.10000, CELoss: 1.49478, loss: 1.49478, top1: 0.31934, top5: 1.00000, batch_cost: 0.90607s, reader_cost: 0.76399, ips: 141.26936 images/sec, eta: 0:05:30
    [2021/08/15 15:57:45] root INFO: [Train][Epoch 10/20][Iter: 32/36]lr: 0.10000, CELoss: 1.49403, loss: 1.49403, top1: 0.31842, top5: 1.00000, batch_cost: 0.90622s, reader_cost: 0.76409, ips: 141.24661 images/sec, eta: 0:05:29
    [2021/08/15 15:57:46] root INFO: [Train][Epoch 10/20][Iter: 33/36]lr: 0.10000, CELoss: 1.49257, loss: 1.49257, top1: 0.32008, top5: 1.00000, batch_cost: 0.90590s, reader_cost: 0.76381, ips: 141.29526 images/sec, eta: 0:05:28
    [2021/08/15 15:57:47] root INFO: [Train][Epoch 10/20][Iter: 34/36]lr: 0.10000, CELoss: 1.48888, loss: 1.48888, top1: 0.32232, top5: 1.00000, batch_cost: 0.90573s, reader_cost: 0.76366, ips: 141.32209 images/sec, eta: 0:05:27
    [2021/08/15 15:57:47] root INFO: [Train][Epoch 10/20][Iter: 35/36]lr: 0.10000, CELoss: 1.48969, loss: 1.48969, top1: 0.32156, top5: 1.00000, batch_cost: 0.88561s, reader_cost: 0.74075, ips: 22.58337 images/sec, eta: 0:05:19
    [2021/08/15 15:57:47] root INFO: [Train][Epoch 10/20][Avg]CELoss: 1.48969, loss: 1.48969, top1: 0.32156, top5: 1.00000
    [2021/08/15 15:57:48] root INFO: [Eval][Epoch 10][Iter: 0/4]CELoss: 1.38415, loss: 1.38415, top1: 0.36719, top5: 1.00000, batch_cost: 0.92912s, reader_cost: 0.81468, ips: 137.76550 images/sec
    [2021/08/15 15:57:49] root INFO: [Eval][Epoch 10][Iter: 1/4]CELoss: 1.35321, loss: 1.35321, top1: 0.30469, top5: 1.00000, batch_cost: 0.90971s, reader_cost: 0.79510, ips: 140.70463 images/sec
    [2021/08/15 15:57:50] root INFO: [Eval][Epoch 10][Iter: 2/4]CELoss: 1.43609, loss: 1.43609, top1: 0.33594, top5: 1.00000, batch_cost: 0.89889s, reader_cost: 0.78420, ips: 142.39825 images/sec
    [2021/08/15 15:57:51] root INFO: [Eval][Epoch 10][Iter: 3/4]CELoss: 1.60609, loss: 1.60609, top1: 0.25000, top5: 1.00000, batch_cost: 0.87565s, reader_cost: 0.76290, ips: 132.47345 images/sec
    [2021/08/15 15:57:51] root INFO: [Eval][Epoch 10][Avg]CELoss: 1.44102, loss: 1.44102, top1: 0.31600, top5: 1.00000
    [2021/08/15 15:57:52] root INFO: Already save model in ./output/ResNet50/best_model
    [2021/08/15 15:57:52] root INFO: [Eval][Epoch 10][best metric: 0.316]
    [2021/08/15 15:57:52] root INFO: Already save model in ./output/ResNet50/epoch_10
    [2021/08/15 15:57:53] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:57:54] root INFO: [Train][Epoch 11/20][Iter: 0/36]lr: 0.10000, CELoss: 1.50306, loss: 1.50306, top1: 0.28906, top5: 1.00000, batch_cost: 1.05958s, reader_cost: 0.91479, ips: 120.80289 images/sec, eta: 0:06:21
    [2021/08/15 15:57:55] root INFO: [Train][Epoch 11/20][Iter: 1/36]lr: 0.10000, CELoss: 1.51002, loss: 1.51002, top1: 0.27734, top5: 1.00000, batch_cost: 1.05513s, reader_cost: 0.91043, ips: 121.31210 images/sec, eta: 0:06:18
    [2021/08/15 15:57:56] root INFO: [Train][Epoch 11/20][Iter: 2/36]lr: 0.10000, CELoss: 1.51143, loss: 1.51143, top1: 0.28125, top5: 1.00000, batch_cost: 1.05049s, reader_cost: 0.90587, ips: 121.84804 images/sec, eta: 0:06:16
    [2021/08/15 15:57:57] root INFO: [Train][Epoch 11/20][Iter: 3/36]lr: 0.10000, CELoss: 1.50538, loss: 1.50538, top1: 0.31055, top5: 1.00000, batch_cost: 1.04627s, reader_cost: 0.90176, ips: 122.33988 images/sec, eta: 0:06:13
    [2021/08/15 15:57:57] root INFO: [Train][Epoch 11/20][Iter: 4/36]lr: 0.10000, CELoss: 1.52895, loss: 1.52895, top1: 0.29688, top5: 1.00000, batch_cost: 1.04204s, reader_cost: 0.89761, ips: 122.83562 images/sec, eta: 0:06:10
    [2021/08/15 15:57:58] root INFO: [Train][Epoch 11/20][Iter: 5/36]lr: 0.10000, CELoss: 1.50923, loss: 1.50923, top1: 0.29818, top5: 1.00000, batch_cost: 0.89663s, reader_cost: 0.75518, ips: 142.75691 images/sec, eta: 0:05:18
    [2021/08/15 15:57:59] root INFO: [Train][Epoch 11/20][Iter: 6/36]lr: 0.10000, CELoss: 1.52728, loss: 1.52728, top1: 0.30246, top5: 1.00000, batch_cost: 0.89640s, reader_cost: 0.75570, ips: 142.79334 images/sec, eta: 0:05:17
    [2021/08/15 15:58:00] root INFO: [Train][Epoch 11/20][Iter: 7/36]lr: 0.10000, CELoss: 1.52585, loss: 1.52585, top1: 0.29590, top5: 1.00000, batch_cost: 0.89717s, reader_cost: 0.75576, ips: 142.67119 images/sec, eta: 0:05:16
    [2021/08/15 15:58:01] root INFO: [Train][Epoch 11/20][Iter: 8/36]lr: 0.10000, CELoss: 1.53790, loss: 1.53790, top1: 0.29253, top5: 1.00000, batch_cost: 0.89868s, reader_cost: 0.75688, ips: 142.43049 images/sec, eta: 0:05:16
    [2021/08/15 15:58:02] root INFO: [Train][Epoch 11/20][Iter: 9/36]lr: 0.10000, CELoss: 1.52121, loss: 1.52121, top1: 0.29453, top5: 1.00000, batch_cost: 0.90158s, reader_cost: 0.75979, ips: 141.97325 images/sec, eta: 0:05:16
    [2021/08/15 15:58:03] root INFO: [Train][Epoch 11/20][Iter: 10/36]lr: 0.10000, CELoss: 1.51398, loss: 1.51398, top1: 0.29901, top5: 1.00000, batch_cost: 0.90012s, reader_cost: 0.75836, ips: 142.20276 images/sec, eta: 0:05:15
    [2021/08/15 15:58:04] root INFO: [Train][Epoch 11/20][Iter: 11/36]lr: 0.10000, CELoss: 1.50395, loss: 1.50395, top1: 0.30208, top5: 1.00000, batch_cost: 0.89872s, reader_cost: 0.75696, ips: 142.42473 images/sec, eta: 0:05:13
    [2021/08/15 15:58:05] root INFO: [Train][Epoch 11/20][Iter: 12/36]lr: 0.10000, CELoss: 1.49516, loss: 1.49516, top1: 0.30288, top5: 1.00000, batch_cost: 0.89801s, reader_cost: 0.75618, ips: 142.53768 images/sec, eta: 0:05:12
    [2021/08/15 15:58:06] root INFO: [Train][Epoch 11/20][Iter: 13/36]lr: 0.10000, CELoss: 1.49289, loss: 1.49289, top1: 0.30190, top5: 1.00000, batch_cost: 0.89805s, reader_cost: 0.75628, ips: 142.53118 images/sec, eta: 0:05:11
    [2021/08/15 15:58:06] root INFO: [Train][Epoch 11/20][Iter: 14/36]lr: 0.10000, CELoss: 1.48761, loss: 1.48761, top1: 0.30677, top5: 1.00000, batch_cost: 0.89804s, reader_cost: 0.75633, ips: 142.53194 images/sec, eta: 0:05:10
    [2021/08/15 15:58:07] root INFO: [Train][Epoch 11/20][Iter: 15/36]lr: 0.10000, CELoss: 1.47987, loss: 1.47987, top1: 0.31152, top5: 1.00000, batch_cost: 0.90073s, reader_cost: 0.75901, ips: 142.10728 images/sec, eta: 0:05:10
    [2021/08/15 15:58:08] root INFO: [Train][Epoch 11/20][Iter: 16/36]lr: 0.10000, CELoss: 1.47752, loss: 1.47752, top1: 0.31480, top5: 1.00000, batch_cost: 0.90112s, reader_cost: 0.75945, ips: 142.04559 images/sec, eta: 0:05:09
    [2021/08/15 15:58:09] root INFO: [Train][Epoch 11/20][Iter: 17/36]lr: 0.10000, CELoss: 1.47815, loss: 1.47815, top1: 0.31207, top5: 1.00000, batch_cost: 0.90093s, reader_cost: 0.75929, ips: 142.07549 images/sec, eta: 0:05:09
    [2021/08/15 15:58:10] root INFO: [Train][Epoch 11/20][Iter: 18/36]lr: 0.10000, CELoss: 1.47327, loss: 1.47327, top1: 0.31538, top5: 1.00000, batch_cost: 0.90050s, reader_cost: 0.75879, ips: 142.14343 images/sec, eta: 0:05:07
    [2021/08/15 15:58:11] root INFO: [Train][Epoch 11/20][Iter: 19/36]lr: 0.10000, CELoss: 1.47537, loss: 1.47537, top1: 0.31484, top5: 1.00000, batch_cost: 0.90267s, reader_cost: 0.76096, ips: 141.80228 images/sec, eta: 0:05:07
    [2021/08/15 15:58:12] root INFO: [Train][Epoch 11/20][Iter: 20/36]lr: 0.10000, CELoss: 1.46918, loss: 1.46918, top1: 0.31734, top5: 1.00000, batch_cost: 0.90242s, reader_cost: 0.76068, ips: 141.84122 images/sec, eta: 0:05:06
    [2021/08/15 15:58:13] root INFO: [Train][Epoch 11/20][Iter: 21/36]lr: 0.10000, CELoss: 1.47065, loss: 1.47065, top1: 0.31605, top5: 1.00000, batch_cost: 0.90313s, reader_cost: 0.76143, ips: 141.72912 images/sec, eta: 0:05:06
    [2021/08/15 15:58:14] root INFO: [Train][Epoch 11/20][Iter: 22/36]lr: 0.10000, CELoss: 1.46469, loss: 1.46469, top1: 0.32065, top5: 1.00000, batch_cost: 0.90240s, reader_cost: 0.76076, ips: 141.84340 images/sec, eta: 0:05:05
    [2021/08/15 15:58:15] root INFO: [Train][Epoch 11/20][Iter: 23/36]lr: 0.10000, CELoss: 1.46031, loss: 1.46031, top1: 0.32031, top5: 1.00000, batch_cost: 0.90219s, reader_cost: 0.76058, ips: 141.87748 images/sec, eta: 0:05:04
    [2021/08/15 15:58:16] root INFO: [Train][Epoch 11/20][Iter: 24/36]lr: 0.10000, CELoss: 1.46404, loss: 1.46404, top1: 0.32000, top5: 1.00000, batch_cost: 0.90198s, reader_cost: 0.76040, ips: 141.90929 images/sec, eta: 0:05:03
    [2021/08/15 15:58:16] root INFO: [Train][Epoch 11/20][Iter: 25/36]lr: 0.10000, CELoss: 1.46283, loss: 1.46283, top1: 0.32272, top5: 1.00000, batch_cost: 0.90346s, reader_cost: 0.76186, ips: 141.67781 images/sec, eta: 0:05:02
    [2021/08/15 15:58:17] root INFO: [Train][Epoch 11/20][Iter: 26/36]lr: 0.10000, CELoss: 1.46774, loss: 1.46774, top1: 0.32234, top5: 1.00000, batch_cost: 0.90384s, reader_cost: 0.76226, ips: 141.61756 images/sec, eta: 0:05:01
    [2021/08/15 15:58:18] root INFO: [Train][Epoch 11/20][Iter: 27/36]lr: 0.10000, CELoss: 1.47283, loss: 1.47283, top1: 0.32059, top5: 1.00000, batch_cost: 0.90378s, reader_cost: 0.76219, ips: 141.62784 images/sec, eta: 0:05:00
    [2021/08/15 15:58:19] root INFO: [Train][Epoch 11/20][Iter: 28/36]lr: 0.10000, CELoss: 1.46790, loss: 1.46790, top1: 0.32058, top5: 1.00000, batch_cost: 0.90389s, reader_cost: 0.76229, ips: 141.61007 images/sec, eta: 0:05:00
    [2021/08/15 15:58:20] root INFO: [Train][Epoch 11/20][Iter: 29/36]lr: 0.10000, CELoss: 1.46374, loss: 1.46374, top1: 0.32396, top5: 1.00000, batch_cost: 0.90355s, reader_cost: 0.76191, ips: 141.66348 images/sec, eta: 0:04:59
    [2021/08/15 15:58:21] root INFO: [Train][Epoch 11/20][Iter: 30/36]lr: 0.10000, CELoss: 1.46429, loss: 1.46429, top1: 0.32334, top5: 1.00000, batch_cost: 0.90329s, reader_cost: 0.76164, ips: 141.70498 images/sec, eta: 0:04:58
    [2021/08/15 15:58:22] root INFO: [Train][Epoch 11/20][Iter: 31/36]lr: 0.10000, CELoss: 1.46449, loss: 1.46449, top1: 0.32495, top5: 1.00000, batch_cost: 0.90387s, reader_cost: 0.76222, ips: 141.61357 images/sec, eta: 0:04:57
    [2021/08/15 15:58:23] root INFO: [Train][Epoch 11/20][Iter: 32/36]lr: 0.10000, CELoss: 1.46243, loss: 1.46243, top1: 0.32765, top5: 1.00000, batch_cost: 0.90403s, reader_cost: 0.76237, ips: 141.58777 images/sec, eta: 0:04:56
    [2021/08/15 15:58:24] root INFO: [Train][Epoch 11/20][Iter: 33/36]lr: 0.10000, CELoss: 1.45567, loss: 1.45567, top1: 0.32858, top5: 1.00000, batch_cost: 0.90457s, reader_cost: 0.76289, ips: 141.50338 images/sec, eta: 0:04:55
    [2021/08/15 15:58:25] root INFO: [Train][Epoch 11/20][Iter: 34/36]lr: 0.10000, CELoss: 1.45271, loss: 1.45271, top1: 0.32612, top5: 1.00000, batch_cost: 0.90470s, reader_cost: 0.76300, ips: 141.48285 images/sec, eta: 0:04:54
    [2021/08/15 15:58:25] root INFO: [Train][Epoch 11/20][Iter: 35/36]lr: 0.10000, CELoss: 1.45233, loss: 1.45233, top1: 0.32578, top5: 1.00000, batch_cost: 0.88460s, reader_cost: 0.74004, ips: 22.60910 images/sec, eta: 0:04:47
    [2021/08/15 15:58:25] root INFO: [Train][Epoch 11/20][Avg]CELoss: 1.45233, loss: 1.45233, top1: 0.32578, top5: 1.00000
    [2021/08/15 15:58:26] root INFO: [Eval][Epoch 11][Iter: 0/4]CELoss: 1.67997, loss: 1.67997, top1: 0.28906, top5: 1.00000, batch_cost: 0.93393s, reader_cost: 0.81919, ips: 137.05515 images/sec
    [2021/08/15 15:58:27] root INFO: [Eval][Epoch 11][Iter: 1/4]CELoss: 1.71250, loss: 1.71250, top1: 0.28125, top5: 1.00000, batch_cost: 0.91282s, reader_cost: 0.79784, ips: 140.22548 images/sec
    [2021/08/15 15:58:28] root INFO: [Eval][Epoch 11][Iter: 2/4]CELoss: 1.41917, loss: 1.41917, top1: 0.34375, top5: 1.00000, batch_cost: 0.90741s, reader_cost: 0.79244, ips: 141.06069 images/sec
    [2021/08/15 15:58:28] root INFO: [Eval][Epoch 11][Iter: 3/4]CELoss: 1.38114, loss: 1.38114, top1: 0.33621, top5: 1.00000, batch_cost: 0.88119s, reader_cost: 0.76851, ips: 131.64031 images/sec
    [2021/08/15 15:58:28] root INFO: [Eval][Epoch 11][Avg]CELoss: 1.55220, loss: 1.55220, top1: 0.31200, top5: 1.00000
    [2021/08/15 15:58:28] root INFO: [Eval][Epoch 11][best metric: 0.316]
    [2021/08/15 15:58:29] root INFO: Already save model in ./output/ResNet50/epoch_11
    [2021/08/15 15:58:30] root INFO: Already save model in ./output/ResNet50/latest
    [2021/08/15 15:58:31] root INFO: [Train][Epoch 12/20][Iter: 0/36]lr: 0.10000, CELoss: 1.66663, loss: 1.66663, top1: 0.26562, top5: 1.00000, batch_cost: 1.03365s, reader_cost: 0.88918, ips: 123.83288 images/sec, eta: 0:05:34
    [2021/08/15 15:58:31] root INFO: [Train][Epoch 12/20][Iter: 1/36]lr: 0.10000, CELoss: 1.50555, loss: 1.50555, top1: 0.28125, top5: 1.00000, batch_cost: 1.02983s, reader_cost: 0.88542, ips: 124.29218 images/sec, eta: 0:05:32
    [2021/08/15 15:58:32] root INFO: [Train][Epoch 12/20][Iter: 2/36]lr: 0.10000, CELoss: 1.46671, loss: 1.46671, top1: 0.28906, top5: 1.00000, batch_cost: 1.02652s, reader_cost: 0.88217, ips: 124.69355 images/sec, eta: 0:05:30


### 3.3 预测一张


```python
# 更换为你训练的网络，需要预测的文件，上面训练所得到的的最优模型文件
# 我这里是不严谨的，直接使用训练集的图片进行验证，大家可以去百度搜一些相关的图片传上来，进行预测
!python3 tools/infer.py \
    -c ./ppcls/configs/quick_start/new_user/ShuffleNetV2_x0_25.yaml \
    -o Infer.infer_imgs=dataset/foods/baby_back_ribs/319516.jpg \
    -o Global.pretrained_model=output/ResNet50/best_model
```

运行完成，最后几行会得到结果如下形式：
```
[{'class_ids': [5, 1, 3, 4, 2],
'scores': [0.48433, 0.26765, 0.13903, 0.05609, 0.05162],
'file_name': 'dataset/foods/baby_back_ribs/319516.jpg', 
'label_names': ['baklava', 'beef_carpaccio', 'beef_tartare', 'apple_pie', 'baby_back_ribs']}]
```

## 四、总结
* 此项目并不是一个非常完美的项目，可以发现，预测结果不对，准确率很低。    
* 目前也只掌握了图像分类基本步骤，后续也将深入学习，扩充自己的知识储备量。
* AI创造营课程也将告一段落，但是每一位老师都非常认真负责，可以说是手把手教学，无论是从数据处理，模型训练还是项目写作等方便都令我受益匪浅。
