简体中文 | [English](README.md)

# AllInOne

- [任务介绍](#任务介绍)
- [数据集介绍](#数据集介绍)
  * [训练集](#训练集)
  * [测试集](#测试集)
- [方法介绍](#方法介绍)
  * [基础框架](#基础框架)
  * [统一各任务的配置](#统一各任务的配置)
  * [异构batch与同构batch](#异构batch与同构batch)
  * [任务过拟合问题](#任务过拟合问题)
- [UFO与单任务SOTA结果对比](#UFO与单任务SOTA结果对比)
- [Demo](#Demo)
- [Aistudio_Demo](#Aistudio_Demo)

## 任务介绍
UFO 这个技术设想的出发点是视觉的大一统，即一个模型能够覆盖所有主流的视觉任务。我们从垂类应用出发，选择了人脸、人体、车辆、商品四个任务作为视觉模型大一统的第一步。AllInOne的目标是，一个模型超过四个任务的 SOTA 结果。

## 数据集介绍
我们使用了脸、人体、车辆、商品的公开数据集具体如下:

### 训练集

| **任务**                      | **数据集**                     | **图片数**                     | **类别数**                     |
| :-----------------------------| :----------------------------: | :----------------------------: | :----------------------------: |
| 人脸                          |           MS1M-V3              |           5,179,510            |           93,431               |
| 人体                          |           Market1501-Train     |           12,936               |           751                  |
| 人体                          |           DukeMTMC-Train       |           16,522               |           702                  |
| 人体                          |           MSMT17-Train         |           30,248               |           1,041                |
| 车辆                          |           Veri-776-Train       |           37,778               |           576                  |
| 车辆                          |           VehicleID-Train      |           113,346              |           13,164               |
| 车辆                          |           VeriWild-Train       |           277,797              |           30,671               |
| 商品                          |           SOP-Train            |           59,551               |           11,318               |


### 测试集

| **任务**                      | **数据集**                     | **图片数**                     | **类别数**                     |
| :-----------------------------| :----------------------------: | :----------------------------: | :----------------------------: |
| 人脸                          |           LFW                  |           12,000               |           -                    |
| 人脸                          |           CPLFW                |           12,000               |           -                    |
| 人脸                          |           CFP-FF               |           14,000               |           -                    |
| 人脸                          |           CFP-FP               |           14,000               |           -                    |
| 人脸                          |           CALFW                |           12,000               |           -                    |
| 人脸                          |           AGEDB-30             |           12,000               |           -                    |
| 人体                          |           Market1501-Test      |           19,281               |           750                  |
| 人体                          |           DukeMTMC-Test        |           19,889               |           702                  |
| 人体                          |           MSMT17-Test          |           93,820               |           3,060                |
| 车辆                          |           Veri-776-Test        |           13,257               |           200                  |
| 车辆                          |           VehicleID-Test       |           19,777               |           2,400                |
| 车辆                          |           VeriWild-Test        |           138,517              |           10,000               |
| 商品                          |           SOP-Test             |           60,502               |           11,316               |

## 方法介绍

### 基础框架
给定4个任务，我们从各个任务中采样一些数据组成一个 batch，将该 batch 输入给共享的 Transformer 骨干网络，最后分出 4 个头部网络，每个头部网络负责一个任务的输出。4 个任务分别计算损失后求和作为总的损失。使用 SGD 优化器进行模型优化。

### 统一各任务的配置
由于不同任务使用的输入大小以及模型结构都差异较大。从模型优化的层面来说，batch size 大小、学习率大小乃至于优化器都各不相同。为了方便后续的多任务训练，我们首先统一各个任务的模型结构以及优化方法。特别地，我们使用 Transformer 作为骨干网络。统一后的配置如下表所示：

|                               |        **人脸**/ **人体**/**商品**/**车辆**          |
| :-----------------------------| :----------------------------------------------------|
| 输入大小                      |    384 × 384                                         |
| Batch Size                    |    1024/512/512/512                                  |
| 数据增强                      |    Flipping + Random Erasing + AutoAug               |
| 模型结构                      |    ViT-Large                                         |
| 输出特征维度                  |    1024                                              |
| 损失函数                      |    CosFace Loss/(CosFace Loss + Triplet Loss)*3      |
| 优化器                        |    SGD                                               |
| 初始学习率                    |    0.2                                               |
| LR scheduler                  |    Warmup + Cosine LR                                |
| 迭代次数                      |    100,000                                           |

### 异构batch与同构batch 
多任务学习首先面临的问题是如何构建 Batch。常用的方式有两种，一种是同构的 Batch 组成，即 Batch 内的数据均来自同一个任务，通过不同的 Batch 选择不同的任务来保证训练所有任务。另一种是异构的 Batch 组成，即Batch 内的数据来自不同的任务。

同构的 Batch 组成面临的问题是当模型中使用 Batch Norm 这一常见的操作时，因为训练时的统计值（单任务统计值）和测试时的统计值（多任务统计值）差异较大，导致模型效果较差。我们使用ResNet50结构在人体 Market1501 和商品SOP两个任务中验证该结论。如表所示，使用异构 Batch 混合可以大幅提高两任务的效果。因此，我们采用异构Batch。

|    Batch 数据混合    |         Market1501 (rank1/mAP)    |        SOP (rank1)        |
| :--------------------| :--------------------------------:|:-------------------------:|
|  同构                |           73.13 / 50.58           |          79.54            |
|  异构                |           94.27 / 85.77           |          85.76            |

### 任务过拟合问题
在我们的四个任务中，人体和商品的训练集数量最小，都只有 6 万张图片左右，而人脸和车辆则各有约 500 万和 40 万张图片。因此在多任务训练过程中，呈现出了人体、商品快速过拟合，而人脸和车辆欠拟合的现象。为此我们探索了较多不同的手段，其中较为有效的是使用 Drop Path 正则化方法。如表所示，在将 drop path rate 从 0.1 增大到 0.2 后，人体和商品任务有较大提升，同时其他任务效果持平或更好。

|        Model     | DropPath |  CALFW | CPLFW  |  LFW  | CFP-FF | CFP-FP | AGEDB-30 | Market1501  | DukeMTMC    | MSMT17      |   Veri776   |  VehicleID  |  VeriWild   |  SOP  |
| :----------------|----------|--------| :------|-------|--------|--------|---------:|:------------|-------------|-------------|-------------|-------------|-------------|------:|
|  UFO (ViT-Large) | 0.1      |  96.18 | 94.22  | 99.83 |  99.90 |  99.09 |   98.17  | 96.17/91.67 | 92.01/84.63 | 86.21/68.94 | 97.62/88.66 | 85.35/90.09 | 93.31/77.98 | 87.11 |
|  UFO (ViT-Large) | 0.2      |  95.92 | 94.30  | 99.82 |  99.90 |  99.11 |   98.03  | 96.28/92.75 | 92.55/86.19 | 88.10/72.17 | 97.74/89.25 | 87.62/91.32 | 93.62/78.91 | 89.23 |

## UFO与单任务SOTA结果对比

UFO没有使用任何rerank策略以及外部数据。

|                  Model                     |  CALFW | CPLFW  |  LFW  | CFP-FF | CFP-FP | AGEDB-30 | Market1501  | DukeMTMC    | MSMT17      |   Veri776   |  VehicleID  |  VeriWild   |  SOP  |
| :------------------------------------------|--------| :------|-------|--------|--------|---------:|:------------|-------------|-------------|-------------|-------------|-------------|------:|
| previous SOTA (w/o rerank & external data) |  96.20 | 93.37  | 99.85 |  99.89 |  98.99 |   98.35  | 96.3/91.5   | 92.1/83.7   | 86.2/69.4   | 97.0/87.1   | 80.3/86.4   | 92.5/77.3   | 85.9  |
|           UFO (ViT-Large)                  |  95.92 | 94.30  | 99.82 |  99.90 |  99.10 |   97.95  | 96.40/92.60 | 92.86/86.07 | 88.07/72.17 | 98.21/89.27 | 86.13/90.70 | 93.53/78.92 | 89.05 |


## Demo
提供了测试代码和AllInOne模型(allinone_vitlarge.pdmodel）用来
复现在人脸、人体、车辆、商品四个任务上13个测试数据集的精度，目前仅支持单卡测试。

### 环境配置
运行环境为python3.7，cuda11.0和cudnn8.0，测试机器为A100。使用pip的安装依赖包，如下：
```bash
pip install -U pip==22.0.3
pip install -r requirements.txt
pip install faiss-gpu==1.7.2 --no-cache
pip install paddlepaddle-gpu==2.2.2.post110 -f https://www.paddlepaddle.org.cn/whl/linux/mkl/avx/stable.html
```

### 数据配置

[测试数据集地址](https://aistudio.baidu.com/aistudio/datasetdetail/128138)
```bash
cd /path/to/data
# 18 GB
tar -xf test_datasets.tar
```
### 开始测试
```bash
cd /path/to/AllInOne
```

然后下载allinone_vitlarge.pdmodel模型
[模型地址](https://aistudio.baidu.com/aistudio/datasetdetail/128025)
```

配置环境变量
```bash
export CUDA_VISIBLE_DEVICES=0
export PYTHONPATH=/path/to/AllInOne
export FASTREID_DATASETS=/path/to/data/test_datasets
```
并且运行测试脚本，计算在13个数据集上的测试精度
```bash
python  evaluation_script.py --batchsize 8 --model_path allinone_vitlarge.pdmodel
```
也可以仅计算在单个数据集上的测试精度，可选的数据集名称为其中['CALFW', 'CPLFW', 'LFW', 'CFP_FF', 'CFP_FP', 'AgeDB_30', 'Market1501', 
 'MSMT17', 'VeRi', 'LargeVehicleID', 'LargeVeRiWild', 'SOP']一项，以人脸数据集CALFW为例
```bash
python  evaluation_script.py --batchsize 8 --model_path allinone_vitlarge.pdmodel --test_datasets CALFW
```


## Aistudio_Demo

[Aistudio Demo](https://aistudio.baidu.com/aistudio/projectdetail/3488098)

我们还在百度的一站式开发平台Aistudio中提供了环境数据都配置好，可以一件运行的Demo, 欢迎大家尝试！

致谢：部分数据集构建和测评代码参考了https://github.com/facebookresearch/detectron2 和 https://github.com/JDAI-CV/fast-reid , 表示感谢！

如遇到问题请联系xiteng01@baidu.com

