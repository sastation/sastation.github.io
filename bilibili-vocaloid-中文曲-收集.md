---
title: Bilibili - Vocaloid 中文曲收集、下载、处理
date: 2017-05-10 10:35
cdn: header-on
header-img: title01.jpg

---

# Bilibili - Vocaloid 中文曲收集、下载、处理

## 获得VC中文曲相关数据

- 以“VOCALOID中文曲”标签进行搜索
- 按播放数进行排序
- 取前50页的内容（bilibili的搜索最多只有50页，2017-05-10）
- 分别取得：AV号、上传日期、播放数、弹幕数、曲名
- 输出到文件：“vc.list-\<yymmdd\>”

## 数据合并去重

- 将所有收集的VC数据合并
- 重复数据只取重新一条
- 数据文件从最近的一次开始向前处理
- 输出到文件："ranking.list"

## 整理数据以供下载

- 以合并去重后数据为基准
- 选出播放数超过10万的记录，只记录最早一次
- 分别取得：当前日期、AV号，上传日期、曲名
- 输出到文件：“palace.list”

## 下载视频及弹幕

- 以“palace.list”数据为基准
- 通过使用 you-get.py 工具进行下载：```git clone https://github.com/soimort/you-get.git```
- 例子：```you-get -o video av1234567 > /dev/null 2>&1```
- 判断返回值：0-good-成功、other-wrong-失败
- 将信息存入文件“download.list”：av号、[good|wrong]

## 处理弹幕

- 以“you-get”下载的xml弹幕文件为基础
- 通过使用“danmaku2ass”处理：```git clone https://github.com/m13253/danmaku2ass.git```
- 例子：```./danmaku2ass.py -o "sample.ass" -s 1920x1080 -fn "MS PGothic" -fs 46 -a 0.8 -dm 8 -ds 6 "sample.cmt.xml"```

## 视频转换、合并、提取

- 使用 ffmpeg 转换 flv 到 mp4：```ffmpeg -i out.flv -vcodec copy -acodec copy out.mp4```
- 使用 ffmpeg 合并字幕：```ffmpeg -i input.mp4 -vf ass=subtitle.ass output.mp4```
- 使用 ffmpeg 提取音频：```ffmpeg -i apple.mp4 -f mp3 -vn apple.mp3```


