---
layout: post
title:  "Transformer"
date:   2022-03-30 22:20:01 +0800
categories: [Tech]
tag: 
  - Deep Learning
  - Transformer
---

> 学习笔记

# Self Attention

## 数值运算

![]({{ '/assets/images/posts/2022-04-22-transformer/01.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/02.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/03.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/04.png' | prepend: site.baseurl}})

点积在数学中，又称数量积（dot product; scalar product; inner product） [链接](https://baike.baidu.com/item/%E7%82%B9%E7%A7%AF/9648528)

![]({{ '/assets/images/posts/2022-04-22-transformer/05.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/06.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/07.png' | prepend: site.baseurl}})

## 矩阵运算

![]({{ '/assets/images/posts/2022-04-22-transformer/08.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/09.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/10.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/11.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/12.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/13.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/14.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/15.png' | prepend: site.baseurl}})

## Positional Encoding

为 self attention 添加位置信息。每个位置都有一个唯一的位置向量 𝑒。

![]({{ '/assets/images/posts/2022-04-22-transformer/16.png' | prepend: site.baseurl}})

## seq2seq

![]({{ '/assets/images/posts/2022-04-22-transformer/17.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/18.png' | prepend: site.baseurl}})

### encoder

![]({{ '/assets/images/posts/2022-04-22-transformer/19.png' | prepend: site.baseurl}})

mutli-head attention 的输出往往加上 fully connected

![]({{ '/assets/images/posts/2022-04-22-transformer/20.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/21.png' | prepend: site.baseurl}})

### decoder

marked self attention

![]({{ '/assets/images/posts/2022-04-22-transformer/22.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/23.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/24.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/25.png' | prepend: site.baseurl}})

![]({{ '/assets/images/posts/2022-04-22-transformer/26.png' | prepend: site.baseurl}})

## train

![]({{ '/assets/images/posts/2022-04-22-transformer/27.png' | prepend: site.baseurl}})

[交叉熵 (cross entropy)](https://zhuanlan.zhihu.com/p/54066141) 

## 疑问

Word Embedding

fully connected

feed forward network

residual connection

layer normalization
