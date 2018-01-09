---
title: 'Caffe工程化:动态链接库.so文件和可执行文件的创建'
date: 2018-01-09 11:04:10
tags:
---

## 概述

Caffe中自带classification.cpp用于分类，通过传递deploy.prototxt、network.caffemodel、mean.binaryproto、labels.txt和img.jpg，可以利用训练好的模型来预测新的图片。
但是如果部署到实际中，classification.cpp是几乎一定需要修改的:
>A simple C++ code is proposed in examples/cpp_classification/classification.cpp. For the sake of simplicity, this example does not support oversampling of a single sample nor batching of multiple independent samples. This example is not trying to reach the maximum possible classification throughput on a system, but special care was given to avoid unnecessary pessimization while keeping the code readable.

将classfication.cpp根据自己的需求修改后，部分用户会需要将分类程序打包成.so动态链接库或者可执行文件。

## Caffe官方的cmake支持

Caffe官方提供了强大的cmake支持，具体内容可见[官方PR](https://github.com/BVLC/caffe/pull/1667).
这个PR较为详细，可以参考配置。下面主要是个人配置中遇到的一些问题和解决方式，可以参考。配置环境是ubuntu16.04，opencv2.4.13。

首先

    git clone git@github.com:BVLC/caffe.git.
    cd caffe && mkdir cmake_build && cd cmake_build
    cmake .. -DBUILD_SHARED_LIB=ON
* 如果已经下载有caffe,第一句命令就不需要
* 第三句命令中中的DBUILD_SHARED_LIB设置可以参考PR中的说明
* opencv可能会[报错](https://github.com/opencv/opencv/issues/6132),解决方法也在[该PR](https://github.com/opencv/opencv/issues/6132)下

>from opencv/build/ copy OpencvConfig.cmake and OpenCVModules.cmake to opencv/cmake/
then in you project delete file build and run again cmake .. and make

即为

* 将opencv/build下的OpencvConfig.cmake和OpenCVModules.cmake拷贝到opencv/cmake/
* 删除cmake_buid文件夹，再从头开始

解决该问题后再次运行cmake命令，如果输出类似下面，表明已经成功生成了MakeFile。

    -- ******************* Caffe Configuration Summary *******************
    -- General:
    --   Version           :   1.0.0
    --   Git               :   unknown
    --   System            :   Linux
    --   C++ compiler      :   /usr/bin/c++
    --   Release CXX flags :   -O3 -DNDEBUG -fPIC -Wall -Wno-sign-compare -Wno-uninitialized
    --   Debug CXX flags   :   -g -fPIC -Wall -Wno-sign-compare -Wno-uninitialized
    --   Build type        :   Release
    --
    --   BUILD_SHARED_LIBS :   ON
    --   BUILD_python      :   ON
    --   BUILD_matlab      :   OFF
    --   BUILD_docs        :   ON
    --   CPU_ONLY          :   OFF
    --   USE_OPENCV        :   ON
    --   USE_LEVELDB       :   ON
    --   USE_LMDB          :   ON
    --   USE_NCCL          :   OFF
    --   ALLOW_LMDB_NOLOCK :   OFF
    --
    -- Dependencies:
    --   BLAS              :   Yes (Atlas)
    --   Boost             :   Yes (ver. 1.58)
    --   glog              :   Yes
    --   gflags            :   Yes
    --   protobuf          :   Yes (ver. 2.6.1)
    --   lmdb              :   Yes (ver. 0.9.17)
    --   LevelDB           :   Yes (ver. 1.18)
    --   Snappy            :   Yes (ver. 1.1.3)
    --   OpenCV            :   Yes (ver. 2.4.13)
    --   CUDA              :   Yes (ver. 8.0)
    --
    -- NVIDIA CUDA:
    --   Target GPU(s)     :   Auto
    --   GPU arch(s)       :   sm_61
    --   cuDNN             :   Yes (ver. 6.0.21)
    --
    -- Python:
    --   Interpreter       :   /usr/bin/python2.7 (ver. 2.7.12)
    --   Libraries         :   /usr/lib/x86_64-linux-gnu/libpython2.7.so (ver 2.7.12)
    --   NumPy             :   /usr/lib/python2.7/dist-packages/numpy/core/include (ver 1.11.0)
    --
    -- Documentaion:
    --   Doxygen           :   No
    --   config_file       :
    --
    -- Install:
    --   Install path      :   /home/zperfet/caffe-master/.build_release/install
    --
    -- Configuring done

在确认了cmake能够在合适的地方正确找到所有需要的文件后，
运行
>make -j 12

或者像官网推荐的那样，
>cmake . -DCMAKE_BUILD_TYPE=Debug     # switch to debug
make -j 12 && make install           # installs by default to build_dir/install
cmake . -DCMAKE_BUILD_TYPE=Release   # switch to release
make -j 12 && make install           # doesn’t overwrite debug install
make symlink

我选择的是前者，在make到82%的时候报错：
> /usr/bin/ld: cannot find -lopencv_dep_cudart

解决方案1是参考[官方PR](https://github.com/BVLC/caffe/issues/5031)
具体操作如下：
>1 Once you are in the build directory
ccmake ..
This will open like a gui in the terminal
2 if you press "t" you get all the possible options
3 then I have disable CUDA_USE_STATIC_CUDA_RUNTIME (if you are on teh flag just press enter to enable On or disbale OFF)
4 press c to configure
5 press g to generate
6 make all

注意这里是ccmake而不是cmake，
>ccmake is curses (terminal handling library) interface to CMake.

一般有三个相关概念
>cmake:A command line interface (CLI).
ccmake: An ncurses (terminal) GUI. (only available on Unix-like systems)
cmake-gui: A Qt-based GUI.

包cmake-curses-gui包含了ccmake，所以可以这样[安装ccmake](https://askubuntu.com/questions/121797/how-do-i-install-ccmake)
>sudo apt-get install cmake-curses-gui

那解决方案2呢？我们可以看到这里使用ccmake也只是在make之前将CUDA_USE_STATIC_CUDA_RUNTIME从ON修改为OFF。更简单的方法是直接打开CMakeCache.txt文件，将CUDA_USE_STATIC_CUDA_RUNTIME从ON修改为OFF，再继续make即可。后面我没有碰到其他问题，成功make到100%。

## CMakeLists范例

完成上述配置后，我们就可以直接将我们的CMakeLists.txt写成类似下面的样子：

    cmake_minimum_required(VERSION 2.8.8)

    find_package(Caffe)
    include_directories(${Caffe_INCLUDE_DIRS})
    add_definitions(${Caffe_DEFINITIONS})    # ex. -DCPU_ONLY

    # generate executable file
    add_executable(application_dir classification_like.cpp)
    # generate .so file
    add_library(application_dir classification_like.cpp)
    target_link_libraries(application_dir ${Caffe_LIBRARIES})

classification_like.cpp是你修改后的分类程序，application_dir是classification_like.cpp所在的文件夹。

其中

    add_executable(caffeinated_application main.cpp)
用来生成可执行文件，

    add_library(caffeinated_application main.cpp)
用来生成.so文件，根据自己需要进行选择。

这里我们就不需要自己手动指定各种地址，cmake可以帮我们确定，使用简单的find_package(Caffe)等说明即可，极大减少了出错的可能性。

## 参考

[Classifying ImageNet: using the C++ API](http://caffe.berkeleyvision.org/gathered/examples/cpp_classification.html)

[请问如何将深度学习Caffe做成一个动态库，方便在其他应用程序中调用？](https://www.zhihu.com/question/48178994)

[Improved CMake scripts](https://github.com/BVLC/caffe/pull/1667)

[Error compiling OpenCV - CMake](https://github.com/opencv/opencv/issues/6132)

[caffe cmake build erros #5031](https://github.com/BVLC/caffe/issues/5031)

[CMake和CCMake的区别 -- cmake-curses-gui](http://blog.csdn.net/arackethis/article/details/42155589)

[How do I install ccmake?](https://askubuntu.com/questions/121797/how-do-i-install-ccmake)