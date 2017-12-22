---
title: LMDB介绍与在Caffe中的使用
date: 2017-12-22 00:36:52
tags:
---

## LMDB

----
深度学习经常会需要处理图片文件,图片文件因为格式大小等因素差异多多,需要统一处理成神经网络所需要的输入形式。Caffe中将图片文件处理成LMDB或者LevelDB格式，它们都是键/值对（Key/Value Pair）嵌入式数据库管理系统编程库。

[LMDB](http://www.lmdb.tech/doc/)全称为Lightning Memory-Mapped Database Manager。它有着出色的IO性能，并且十分适合处理大规模的数据。LMDB的内存消耗是LevelDB的1.1倍，但是速度比后者快10%到15%，并且支持多种训练模型同时读取同一组数据集，是Caffe中默认的数据处理格式。

> LMDB is a Btree-based database management library modeled loosely on the BerkeleyDB API, but much simplified. The entire database is exposed in a memory map, and all data fetches return data directly from the mapped memory, so no malloc's or memcpy's occur during data fetches. As such, the library is extremely simple because it requires no page caching layer of its own, and it is extremely high performance and memory-efficient.

## 生成Mnist数据集的LMDB文件

----
我们先看一下[Caffe官网上](http://caffe.berkeleyvision.org/gathered/examples/mnist.html)是如何创建Mnist数据集的LMDB文件的。

    cd $CAFFE_ROOT
    ./data/mnist/get_mnist.sh
    ./examples/mnist/create_mnist.sh
CAFFE_ROOT是caffe的主目录，必须在Caffe的主目录下运行命令，否则会报错无法找到对应的可执行文件。

get_mnist.sh用来下载Mnist数据集。
>If it complains that wget or gunzip are not installed, you need to install them respectively.

create_mnist.sh用来创建Mnist数据集的LMDB文件。

    #!/usr/bin/env sh
    # This script converts the mnist data into lmdb/leveldb format,
    # depending on the value assigned to $BACKEND.
    set -e

    EXAMPLE=examples/mnist
    DATA=data/mnist
    BUILD=build/examples/mnist

    BACKEND="lmdb"

    echo "Creating ${BACKEND}..."


    rm -rf $EXAMPLE/mnist_train_${BACKEND}
    rm -rf $EXAMPLE/mnist_test_${BACKEND}

    $BUILD/convert_mnist_data.bin $DATA/train-images-idx3-ubyte \
    $DATA/train-labels-idx1-ubyte $EXAMPLE/mnist_train_${BACKEND} --backend=${BACKEND}

    $BUILD/convert_mnist_data.bin $DATA/t10k-images-idx3-ubyte \
    $DATA/t10k-labels-idx1-ubyte $EXAMPLE/mnist_test_${BACKEND} --backend=${BACKEND}

    echo "Done."

其中

* EXAMPLE路径用来保存生成好的LMDB文件

* DATA是数据文件夹的l路径

* BUILD是可执行文件的路径，这里是convert_mnist_data.bin

* rm -rf这两句用来删除可能存在的LMDB文件，如果已存在同名的LMDB文件而又没有删除，程序会报错。

这样就生成了Mnist数据集的LMDB文件,参照[官网教程](http://caffe.berkeleyvision.org/gathered/examples/mnist.html)可以调用Lenet网络开始训练Mnist数据集，这里不再赘述。

## 生成图片数据的LMDB文件

----

那么如何生成我们自己图片的LMDB文件呢?这里我们可以参照Caffe自带的CAFFE_ROOT/examples/imagenet/create_imagenet.sh

    #!/usr/bin/env sh
    # Create the imagenet lmdb inputs
    # N.B. set the path to the imagenet train + val data dirs
    set -e

    EXAMPLE=examples/imagenet
    DATA=data/ilsvrc12
    TOOLS=build/tools

    TRAIN_DATA_ROOT=/path/to/imagenet/train/
    VAL_DATA_ROOT=/path/to/imagenet/val/

    # Set RESIZE=true to resize the images to 256x256. Leave as false if images have
    # already been resized using another tool.
    RESIZE=false
    if $RESIZE; then
    RESIZE_HEIGHT=256
    RESIZE_WIDTH=256
    else
    RESIZE_HEIGHT=0
    RESIZE_WIDTH=0
    fi

    if [ ! -d "$TRAIN_DATA_ROOT" ]; then
    echo "Error: TRAIN_DATA_ROOT is not a path to a directory: $TRAIN_DATA_ROOT"
    echo "Set the TRAIN_DATA_ROOT variable in create_imagenet.sh to the path" \
        "where the ImageNet training data is stored."
    exit 1
    fi

    if [ ! -d "$VAL_DATA_ROOT" ]; then
    echo "Error: VAL_DATA_ROOT is not a path to a directory: $VAL_DATA_ROOT"
    echo "Set the VAL_DATA_ROOT variable in create_imagenet.sh to the path" \
        "where the ImageNet validation data is stored."
    exit 1
    fi

    echo "Creating train lmdb..."

    GLOG_logtostderr=1 $TOOLS/convert_imageset \
        --resize_height=$RESIZE_HEIGHT \
        --resize_width=$RESIZE_WIDTH \
        --shuffle \
        $TRAIN_DATA_ROOT \
        $DATA/train.txt \
        $EXAMPLE/ilsvrc12_train_lmdb

    echo "Creating val lmdb..."

    GLOG_logtostderr=1 $TOOLS/convert_imageset \
        --resize_height=$RESIZE_HEIGHT \
        --resize_width=$RESIZE_WIDTH \
        --shuffle \
        $VAL_DATA_ROOT \
        $DATA/val.txt \
        $EXAMPLE/ilsvrc12_val_lmdb

    echo "Done."

相比于create_mnist.sh只用于Mnist数据集，create_imagenet经过简单的修改可用于处理任何图片数据。create_imagenet.sh按照train.txt和val.txt将读取图片数据生成对应的LMDB文件。需要注意以下几点：

* 将train.txt和val.txt放在DATA路径下,或者自己在create_imagenet.sh中修改路径，保证create_imagenet.sh可以找到txt文件。

* TRAIN_DATA_ROOT和VAL_DATA_ROOT可以相同，即所有的图片数据可以放在一个文件夹中。这也是推荐的做法，处理起来更加方便。

* TRAIN_DATA_ROOT和VAL_DATA_ROOT最后的反斜杠/不可以少。如果非要删除的话，需要在train.txt和val.txt文件中的每个图片文件名前添加一个反斜杠/。否则图片路径会不正确，无法读取图片。

* 实际情况我们不仅需要训练集和验证集，还需要测试集。我选择的图片划分比例是train:test:val=7:2:1。我们可以添加生成测试集LMDB的命令，照猫画虎即可。

## train.txt、val.txt和test.txt的生成

----
现在来考虑一下，我们拿到了一批图片，想要训练一个针对性的分类网络。以Mnist数据集为例，我们将Mnist数据集的60000训练集和10000测试集图片提取出来，保存在MnistPics文件夹中。而且我们想按照train:test:val=7:2:1的比例将Mnist数据集分为3部分。这样的话，训练模型的时候我们可以使用训练集和验证集，模型训练好了之后，我们可以使用测试集测试模型的泛化效果。

现在我们需要根据MnistPics得到对应的xxx.txt文件，Python代码如下：

    import os
    import random

    class GenerateTxts(object):
        # txt_save_dirname_path默认为空，即txt文件保存在当前文件夹下
        def __init__(self,pic_dir_path,classes,txt_save_dirname_path=''):
            # 不同数据集的占比
            self.train_percent=0.7
            self.test_percent=0.2
            self.val_percent=0.1

            # 图片文件夹的路径
            self.pic_dir_path=pic_dir_path

            # 类别列表，元素是str，如['0','1',...,'9']
            self.classes=classes

            # 保存txt的文件夹路径
            self.txt_dirname_path=txt_save_dirname_path

            # txt文件的路径
            self.train_txt_save_path=os.path.join(self.txt_dirname_path,'train.txt')
            # 因为是追加方式保存，所以每次重新生成必须删除之前的txt文件
            if os.path.exists(self.train_txt_save_path):
                os.remove(self.train_txt_save_path)

            self.test_txt_save_path=os.path.join(self.txt_dirname_path,'test.txt')
            if os.path.exists(self.test_txt_save_path):
                os.remove(self.test_txt_save_path)

            self.val_txt_save_path=os.path.join(self.txt_dirname_path,'val.txt')
            if os.path.exists(self.val_txt_save_path):
                os.remove(self.val_txt_save_path)

            # 产生txt文件
            self.generate_txts()

        def generate_txts(self):
            # 读取文件夹中的全部文件名
            pic_names = os.listdir(self.pic_dir_path)

            # 建立索引字典并初始化
            index_set = dict()
            for i in range(len(classes)):
                index_set[i] = list()
            
            # 将图片划分到对应的类别
            for name_cnt, name in enumerate(pic_names):
                # 这里默认文件名中信息以_分割，第一个信息是图片所属类别
                # 如3_xxx.png表示3的图片，xxx用来和其他3的图片区分开
                char = name.split('_')[0]
                char_index = classes.index(char)
                index_set[char_index].append(name)

            # 按照比例将图片名划分到不同txt文件中
            for i in range(len(classes)):
                name_lst = index_set[i]
                # 将某一类别的文件名打乱
                random.shuffle(name_lst)
                # 按照比例划分图片数据
                train_number = int(len(name_lst) * self.train_percent)
                test_number = int(len(name_lst) * self.test_percent)
                train_part = name_lst[:train_number]
                test_part = name_lst[train_number:train_number + test_number]
                valid_part = name_lst[train_number + test_number:]

                # 将文件名添加到txt文件中
                with open(self.train_txt_save_path, 'a')as f:
                    for name in train_part:
                        f.write(name + ' ' + str(i) + '\n')

                with open(self.test_txt_save_path, 'a')as f:
                    for name in test_part:
                        f.write(name + ' ' + str(i) + '\n')

                with open(self.val_txt_save_path, 'a')as f:
                    for name in valid_part:
                        f.write(name + ' ' + str(i) + '\n')


    if __name__ == '__main__':
        pic_dir_path='path/to/your/pic_dir_path'
        # 对于Mnist数据，classes如下
        classes=[str(i) for i in range(10)]
        GenerateTxts(pic_dir_path,classes)

得到txt文件后，我们就可以利用改好的create_imagenet生成特定图片数据的LMDB文件了。

## 参考

----
[Mnist数据集创建lmdb或leveldb类型的数据](http://blog.csdn.net/whiteinblue/article/details/45330801)

[Creating an LMDB database in Python](http://deepdish.io/2015/04/28/creating-lmdb-in-python/)

[图像数据转换成db（leveldb/lmdb)文件](https://www.cnblogs.com/denny402/p/5082341.html)
