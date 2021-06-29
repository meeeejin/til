# How to install TensorFlow GPU on Ubuntu

- OS: Ubuntu 20.04.1 LTS
- CUDA: v11.0
- cuDNN: v8.0.5
- Python: 3.8.8
- Tensorflow: 2.4.0
- cuDF: 0.19.2

## Miniconda Installation

```bash
$ wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
$ bash Miniconda3-latest-Linux-x86_64.sh
$ source .bashrc
```

## Python Installation

```bash
$ conda update -n base -c defaults conda
$ conda install python=3.8.8
$ conda update --all

$ conda create -n p38tf python=3.8.8
$ conda activate p38tf
```

## cuDNN Installation

1. Download [cuDNN](https://developer.nvidia.com/rdp/cudnn-archive)

2. Extract the tar package:

```bash
$ tar -xvzf cudnn-11.0-linux-x64-v8.0.5.39.tgz
```

3. Install cuDNN:

```bash
$ sudo cp cuda/include/cudnn.h /usr/local/cuda/include/
$ sudo cp cuda/ih /usr/local/cuda/include/
$ sudo chmod a+r /usr/local/cuda/include/cudnn.h
$ sudo chmod a+r /usr/local/cuda/lib64/libcudnn*
```

4. Add cuDNN to the `PATH`:

```bash
$ vi ~/.bash_profile
...
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/usr/local/cuda/include:$LD_LIBRARY_PATH

$ source ~/.bash_profile
```

## Tensorflow GPU Installation

1. Install Tensorflow GPU:

```bash
$ pip install -U tensorflow-gpu==2.4.0
```

2. Make sure GPUs are well recognized (e.g., `[PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')...]`):

```bash
$ python3
Python 3.8.10 | packaged by conda-forge | (default, May 11 2021, 07:01:05)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.config.list_physical_devices("GPU")
...
[PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU'), PhysicalDevice(name='/physical_device:GPU:1', device_type='GPU')]
```

## cuDF Installation

1. Install RAPIDS:

```bash
$ conda install -c rapidsai -c nvidia -c numba -c conda-forge cudf=0.19 python=3.8 cudatoolkit=11.0
```

2. Check it works:

```bash
$ python3
Python 3.8.10 | packaged by conda-forge | (default, May 11 2021, 07:01:05)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import cudf as cd
>>>
```

## Reference

- [Tensorflow: Tested build configurations](https://www.tensorflow.org/install/source#tested_build_configurations)
- [Miniconda: Linux installers](https://docs.conda.io/en/latest/miniconda.html#linux-installers)
- [RAPIDS](https://github.com/rapidsai/cudf/tree/main#development-setup)