# FFMPEG新手向指南: 全是干货


## 简介
FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。

## 平台
FFmpeg可用于 Linux、Windows、macOS平台，[点击](https://www.ffmpeg.org/download.html) 即可跳转至官方下载页面。

## 使用命令
- 从 Video.mp4 中提取音频:
~~~bash
ffmpeg  -i audio1.mp4 1.mp3
~~~
- 合并音视频 （将纯视频video.mp4和音频1.mp3合并成一个音视频final.mp4）：
~~~bash
ffmpeg -i video.mp4 -i 1.mp3 -shortest final.mp4
# -shortest：在其中一个输入停止后立即停止（音频/视频）
~~~
- 删除视频中的音频:
~~~bash
ffmpeg -i video.mp4 -an videofinal.mp4
~~~
