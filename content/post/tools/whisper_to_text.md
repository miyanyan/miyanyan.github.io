---
title: 【程序员的工具】怎么提取mp3文件中的音频为文字
description: whisper的使用
date: 2025-10-14
categories:
    - tools
tags:
    - tools
    - python
    - uv
    - whisper
---

## 前言

最近手头有几个音频文件，想要提取出来，但是网上一搜信息过于繁杂，这时我突然想起来github上我start了whisper这个项目，当时只是看了一下介绍，觉得很有意思，再加上是openai出品，就star了，同时还发现一个whisper.cpp的项目，当然都没有怎么仔细看，只是知道这个项目可以提取mp3文件中的音频为文字。欸！今天就用上了~

## 安装whisper

* 首先安装ffmpeg，并配置好环境变量
* 这里我们使用uv来安装whisper，假设你已经熟悉uv的用法了

    ```
    uv tool install openai-whisper
    ```

## 使用whisper

* 打开命令行，直接输入`whisper xxx.mp3`即可，xxx.mp3是你要提取音频的mp3文件名，程序会自动提取出音频中的文字并打印出来。
* 这里我们什么都不管，直接就敲这两个字符，直接出结果（第一次会先下载模型文件），除了控制台的输出，还会生成一个xxx.txt文件，里面就是提取出的文字。
* 详细用法可以看[官方文档](https://github.com/openai/whisper)

## 总结

不得不感叹程序员就是好呀，有这么多的开源工具可以使用，敲一点点命令就能出结果，`simple is better than complex!`。


