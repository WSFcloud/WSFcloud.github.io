---
title: 让WSL中的Docker使用GPU
published: 2024-11-28
description: '仅使用WSL，不额外安装Docker Desktop即可使用宿主机GPU'
image: ''
tags: [Docker, WSL, Deep Learning]
category: '笔记'
draft: false 
lang: 'zh_CN'
---

## 前言
参考[WSL官方文档](https://learn.microsoft.com/zh-cn/windows/wsl/tutorials/gpu-compute)与[NVIDIA官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/index.html)进行配置[^1]。

使用环境：
- Windows：Windows10 22H2
- WSL: WSL2-2.3.26.0
- Linux：Debian GNU/Linux 12 (bookworm)

[^1]: 截止至2024-11-28，WSL文档里的部分内容与实际运行结果不一致。

## WSL中安装Docker Engine
1. 在宿主机中安装NVIDIA GPU驱动。
   
2. 在WSL中安装Docker Engine

```bash
curl https://get.docker.com | sh
```

重启Docker服务
```bash
sudo service docker start
```


## 安装NVIDIA Container Toolkit
1. 配置APT源
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

（可选）使用实验性软件包
```bash
sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

2. 安装NVIDIA Container Toolkit
```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```


## 配置Docker
1. 配置容器运行时nvidia-ctk
```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

2. 重新启动Docker守护进程
```bash
sudo systemctl restart docker
```


## 检测Docker是否可以使用GPU
使用debian: latest作为容器镜像，加入参数“runtime=nvidia”与“gpus all”启动容器：
```bash
sudo docker run --rm --runtime=nvidia --gpus all debian:latest nvidia-smi
```

看到类似输出：
```bash
Thu Nov 28 10:02:57 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 565.72                 Driver Version: 566.14         CUDA Version: 12.7     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 2060        On  |   00000000:01:00.0  On |                  N/A |
| N/A   35C    P8              7W /  115W |    1012MiB /   6144MiB |      4%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

现在Docker已经可以使用宿主机的GPU了。