---
title : "vscode配置教程"
date : 2026-04-19T13:43:16+08:00
author : "icey"
categories : ["教程"]
tag : ["环境配置"]
draft : false
summary : "关于vscode的配置教程"
---


- ## 前言
    - 作为电子信息的学生,在大一上时我就已经了解到vscode是很棒的IDE,但在什么都不懂的时候面对环境变量，还有终端命令，.vscode下的json文件等烦人的配置实属劝退
    - 在大一下翻阅了网上教程后也是艰难的用上了vscode并体会到了它的好处。在搭建个人博客时想到有不少的人也会遇到同样的问题，故写下此博客(还没写完,但可按这个文章顺序来做)



1. - ## 准备工作
        按下键盘win[^win键]+R输入cmd，弹出的命令提示符中输入whoami

        [^win键]: 四个小正方形的按键，对应微软图标
        ***
        ![win&R](/vscode_enviroment/win&R.png)
        ![whoami](/vscode_enviroment/whoami.png)

        ***

        在示例中cmd返回了icey\windows,其中icey代表的是计算机名,windows代表的是用户名。其中\后的内容即用户名很重要,请确保它是***全英***，若含有中文可能会在未来带来一系列的环境问题

        你也可以在C:/Users下确认你的用户名是否含有中文

        如果你的用户名含有中文,最简单的解决方法是是重新创建一个用户,这里不展开了

2. - ## 软件下载
        - vscode下载  在官网下载
        - mingw下载以及环境变量的配置  [知乎的mingw下载详细解释](https://zhuanlan.zhihu.com/p/26143367916)

3. - ## vscode插件的配置
        - 基础配置 
        - c/c++
        - git
        - 其它 