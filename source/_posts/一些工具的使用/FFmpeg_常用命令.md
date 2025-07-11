---
title: FFmpeg_常用命令
categories:
  - - 一些工具的使用
tags: 
date: 2022/8/28
updated: 
comments: 
published:
---
# 内容

* 常用参数选项
* 抽取视频的音频，保存为单独的文件

# 常用参数选项

| 选项                       | 描述                 |
| -------------------------- | -------------------- |
| `-i`                       | 输入的文件           |
| `-vn`                      | disable video        |
| `-b:v`                     | 视频码率             |
| `-b:a`                     | 音频码率             |
| `-acodec`/`-vcodec` `copy` | 代表复用原音视频格式 |

# 抽取音频

https://superuser.com/questions/332347/how-can-i-convert-mp4-video-to-mp3-audio-with-ffmpeg

The basic command is:

```
ffmpeg -i filename.mp4 filename.mp3
```

or

```
ffmpeg -i video.mp4 -b:a 192K -vn music.mp3
```

Check this URL: [MP4 Video to MP3 File Using ffmpeg (Ubuntu 9.10 Karmic Koala)](http://donnieknows.com/blog/mp4-video-mp3-file-using-ffmpeg-ubuntu-910-karmic-koala) link broken [Updated on 7th Dec 2021]

**Note:** Ubuntu does not supply FFmpeg, but the fork named [Libav](http://libav.org/). The syntax is the same – just use `avconv` instead of `ffmpeg` for the above examples.

---

https://stackoverflow.com/questions/9913032/how-can-i-extract-audio-from-video-with-ffmpeg

To extract the audio stream without re-encoding:

```
ffmpeg -i input-video.avi -vn -acodec copy output-audio.aac
```

- `-vn` is no video.
- `-acodec copy` says use the same audio stream that's already in there.

Read the output to see what codec it is, to set the right filename extension.

# 按时间范围截取

https://blog.csdn.net/u011573853/article/details/103221606

* 截取一段音频

  ```
  ffmpeg -i lesson.mp3 -ss 6:17 -to 8:25 -c copy lessson4.mp3
  ```

* 把音频加入从头加入到视频
  ```
  ffmpeg -i lessson4.mp3 -i video.mp4 -map 0:0 -map 1:1 -c copy movie.mp4
  ```

* 截取一段视频
  ```
  ffmpeg -i '1.mp4' -ss 00:01:17 -t 00:10:45 -vcodec copy -acodec copy yundong.mp4
  ffmpeg -ss 00:00:10 -i input.mp4 -c copy output.mp4 # 减去前面10秒
  ```

# 常用操作命令总结

https://www.jianshu.com/p/341e0673bfe8

## 截图

```bash
ffmpeg -ss 00:43:55 -i video.mp4  -f image2  -vframes 1 -y frame.png
```

注意将`ss`放到最前面可以加快速度, `-y`代表覆盖文件 `-vframes`代表帧数 `-i`代表输入，即in；`-ss`也可以使用单个数字，代表秒数，从0开始计算。

## 去固定水印

```bash
ffmpeg -i video.mp4 -vf "delogo=x=1680:y=60:w=160:h=55"  -y  new_1.mp4
```

这里`-vf`表示`video filter`， 其中`delogo`的参数代表水印的坐标和大小，把视频左上角作为坐标原点，横向为x轴，纵向为y轴。这种情况除非预先知道水印的位置和大小，否则不是特别方便，当然，准确识别水印位置也是一个难点，不是很轻易能实现的。
 可能根据某些`ffmpeg`版本不同，需要加`-strict experimental` 参数，一种情况是比较老的版本音频`ACC`属于实验阶段，可以按情况设置或者升级`ffmpeg`版本。

## 获取视频时长

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 new_out.mp4
```

ffprobe 是ffmpeg 配套的一个工具，可以查看一些内容信息。上面的命令可以直接获得秒为单位的视频时长

1. 获取视频信息并优化展示
    `ffprobe -v quiet -print_format json -show_format -show_streams pianpian.mp4`
    这里`-v`代表日志级别，可以使用`debug`用来分析某些异常；  上面的命令会以json格式输出`format`和`stream`两项的信息.
2. 转码再生成m3u8
    先转为ts格式:

```bash
ffmpeg -y -i video.mp4 -c:v copy -c:a copy -vbsf h264_mp4toannexb output.ts
```

这里 `-c:v`, `-c:a`分别代表视频，音频格式，`copy`代表原视频格式， `-vbsf`或者`-bsf:v`(`-bsf:a`)，表示`bitstream filter`，转码格式。
 转换m3u8：

```bash
ffmpeg -i output.ts  -c copy -map 0 -f segment -segment_list index.m3u8 -segment_time 10 video_sgs/video-%03d.ts
```

中间参数没有太了解，功能是将视频分段并生成m3u8文件， 包括设置分段视频的长度。

## 视频分帧

```bash
ffmpeg -i src01.avi %d.jpg
```

将视频所有帧保存为图片。 注意整体内容可能比较大，实验中19MB的1280*720视频，分帧后的图片有3.8G。

## 视频加文字水印

```bash
ffmpeg -i input.mp4 -vf "drawtext=fontfile=simhei.ttf:text='雷':x=100:y=10:fontsize=24:fontcolor=yellow:shadowy=2" drawtext.mp4
```

可以给水印设置字体，大小，颜色等。 字体颜色可以用RGB代码，比如`fontcolor=#FFFF00`，如果要设置透明度可以这样写：`fontcolor=#FFFFFF@0.6`，表示0.6的透明度，取值为`0.1-1.0`。
 `shadowy`表示阴影。

> 注意： 这里冒号`:`是关键字，如果是要加到水印里，需要转义，用四个\下面是一个例子

```bash
ffmpeg -i source.mp4 -vf "drawtext=fontfile=MicroYaHei.ttf:text=By\\\\:三峡不好人:x=1240:y=44:fontsize=73:fontcolor=#FFFFFF@0.8"  -y drawtext_out.mp4
```

## ffmpeg限制cpu数

 ffmpeg在去水印，加水印的时候，默认都是占满可用CPU的，某些情况下需要限制CPU数。网上文章乱七八糟，各种抄，很多说用`-threads` 参数，但说的不明不白。 以下亲测，`-threads`参数放到 -y 前面是可以生效的， Linux 可以用`top -H -p <pid>` 看运行线程数来验证， 同时可以用`uptime`比较限制线程和不限制的CPU使用率。 如下是限制为2个线程。

```bash
ffmpeg -i source.mp4 -vf "drawtext=fontfile=MicroYaHei.ttf:text=By\\\\:三峡不好人:x=1240:y=44:fontsize=73:fontcolor=#FFFFFF@0.8"  -threads 2  -y drawtext_out.mp4
```
# 将多个TS文件批量转换为MP4格式
批量转换脚本
- ​**Windows（批处理脚本）​**：    
    1. 新建文本文件，输入以下内容：        
        ```bash
        @echo off
        for %%i in (*.ts) do (
            ffmpeg -i "%%i" -c copy "%%~ni.mp4"
        )
        pause
        ```
    2. 保存为 `convert.bat`，放到TS文件所在文件夹，双击运行。
- ​**Linux/macOS（Shell脚本）​**：
    1. 新建脚本文件 `convert.sh`：
        ```bash
        #!/bin/bash
        for file in *.ts; do
            ffmpeg -i "$file" -c copy "${file%.ts}.mp4"
        done
        ```
    2. 添加执行权限并运行：
        ```bash
        chmod +x convert.sh
        ./convert.sh
        ```
## Windows
在Windows批处理脚本中，`"%%~ni.mp4"` 是一个特殊的 ​**变量扩展语法**，用于处理文件名。具体含义分解如下：

- 在批处理脚本中必须使用 ​**双百分号**​（`%%i`），而在命令行直接执行时用单百分号（`%i`）。

| 符号          | 含义                                 |
| ----------- | ---------------------------------- |
| `%%i`       | 循环变量，表示当前处理的文件名（如 `video1.ts`）     |
| `%~ni`      | 去掉文件名的扩展名（`.ts`），只保留主文件名（`video1`） |
| `.mp4`      | 添加新的扩展名                            |
| `%%~ni.mp4` | 组合后的完整新文件名（`video1.mp4`）           |
其他常见变量扩展

|语法|作用|示例（原文件名：`D:\Videos\test.ts`）|
|---|---|---|
|`%%~i`|完整路径+文件名|`D:\Videos\test.ts`|
|`%%~di`|驱动器盘符|`D:`|
|`%%~pi`|文件路径（不含文件名）|`\Videos\`|
|`%%~ni`|主文件名（不含扩展名）|`test`|
|`%%~xi`|扩展名（包含点号）|`.ts`|
|`%%~fi`|完整绝对路径|`D:\Videos\test.ts`|
- `-c copy`：直接复制音视频流，无需重新编码（速度快）。
- 如需重新编码（兼容性更好，但速度慢）：

```bash
ffmpeg -i input.ts -c:v libx264 -c:a aac output.mp4
```