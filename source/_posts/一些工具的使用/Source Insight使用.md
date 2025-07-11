---
title: Source Insight使用
categories:
  - - 一些工具的使用
tags: 
date: 2024/5/6
updated: 
comments: 
published:
---
# 快速上手

1. 左上角，Project，New Project，新建一个工程。出现以下界面：
   ![](../../images/Source%20Insight使用/image-20240506213527152.png)
2. Source Insight的工程文件夹和源码根文件夹需要并列在一起。可以在源码根文件夹之外创建一个"Insight"（名字自定）的文件夹。
   ![](../../images/Source%20Insight使用/image-20240506213827928.png)
3. 将Insight的目录地址拷贝到第1步的第2个空内。点击OK。
   ![](../../images/Source%20Insight使用/image-20240506214358083.png)
4. 出现New Project Settings窗口，默认不改变设置。继续OK。
   ![](../../images/Source%20Insight使用/image-20240506214516331.png)
5. 出现Add and Remove Project Files窗口，点击源码根文件夹，再点击右侧边的Add Tree按钮。添加完成后，Close此窗口即可。
   ![](../../images/Source%20Insight使用/image-20240506214715147.png)
6. 添加完工程文件后。接着，左上角，Project，Synchronize Files。同步以后，在Source Insight中编辑时可以同时同步磁盘上的文件。（不同步其实也可以，麻烦之处就是如果稍微改动一下代码它就总会提示是否保存）
   ![](../../images/Source%20Insight使用/image-20240506215047348.png)
7. 同步完后，即可点击工具栏中的圆圈带P的按钮，打开菜单，查看文件。
   ![](../../images/Source%20Insight使用/image-20240506215346027.png)
# 配置

如果文件后缀名不是常规的，比如Cpp的`.cpp`扩展名写作了`.cc`，那么就需要人为配置。
1. 左上方，Options，File Type Options
   ![](../../images/Source%20Insight使用/image-20240506215618397.png)
2. 左侧栏选择某一语言的文件类型，在右侧最上方的File Filter中详细加规则，如图：
   ![](../../images/Source%20Insight使用/image-20240506215938879.png)
3. 配置完后，需要再次Add Tree，需要点击Project，Add and Remove Project Files
   ![](../../images/Source%20Insight使用/image-20240506220109756.png)
# 快捷键配置

Options，Key Assignments。可以配置代码高亮的快捷键。
![](../../images/Source%20Insight使用/image-20240506224734346.png)

