# 2019年遥感图像稀疏表征与智能分析竞赛 - 目标检测主题
比赛官网： http://rscup.bjxintong.com.cn/#/theme/2

使用RRPN模型，ResNeXt-101-FPN作为backbone，mAP达到0.336（单模，无任何trick）。主要参考：https://github.com/mjq11302010044/RRPN_pytorch



## 环境搭建

```
# first, make sure that your conda is setup properly with the right environment# for that, check that `which conda`, `which pip` and `which python` points to the# right path. From a clean conda env, this is what you need to do
conda create -n rrpn_pytorch python=3.6
conda activate rrpn_pytorch

# this installs the right pip and dependencies for the fresh python
conda install ipython

# maskrcnn_benchmark and coco api dependencies
pip install ninja yacs cython matplotlib tqdm opencv-python （可以先which pip确认下pip在当前环境）

# follow PyTorch installation in https://pytorch.org/get-started/locally
 (注意,这里一定要写=1.0或1.1,否则默认装1.2版本,后面一堆问题....)
conda install -c pytorch pytorch-nightly=1.0 torchvision=0.2.1 cudatoolkit=10.0（cudatoolkit的版本可以用nvcc -V看一下cuda编译器的版本，这两个最好一致！另外注意如果是RTX等图灵架构系列，一定不要用9.0版本的cudatoolkit，必须重装10.0，否则后面执行train_net.py时会报错）

# install pycocotoolscd ~/github
git clone https://github.com/cocodataset/cocoapi.git
cd cocoapi/PythonAPI
python setup.py build_ext install

# install RRPN
git clone https://github.com/clw5180/remote_sensing_object_detection_2019.git
cd RRPN_pytorch
# the following will install the lib with# symbolic links, so that you can modify
# the files if you want and won't need to
# re-build it
python setup.py build develop（一定注意：后期若有修改maskrcnn-benchmark文件夹下的代码，一定要用这条命令重新编译！）

# 后面这几步是RRPN项目特有的
python rotation_setup.py install
mv build/lib.linux-x86_64-3.6/rotation/*.so ./rotation (如果python版本不是3.6，就改成版本对应的数字）

# or if you are on macOS# MACOSX_DEPLOYMENT_TARGET=10.9 CC=clang CXX=clang++ python setup.py build develop
```

坑比较多，可以参考：

https://github.com/facebookresearch/maskrcnn-benchmark

https://zhuanlan.zhihu.com/p/64605565



## 训练

1、采用800x800的crop，overlap取256，大约生成25000张的图片。注意crop的同时要转换出对应的txt文件，放在labelTxt下。具体实现可以参考我的github，位置：Python_Practice/常用数据预处理脚本/超大图片crop/2019遥感_Step1_train_crop切图.py 

2、在maskrcnn_benchmark/config/paths_catalog.py根据个人数据集的路径进行修改

3、执行python setup.py build develop（只要maskrcnn_benchmark目录下任何文件修改,都需要做这一步，目的是同步build文件夹下的内容）

```
if __name__ == '__main__':
    get_DOTA('train', '/media/clwclw/Elements/deep_learning/competion/2019yaogan/train/train_crop_800')  # 根据个人数据集的路径进行修改
```

```
        "RRPN_train": {  # including IC13 and IC15
            'dataset_list':{
                'DOTA': '/media/clwclw/Elements/deep_learning/competion/2019yaogan/train/train_crop_800/' # 根据个人数据集的路径进行修改
            },
            "split": 'train'
        },
```

```
        elif "RRPN_train" in name:
            data_dir = DatasetCatalog.DATA_DIR
            attrs = DatasetCatalog.DATASETS['RRPN_train']
            args = dict(
                database=attrs["dataset_list"],
            )
            return dict(
                factory="RotationDataset",
                args=args,
            )
```

4、tools文件夹下，执行python train_net.py



## 预测

1、demo文件夹下，执行python RRPN_Demo.py

主要是增加了在线crop，对bbox进行坐标变换，再用nms去掉overlap的重复检测框。



## 其他注意事项

- 如果需要作一定修改，或者遇到问题，需要注意几点：

1、我这里首先在maskrcnn_benchmark/data/datasets/rotation_series.py里自己写了一个获取数据集标注的方法get_DOTA()，主要是为了将DOTA数据集（也就是本次比赛的数据集）的标注数据以一定格式读进来，即下面的im_infos；

```
        im_info = {
            'gt_classes': gt_classes,
            'max_classes': max_classes,
            'image': im_path,
            'boxes': gt_boxes,
            'gt_words': gt_words,
            'flipped': False,
            'gt_overlaps': overlaps,
            'seg_areas': seg_areas,
            'height': im.shape[0],
            'width': im.shape[1],
            'max_overlaps': max_overlaps,
            'rotated': True
        }
        im_infos.append(im_info)
        ...
        return im_infos
```

同时也会将im_infos转换成pkl格式文件，然后保存在tools/data_cache下，

```
    f_save_pkl = open(cache_pkl, 'wb')
    pickle.dump(im_infos, f_save_pkl)
    f_save_pkl.close()
    print("Save pickle done.")
```

以便下一次再训练的时候可以直接读入：

```
    if os.path.isfile(cache_pkl):
        return pickle.load(open(cache_pkl, 'rb'))
```

这里我首先删掉了50kB以上的txt及对应的images（因为比如small-vehicle这个类别，在一张800x800的切图中最多能达到2500+的个数，所以很容易就会OOM），然后过滤掉了训练集中小于8x8的bbox（会在训练中引起一些overlap的错误或loss=nan的现象，另外这么小的尺度，下采样几次之后在feature map基本就找不到了，因此也没必要保留），。

```
        elif "RRPN_train" in name:
            data_dir = DatasetCatalog.DATA_DIR
            attrs = DatasetCatalog.DATASETS['RRPN_train']
            args = dict(
                database=attrs["dataset_list"],
            )
            return dict(
                factory="RotationDataset",
                args=args,
            )
```

当然也可以将数据集转化为COCO官方提供的json格式，只需要在maskrcnn_benchmark/config/paths_catalog.py中增加如下内容：

```
        "DOTA_train": { 
            "img_dir": '/media/clwclw/Elements/deep_learning/competion/2019yaogan/train/train_crop_800/images/',
            "ann_file": '/media/clwclw/Elements/deep_learning/competion/2019yaogan/train/train_crop_800/train.json',
            "split": 'train'
        },
```

```python
        elif "DOTA_train" in name:
            data_dir = DatasetCatalog.DATA_DIR
            attrs = DatasetCatalog.DATASETS[name]
            args = dict(
                root=os.path.join(data_dir, attrs["img_dir"]),
                ann_file=os.path.join(data_dir, attrs["ann_file"]),
            )
            return dict(
                factory="COCODataset",    # 通过官方API导入json格式数据
                args=args,
            )
```



- 此外如果使用其他数据集进行训练，还有几个地方需要修改：

1、配置文件e2e_rrpn_X_101_32x8d_FPN_1x_DOTA.yaml需要根据情况进行修改：

```
DATASETS:
  TRAIN: ("RRPN_train", )

    ......
  ROI_BOX_HEAD:
    NUM_CLASSES: 19 # 18类物体 + 1背景


OUTPUT_DIR: './models/DOTA/'
```

2、tools/train_net.py中，需要根据情况修改成自己的.yaml配置文件，如default="../configs/rrpn/e2e_rrpn_X_101_32x8d_FPN_1x_DOTA.yaml"

3、修改maskrcnn_benchmark/config/paths_catalog.py中需要添加自己的数据集，如

```
"XXXX_train": {  
"img_dir":'/home/xxx/2019yaogan/train_data/images/',    
"ann_file":'/home/xxx/2019yaogan/train_data/Annotations/train.json',   
"split": 'train'
},
```

4、在maskrcnn_benchmark/data/datasets/rotation_series.py中需要修改的地方

```
DATASET = {
    'IC13':get_ICDAR2013,
    'IC15':get_ICDAR2015_RRC_PICK_TRAIN,
    'IC17mlt':get_ICDAR2017_mlt,
    'LSVT':get_ICDAR_LSVT_full,
    'ArT':get_ICDAR_ArT,
    'ReCTs':get_ICDAR_ReCTs_full,
    'DOTA':get_DOTA,   # 增加自己的数据集名称及对应获取数据的方法
}
```

```
class RotationDataset(torch.utils.data.Dataset):
    CLASSES = (
        "__background__ ", #"background",
        "roundabout",
        "tennis-court",
        "swimming-pool",
        "storage-tank",
        "soccer-ball-field",
        "small-vehicle",
        "ship",
        "plane",
        "large-vehicle",
        "helicopter",
        "harbor",
        "ground-track-field",
        "bridge",
        "basketball-court",
        "baseball-diamond",
        "helipad",
        "airport",
        "container-crane"  # 
    )
```



写得比较仓促，如有遗漏或者遇到问题可在issue提出，我会及时修正。

欢迎star！