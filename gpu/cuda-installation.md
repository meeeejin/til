# How to install CUDA on Ubuntu

## Installation

1. Download [CUDA Toolkit](https://developer.nvidia.com/cuda-11.0-update1-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=2004&target_type=deblocal):

```bash
$ wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
$ sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/11.0.3/local_installers/cuda-repo-ubuntu2004-11-0-local_11.0.3-450.51.06-1_amd64.deb
$ sudo dpkg -i cuda-repo-ubuntu2004-11-0-local_11.0.3-450.51.06-1_amd64.deb
$ sudo apt-key add /var/cuda-repo-ubuntu2004-11-0-local/7fa2af80.pub
$ sudo apt-get update
```

2. Purge previous Nvidia packages:

```bash
sudo apt clean
sudo apt update
sudo apt-get purge nvidia-* 
sudo apt autoremove
```

3. Install CUDA Toolkit

```bash
$ sudo apt-get -y install cuda
```

4. Check the CUDA version: 

```bash
$ nvidia-smi
Fri Jun 25 17:36:29 2021
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  A100-PCIE-40GB      On   | 00000000:C5:00.0 Off |                    0 |
| N/A   37C    P0    34W / 250W |      4MiB / 40537MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
|   1  A100-PCIE-40GB      On   | 00000000:C8:00.0 Off |                    0 |
| N/A   35C    P0    33W / 250W |      4MiB / 40537MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
...
```

## Reference

- [CUDA Toolkit 11.0 Update 1 Downloads](https://developer.nvidia.com/cuda-11.0-update1-download-archive?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=2004&target_type=deblocal)