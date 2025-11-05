---
title: 【程序员的工具】在手机上使用Termux和Claude Code
description: 手机上的编程环境配置指南
date: 2025-11-05
categories:
    - tools
tags:
    - tools
    - termux
    - claude-code
    - mobile
---

## 前言

作为一名程序员，我们经常需要随时随地处理代码或者进行开发工作。虽然笔记本电脑便携性不错，但有时候我们可能只想用手机来处理一些简单的编程任务。今天我要介绍如何在Android手机上通过Termux配置一个完整的Claude Code开发环境，让你真正做到"手机就是开发机"！

## 什么是Termux

Termux是一个Android终端模拟器和Linux环境应用程序，它可以直接在手机上运行许多Linux命令行工具，无需root权限。通过Termux，我们可以在手机上拥有一个功能强大的命令行环境。

## 安装Termux

首先需要安装Termux应用。由于Google Play上的Termux版本比较老旧，推荐从GitHub下载最新版本：

1. 访问 [Termux GitHub Releases](https://github.com/termux/termux-app/releases)
2. 下载适合你手机架构的APK文件，通常选择 `termux-app_v0.118.3+github-debug_arm64-v8a.apk`
3. 在手机上安装该APK文件

## 配置Termux环境

安装完成后，打开Termux应用，首先更新软件包列表和已安装的软件包：

```bash
pkg update && pkg upgrade
```

然后安装必要的开发工具：

```bash
pkg install nodejs
pkg install git
```

## 安装Claude Code

Claude Code是Anthropic官方提供的命令行工具，可以让我们直接在终端中与Claude进行交互。通过npm安装：

```bash
npm install -g @anthropic-ai/claude-code
```

## 配置API密钥

要使用Claude Code，需要配置API密钥。如果你有Anthropic的API密钥，可以直接使用。如果没有，可以使用第三方API服务, 这里以glm为例：

```bash
export ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic
export ANTHROPIC_AUTH_TOKEN=YOUR_API_KEY
```

将 `YOUR_API_KEY` 替换为你的实际API密钥。

为了方便使用，可以将这些环境变量添加到shell配置文件中：

```bash
echo 'export ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic' >> ~/.bashrc
echo 'export ANTHROPIC_AUTH_TOKEN=YOUR_API_KEY' >> ~/.bashrc
source ~/.bashrc
```

## 使用Claude Code

配置完成后，就可以直接使用Claude Code了：

```bash
claude
```

如果配置正确，你应该能看到Claude Code的启动界面。
