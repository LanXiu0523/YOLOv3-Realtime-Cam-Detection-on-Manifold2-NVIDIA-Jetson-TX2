# YOLOv3-Realtime-Cam-Detection-on-Manifold2-NVIDIA-Jetson-TX2
## 0 使用须知

1. 系统和软件版本最好和我严格对应，至少大版本应该一样
2. 推荐安装`jtop`，可以实时监控CPU、GPU、内存、磁盘等系统状态和系统软件版本，但不是必须，故未列出安装方法。
3. 在编译过程中，尽量启动风扇提高编译速度，中途风扇停止可再次启动。如果觉得很吵可以更改风扇速度（下文有说）。
4. 如果`apt-get install`不顺利，可以尝试使用`aptitude`软件包管理工具。
5. 本项目采用源码编译安装`python3.6.13`，也可以采用别的方式安装，只要版本一样即可，至少大版本应该一样。
5. 安装过程中遇到问题请先查看在文末提到的可能遇到的问题。



## 1 查看系统和软件版本

```bash
# 查看系统内核
uname -a 
# 查看系统版本
cat /etc/lsb-release
# 查看jetson TX2 L4T版本
head -n 1 /etc/nv_tegra_release
# 查看cmake版本
cmake -version
# 查看cuda版本
nvcc --version
# 查看cudnn版本
cat /usr/include/cudnn.h | grep CUDNN_MAJOR -A 2
# 查看opencv版本
pkg-config --modversion opencv
```

My logs:
```bash
dji@manifold2:~$ uname -a
Linux manifold2 4.4.38+ #2 SMP PREEMPT Mon Jun 3 20:19:02 CST 2019 aarch64 aarch64 aarch64 GNU/Linux
dji@manifold2:~$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.7 LTS"
dji@manifold2:~$ head -n 1 /etc/nv_tegra_release
# R28 (release), REVISION: 2.1, GCID: 11272647, BOARD: t186ref, EABI: aarch64, DATE: Thu May 17 07:29:06 UTC 2018
dji@manifold2:~$ cmake -version
cmake version 3.21.4

CMake suite maintained and supported by Kitware (kitware.com/cmake).
dji@manifold2:~$ nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2017 NVIDIA Corporation
Built on Sun_Nov_19_03:16:56_CST_2017
Cuda compilation tools, release 9.0, V9.0.252
dji@manifold2:~$ cat /usr/include/cudnn.h | grep CUDNN_MAJOR -A 2
#define CUDNN_MAJOR 7
#define CUDNN_MINOR 1
#define CUDNN_PATCHLEVEL 5
--
#define CUDNN_VERSION    (CUDNN_MAJOR * 1000 + CUDNN_MINOR * 100 + CUDNN_PATCHLEVEL)

#include "driver_types.h"
dji@manifold2:~$ pkg-config --modversion opencv
3.3.1
```



## 2 风扇设置

```bash
# 启动风扇（默认最大功率）
sudo /home/dji/jetson_clocks.sh

# 更改风扇速度（功率）
vim /home/dji/jetson_clocks.sh
#   > etson_clocks.sh: Modify FAN_SPEED=255 to FAN_SPEED=127 on line 113
# 注意FAN_SPEED的值应为0～255
```



## 3.配置apt依赖环境

### 3.1 换源（非必需）
```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak  #备份源文件    
sudo vim /etc/apt/sources.list  
```

修改为：

```bash
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main restricted universe multiverse 
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main restricted universe multiverse 
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main restricted universe multiverse 
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main restricted universe multiverse 
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main restricted universe multiverse 
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main restricted universe multiverse 
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main universe restricted 
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main universe restricted
```

### 3.2 安装apt依赖包

```bash
sudo apt-get update

sudo apt-get install gcc g++ build-essential checkinstall libatlas-dev libc6-dev libffi-dev libfreetype6-dev libjpeg-dev liblapack-dev libopenblas-dev libsqlite3-dev libssl-dev libxft-dev openssl python-dev python3-dev python3.6-dev python-pip python3-pip python-setuptools python3-sklearn tcl tk tk-dev zlib1g-dev 
```



## 4.源码编译安装Python 3.6.13

```bash
wget https://www.python.org/ftp/python/3.6.13/Python-3.6.13.tgz
tar -zxvf Python-3.6.13.tgz

./configure --prefix=/usr/local/python3
sudo make && sudo make install

# 设置软连接
sudo rm /usr/bin/python3
sudo ln -s /usr/local/python3/bin/python3.6 /usr/bin/python3
sudo rm /usr/bin/pip3
sudo ln -s /usr/local/python3/bin/pip3.6 /usr/bin/pip3

# 查看python3版本，应为Python 3.6.13
sudo python3 -V
```



## 5 源码编译安装Pytorch v1.0.0

### 5.1 安装pip依赖

```bash
sudo python3 -m pip install --upgrade pip
sudo python3 -m pip install numpy==1.19.4 scipy pyyaml scikit-build cffi ninja
```

### 5.2  开辟Swap空间

```bash
free -h
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon -show
free -h
# 此时Swap空间total应为2G
```

### 5.3 源码编译安装pytorch

```bash
# 启动风扇
sudo /home/dji/jetson_clocks.sh

# 建议用我给的python源码包，从官方下载的需要修改一些地方，包括仅用NCCL、CUDA，修改load函数等
cd /YOLOv3-Realtime-Cam-Detection-on-Manifold2-NVIDIA-Jetson-TX2
tar -zxvf pytorch_v1.0.0_for_Manifold2.tar.gz 
cd pytorch
git submodule update --init --recursive

sudo python3 -m pip install -U setuptools
sudo python3 -m pip install -r requirements.txt
sudo python3 -m pip install scikit-build --user

sudo ldconfig
export USE_NCCL=0 USE_DISTRIBUTED=1 USE_OPENCV=ON USE_CUDNN=1 USE_CUDA=1 ONNX_ML=1

# 接下来两段编译用时较长，尽量保证风扇启动以加速编译
sudo -E USE_MKLDNN=0 USE_QNNPACK=0 USE_NNPACK=0 USE_DISTRIBUTED=0 BUILD_TEST=0 python3 setup.py bdist_wheel

sudo -E USE_MKLDNN=0 USE_QNNPACK=0 USE_NNPACK=0 USE_DISTRIBUTED=0 BUILD_TEST=0 DEBUG=1 python3 setup.py build develop

sudo apt clean
```

### 5.4 检查Pytorch是否成功安装并能识别到GPU

```bash
sudo python3
>>> import torch
>>> torch.__version__
>>> torch.cuda.is_available()
>>> torch.cuda.device_count()
>>> torch.cuda.get_device_name(torch.cuda.current_device())
```

My logs:

```bash
dji@manifold2:~$ sudo python3
Python 3.6.13 (default, Feb 20 2021, 21:42:50) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import torch
>>> torch.__version__
'1.0.0a0+db5d313'
>>> torch.cuda.is_available()
True
>>> torch.cuda.device_count()
1
>>> torch.cuda.get_device_name(torch.cuda.current_device())
'NVIDIA Tegra X2'
```



## 6 运行YOLOv3 实现相机实时目标检测

将免驱动的摄像头连接到Manifold2开发版上

```bash
cd /YOLOv3-Realtime-Cam-Detection-on-Manifold2-NVIDIA-Jetson-TX2/yolov3
# 接下来bulid whell for opencv用时也会较长，尽量保证风扇启动以加速bulid
sudo python3 -m pip install -r requirements.txt

# 运行测试demo
sudo python3 detect.py 

# 改变参数运行detect任务(可选)
sudo python3 detect.py --cfg=$CFG --weights=$WEIGHTS --img-size=$IMG_SIZE
```





## 7 可能遇到的问题

#### 7.1 Python报错：subprocess.CalledProcessError: Command '('lsb_release', '-a')' returned non-zero exit status 1.

报错信息：

```bash
Exception:
Traceback (most recent call last):
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/cli/base_command.py", line 143, in main
    status = self.run(options, args)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/commands/install.py", line 259, in run
    with self._build_session(options) as session:
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/cli/base_command.py", line 79, in _build_session
    insecure_hosts=options.trusted_hosts,
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/download.py", line 337, in __init__
    self.headers["User-Agent"] = user_agent()
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/download.py", line 100, in user_agent
    zip(["name", "version", "id"], distro.linux_distribution()),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 120, in linux_distribution
    return _distro.linux_distribution(full_distribution_name)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 675, in linux_distribution
    self.version(),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 735, in version
    self.lsb_release_attr('release'),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 892, in lsb_release_attr
    return self._lsb_release_info.get(attribute, '')
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 550, in __get__
    ret = obj.__dict__[self._fname] = self._f(obj)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 998, in _lsb_release_info
    stdout = subprocess.check_output(cmd, stderr=devnull)
  File "/usr/local/python3/lib/python3.6/subprocess.py", line 356, in check_output
    **kwargs).stdout
  File "/usr/local/python3/lib/python3.6/subprocess.py", line 438, in run
    output=stdout, stderr=stderr)
subprocess.CalledProcessError: Command '('lsb_release', '-a')' returned non-zero exit status 1.
Traceback (most recent call last):
  File "/usr/local/python3/lib/python3.6/runpy.py", line 193, in _run_module_as_main
    "__main__", mod_spec)
  File "/usr/local/python3/lib/python3.6/runpy.py", line 85, in _run_code
    exec(code, run_globals)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/__main__.py", line 19, in <module>
    sys.exit(_main())
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/__init__.py", line 78, in main
    return command.main(cmd_args)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/cli/base_command.py", line 184, in main
    timeout=min(5, options.timeout)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/cli/base_command.py", line 79, in _build_session
    insecure_hosts=options.trusted_hosts,
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/download.py", line 337, in __init__
    self.headers["User-Agent"] = user_agent()
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_internal/download.py", line 100, in user_agent
    zip(["name", "version", "id"], distro.linux_distribution()),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 120, in linux_distribution
    return _distro.linux_distribution(full_distribution_name)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 675, in linux_distribution
    self.version(),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 735, in version
    self.lsb_release_attr('release'),
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 892, in lsb_release_attr
    return self._lsb_release_info.get(attribute, '')
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 550, in __get__
    ret = obj.__dict__[self._fname] = self._f(obj)
  File "/usr/local/python3/lib/python3.6/site-packages/pip/_vendor/distro.py", line 998, in _lsb_release_info
    stdout = subprocess.check_output(cmd, stderr=devnull)
  File "/usr/local/python3/lib/python3.6/subprocess.py", line 356, in check_output
    **kwargs).stdout
  File "/usr/local/python3/lib/python3.6/subprocess.py", line 438, in run
    output=stdout, stderr=stderr)
subprocess.CalledProcessError: Command '('lsb_release', '-a')' returned non-zero exit status 1.
```

解决方案：

```bash
# 删除lsb_release
rm -rf /usr/bin/lsb_release
```



#### 7.2 报错：error adding symbols: File in wrong format

报错信息：

```bash
[ 64%] Linking CXX shared library ../lib/libcaffe2_gpu.so
/usr/local/cuda/lib64/libcudnn.so.7: error adding symbols: File in wrong format
collect2: error: ld returned 1 exit status
caffe2/CMakeFiles/caffe2_gpu.dir/build.make:197603: recipe for target 'lib/libcaffe2_gpu.so' failed
make[2]: *** [lib/libcaffe2_gpu.so] Error 1
CMakeFiles/Makefile2:1523: recipe for target 'caffe2/CMakeFiles/caffe2_gpu.dir/all' failed
make[1]: *** [caffe2/CMakeFiles/caffe2_gpu.dir/all] Error 2
make[1]: *** Waiting for unfinished jobs....
[ 64%] Built target python_copy_files
Makefile:138: recipe for target 'all' failed
make: *** [all] Error 2
Failed to run 'bash ../tools/build_pytorch_libs.sh --use-cuda --use-nnpack --use-qnnpack caffe2'
```
解决方案：

```bash
# 从报错中得知路径 `/usr/local/cuda/lib64` 下 `File in wrong format`
cd /usr/local/cuda/lib64/sudo 
file * | grep x86-64

# 发现有x64格式文件，我们的系统是arm64的
sudo mkdir x86-64_files

# 将刚才打印的x64格式文件移动到文件夹 `x86-64_files`
sudo mv libcudnn.so.7 libcudnn.so.7.1.1 libcudnn.so.7.4.2  x86-64_files/
```

