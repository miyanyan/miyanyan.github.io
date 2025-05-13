---
title: Qt-ColorEditor，Qt颜色编辑器
description: QColorDialog的优化版，支持RGB和HSV等多种方式选色
date: 2024-01-30
categories:
    - qt
tags:
    - c++
    - qt
---

## 外观

分享一下我实现的颜色编辑器，主要原因是Qt的QColorDialog功能较少没法满足需求，目前已经在[zeno](https://github.com/zenustech/zeno)中使用了，由于zeno有自己的样式表，所以在zeno里长这样：

![zeno中的样式](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/zeno-style.png)

| 类名                | 描述                                   |
|---------------------|--------------------------------------|
| ColorWheel          | 颜色选择器，用于选择颜色               |
| GradientSlider       | 渐变滑块，带有渐变颜色                 |
| ColorSpinHSlider     | 带标签和旋钮的水平颜色滑块             |
| ColorButton          | 颜色按钮，显示颜色并支持拖拽和释放颜色 |
| ColorPalette         | 颜色调板，显示颜色列表并支持拖拽和释放 |
| ColorPreview         | 颜色预览，显示当前和之前选择的颜色     |
| ColorComboWidget     | 颜色组合切换器，用于切换颜色组合        |
| ColorLineEdit        | 颜色名称文本框，显示颜色名称           |
| ColorPicker          | 颜色拾取器，用于抓取屏幕颜色            |

## 功能预览（未配置样式表）

* srgb切换
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/srgb.gif)

* 颜色轮选色
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorwheel.gif)

* 颜色文字选色
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colortext.gif)

* 屏幕取色，主要实现是截取当前的屏幕然后根据鼠标的位置设置颜色，支持多个屏幕（我自己只测试了2个屏幕）
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorpicker.gif)

* 颜色滑动条选色，RGB和HSV
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorslider.gif)

* 颜色面板取色
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorpalette-select.gif)

* 颜色面板，可以把想要的颜色记录在这，持久化存储，即便关闭下一次打开也会自动加载
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorpalette-saveremove.gif)

* 上一个/当前颜色切换，这个主要是类似于PS之类的软件，可以缓存一个颜色用来备选或者撤销
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/color-precur.gif)

* 互补色取色，主要参考[color-wheel](https://www.canva.com/colors/color-wheel/)
![](https://cdn.jsdelivr.net/gh/miyanyan/miyan-image@main/img/colorcombination.gif)

## 如何使用

github地址[Qt-ColorEditor](https://github.com/miyanyan/Qt-ColorEditor)

复制 `ColorWidgets` 文件夹(只包含两个文件: `ColorEditor.h` 和 `ColorEditor.cpp`) 到你的项目，记得添加到构建系统如cmake中.

接口参照QColorDialog的方式：

```c++
#include "ColorWidgets/ColorEditor.h"
// ...
// call here, you can find this in MainWindow.cpp
auto btn = new ColorButton(this);
btn->setColor(Qt::blue);
setCentralWidget(btn);

connect(btn, &ColorButton::clicked, this, [this, btn](){
    auto color = ColorEditor::getColor(btn->color(), this, "");
    btn->setColor(color);
});
```

最后还得说一句，QColorDialog的功能确实有点少了，我在实现的时候参考了很多3D软件，如Houdini、Blender、Unity等，其中Houdini的功能最多，因此最终的形态也和Houdini类似了。
