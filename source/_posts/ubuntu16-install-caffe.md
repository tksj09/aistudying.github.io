---
layout: post
title:  "Ubuntu16安装caffe"
date:   2018-05-24 15:51:22
categories: DeepLearning
tags: ubuntu caffe cuda nvidia
author: Daniel
---

## 安装驱动
### 准备工作
* 禁用nouveau
```bash
$sudo -s
$echo "blacklist nouveau">>/etc/modprobe.d/blacklist.conf
```
* 重新生成initrd文件
```bash
$sudo mv /boot/initrd.img-$(uname -r)  /boot/initrd.img-$(uname -r)-nouveau
$sudo update-initramfs -u
```
编辑``/etc/default/grub`` 让ubuntu显示引导信息，方便以后调试维护
```bash
$sudo vi /etc/default/grub
```
将 ``GRUB_CMDLINE_LINUX_DEFAULT=``里的``quiet``去掉，``splash``改为``nosplash``
然后``sudo update-grub``
重启后可以用命令
``lsmod| grep nouveau``验证是否正确屏蔽了``nouveau``
运行``sudo /etc/init.d/lightdm stop``，然后使用``ctrl+alt+F1``进入``tty1``

### 安装驱动
这里可以选择用独立驱动（版本不低于381.09）进行安装.
独立驱动：安装NVIDIA-Linux-x86_64-381.09.run（若是集显，则加上参数--no-opengl-files)
依次按提示做出选择即可
 
## 安装CUDA
CUDA安装包内置显卡驱动，但是一般版本较低，为满足该版本CUDA的最低驱动版本，一般推荐独立安装显卡驱动
```bash
sudo ./cuda_8.0.61_linux.run
```
toolkit安装在默认路径：/usr/local/cuda-8.0
Samples安装在~/NVIDIA_CUDA-8.0_Samples/
* 添加环境变量
```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >>~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >>~/.bashrc
```
然后``source~/.bashrc``使环境变量生效

## 安装cudnn5.1
```bash
cp cudnn-8.0-linux-x64-v5.1.tgz ~
cd /usr/local
sudo tar zxvf ~/cudnn-8.0-linux-x64-v5.1.tgz
```
注：由于未知的原因，在装好以上系统部分的软件后，有可能重启后无法进入系统，表现为左上角一直有个光标，系统完全锁死。通过ssh登录后，dmesg发现内核已经崩溃，解决方法如下：
```bash
sudo dpkg-reconfigure lightdm        //重新配置lightdm
sudo update-initramfs -u             //重新生成initrd文件
sudo apt-get install --reinstall lightdm  //重新安装lightdm
```
## 安装caffe
* 手动安装caffe（官方版本，此版本对多卡支持不好，可安装NV优化过的caffe）
```bash
sudo apt-get install net-tools
sudo apt-get install python-software-properties
```
* 安装依赖
```bash
sudo apt-get install  libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev libgflags-dev libgoogle-glog-dev liblmdb-dev protobuf-compiler libatlas-base-dev python-pip python-dev python-numpy python-scipy python-skimage python-matplotlib python-h5py python-leveldb python-networkx python-pandas python-dateutil python-protobuf python-gflags python-yaml libboost-filesystem-dev libboost-thread-dev gfortran cython python-pil libboost-dev libopenblas-dev libboost-all-dev
```
* 安装PyCaffe依赖项
```bash
cd caffe
pip install -r python/requirements.txt
```
* 安装``nccl``
从 https://github.com/NVIDIA/nccl 下载nccl
```bash
unzip nccl-master.zip
cd nccl-master
sudo make PREFIX=/usr/local/cuda install
```
nccl默认安装在/usr/local/cuda/lib，可以手动复制到/usr/local/cuda/lib64
```bash
sudo cp -ar /usr/local/cuda/lib/* /usr/local/cuda/lib64
```
* 安装nvcaffe
从 https://github.com/NVIDIA/caffe 下载nv优化过的caffe后解压
```bash
unzip caffe-caffe-0.15.zip
cd caffe-caffe-0.15
cp Makefile.config.example Makefile.config
```
* 修改Makefile.config（共3处）
```
a)  #USE_CUDNN := 1修改为：USE_CUDNN := 1   
b)  #USE_NCCL := 1修改为：USE_NCCL := 1   
c)  #WITH_PYTHON_LAYER := 1修改为：WITH_PYTHON_LAYER := 1   
```
* 编译
```bash
make all -j 12
make pycaffe
```
* 最后添加PyCaffe的环境变量
```bash
vim ~/.bashrc  #打开  
export PYTHONPATH=/home/usrname/caffe/python:$PYTHONPATH   #每个人都不一样，根据caffe所在路径填写
source ~/.bashrc   #生效  
```
之后再终端上输入`python`， 然后输入`import caffe`，便可以知道是否相连接成功。如无报错，则为配置成功。

### 相关错误及解决办法
* 错误1：
``fatal error: hdf5.h: No such file or directory``
解决办法：
``sudo cp -ar /usr/include/hdf5/serial/* /usr/include``
* 错误2：
``
/usr/bin/ld: cannot find -lhdf5_hl
/usr/bin/ld: cannot find -lhdf5
``
解决办法：
``
sudo ln -s /usr/lib/x86_64-linux-gnu/libhdf5_serial_hl.so /usr/lib/libhdf5_hl.so
sudo ln -s /usr/lib/x86_64-linux-gnu/libhdf5_serial.so /usr/lib/libhdf5.so
``

## 测试caffe
切换到caffe根目录
准备测试数据：
```bash
sh data/mnist/get_mnist.sh
```
这个地方下载很慢，可以手动下载后放到data/mnist目录
```bash
cd data/mnist
wget http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz
wget http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz
wget http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz
wget http://yann.lecun.com/exdb/mnist/t10k-labels-idx1-ubyte.gz
```
然后手动运行解压命令：
```bash
gunzip *.gz
```
创建测试数据,回到caffe主目录：
```bash
cd ../../
sh examples/mnist/create_mnist.sh
```
测试：
```bash
time sh examples/mnist/train_lenet.sh
```
测试多卡：
修改``examples/mnist/train_lenet.sh``，在后边加上``--gpu 0,1,2,…7``
