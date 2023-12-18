# 3D Gaussian Splatting for Real-Time Radiance Field Rendering
## 环境配置
https://github.com/graphdeco-inria/gaussian-splatting
## 新增功能
- [x] 支持nerfstudio的transform.json格式数据
- [x] 支持来自polycam的transform_polycam.json格式数据
- [x] 支持来自大疆制图的transform.json格式数据和对应的点云init
- [x] 提供来自slam的数据转换脚本
- [x] 支持渲染深度图
- [x] 支持来自单目深度估计的深度监督
- [x] 基于taichi的全新可视化界面，支持场景编辑，场景导入等功能
## 分割分支功能
- [x] 支持分割结果交互可视化
- [x] 支持结合分割的删除复制等场景编辑
- [x] 分割图的实时查看以及渲染结果导出

## 数据集准备及模型训练

参考原repo：https://github.com/graphdeco-inria/gaussian-splatting

### nerfstudio(polycam以及rtk)格式数据训练
#### 数据准备：

    |- dataset
    |- |- images
    |- |- images_2
    |- |- images_4
    |- |- images_8
    |- |- transforms.json
    |- |- points3d.ply (可选，从slam生成或者从sfm生成)

#### 训练模型：
> python train.py -s <path to NeRF Studio or instant ngp dataset> --eval # Train with train/test split

### slam获取的数据训练
#### 数据准备：

    |- slam dataset
    |- |- images
    |- |- images_2
    |- |- images_4
    |- |- images_8
    |- |- Keyframes.txt（comap格式）
    |- |- points3d.ply (可选，slam生成)


* 首先需要进行数据转换
>python process_data/slam2nerf.py #将路径改为slam得到的数据集路径


此脚本还支持对数据集的一些规则划分方法，如序列划分以及localrf中提到的划分方法

* 得到nerfstudio格式的数据


    |- dataset
    |- |- images
    |- |- images_2
    |- |- images_4
    |- |- images_8
    |- |- transforms.json
    |- |- points3d.ply (可选，从slam生成或者从sfm生成)


### 深度图渲染
根据PR：https://github.com/graphdeco-inria/gaussian-splatting/pull/30，实现了对深度图的渲染
>python render.py -m <path to trained model> # Generate renderings
### 深度监督
在深度渲染的基础上，参考[Towards Robust Monocular Depth Estimation: Mixing Datasets for Zero-shot Cross-dataset Transfer](https://arxiv.org/abs/1907.01341)
实现了深度监督损失 
##### 如何启动深度监督？
1.运行单目深度估计DPT算法估计数据集的深度图
> python DPT/run_monodepth.py #修改数据集路径
2. 算法将会自行检测数据集路径下是否存在命名为depth的文件夹，如果存在，将会读取图像并进行深度监督
> python train.py -s <path to NeRF Studio or instant ngp dataset> --using_depth(可选) --eval # Train with train/test split 
> --depth_loss_choice in ["localrf", "rank_loss", "continue_loss, "hybrid_loss", "l1_loss"]

### 外观潜入学习
> python train.py -s <path to NeRF Studio or instant ngp dataset> --able_appearance_embedding(可选，是否学习外观潜入来应对光照变化剧烈的场景) --eval # Train with train/test split 

### segment 3DGS
#### 首先切换到分割分支
> git checkout add_segment_feature_with_3d_mean
#### 其次根据标注的分割结果更改光栅器的分割配置，并重新编译
> vi submodules/diff-gaussian-rasterization/cuda_rasterizer/config.h
> cd submodules/diff-gaussian-rasterization $$ pip install -v -e .
#### 发起训练并指定启用分割
> python train_segment.py -s <path to NeRF Studio or instant ngp dataset> --using_seg --eval -r 4

### 基于taichi的可视化界面
#### 特征
- [x] 背景裁剪
- [x] 场景融合(融合多个子场景联合渲染)
- [x] 场景编辑(放大，缩小，移动，旋转等)
- [x] 子场景编号投影(可以更好的选择子场景)
- [x] 深度渲染
- [x] 协方差缩放渲染
#### 按键介绍
- 旋转平移

      q(上移) w(前) e(下移)  |  u      i(上旋) o
      a(左)   s(后) d(右)   |  j(左旋) k(下旋) l(右旋) 

- z 协方差尺度控制[1.0, 0.25, 0.1, 0.01, 0.001]
- c clip模式切换[True, False]
- v 保存当前裁剪场景
- b 子场景复制，直接复制crop到的区域为子场景表示
- n 主场景移除，删除crop的区域从而达到物体移除的效果
- g 子场景编号投影
- p 投影模式切换[xyz, bbox, none]（在crop 模式下执行）
- r 渲染训练路径图像
- 1~9 场景选择按钮，选择对应的子场景
- , 添加关键帧
- . 删除关键帧
- SPACE(空格) 关键帧插值渲染
----------------------
- x 依次分割标注的物体
- m 添加分割图渲染模式[segment(全景分割图), hybrid(与rgb结果叠加)]

> python visualizer.py -m model_path --fast_gui（是否使用图像cache来加速，开启无法启用cilp功能） --low_memory（是否只load pose，从而减少内存消耗） -r（render 分辨率）

## QA
Q1：分割光栅器从哪里下载？
A1: 光栅器独立于该算法库存在，需要从 https://codeup.aliyun.com/62bad13611fc0f0c9e2adf9d/library-robot/diff-gaussian-rasterization.git clone
