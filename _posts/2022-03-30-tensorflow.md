---
layout: post
title:  "Tensorflow"
date:   2022-03-30 22:20:01 +0800
categories: [Tech]
tag: 
  - Tensorflow
---

# Tensorflow 错误处理

## 第一中错误

```
Could not load dynamic library 'libcudart.so.11.0'; dlerror: libcudart.so.11.0: cannot open shared object file: No such file or directory
```

TensorFlow-GPU 与 CUDA cudnn Python 版本关系：[链接](https://tensorflow.google.cn/install/source_windows?hl=en#gpu)

[CUDA 安装](https://developer.nvidia.com/cuda-toolkit-archive)
[解决办法](https://blog.csdn.net/qq_44703886/article/details/112393149)