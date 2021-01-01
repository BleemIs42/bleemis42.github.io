---
title: "解析视频的元数据"
date: 2021-01-01T23:02:54+08:00
cover: "/img/hanzhong.jpg"
categories: ["JavaScript"]
tags: ["JavaScript", "image", "video"]
---

<video controls autoplay src="/media/hanzhong.mp4"></video>

### 视频的元数据
> [ISO/IEC 14496-12:2005(E) - MP4](https://web.archive.org/web/20180219054429/http://l.web.umkc.edu/lizhu/teaching/2016sp.video-communication/ref/mp4.pdf)     
> [ISO/IEC 14496-1 Media Format - MP4](http://xhelmboyx.tripod.com/formats/mp4-layout.txt)
> [参考](https://www.jianshu.com/p/529c3729f357)

Mp4 或称 MPEG-4 Part 14，是一种多媒体容器格式，扩展名为 `.mp4`.    

Mp4 文件由一系列的 box 构成, 每个 box 包含 box 头部和 box 体. box 体可以包含普通的数据, 也可以包含其他的 box, 如果 box 中包含了另一个 box, 这种box 称为 container box.    

如下图:    
<img width="400" src="/img/media-format.png">

了解这个是因为工作中上传视频, 需要拿到视频的播放时长, 刚开始使用 `FileReader` 读取 `input` 的文件, 放到 `video` 的 `src` 去 `preload="metadata"`, 然后通过 `video element` 拿到 `duration`, 小视频没问题, 但是当上传的视频比较大的时候, `FileReader` 二次读取并加载到内存, 导致内存爆掉了, 于是决定去解析视频的元数据, 拿到 `duration`.

视频的元数据编码在上图中的 `mvhd` box 内, 编码规则如下:
```
   * 8+ bytes movie (presentation) header box
       = long unsigned offset + long ASCII text string 'mvhd'
     -> 1 byte version = 8-bit unsigned value
       - if version is 1 then date and duration values are 8 bytes in length
     -> 3 bytes flags =  24-bit hex flags (current = 0)

     -> 4 bytes created mac UTC date
         = long unsigned value in seconds since beginning 1904 to 2040
     -> 4 bytes modified mac UTC date
         = long unsigned value in seconds since beginning 1904 to 2040
     OR
     -> 8 bytes created mac UTC date
         = 64-bit unsigned value in seconds since beginning 1904
     -> 8 bytes modified mac UTC date
         = 64-bit unsigned value in seconds since beginning 1904

     -> 4 bytes time scale = long unsigned time unit per second (default = 600)

     -> 4 bytes duration = long unsigned time length (in time units)
     OR
     -> 8 bytes duration = 64-bit unsigned time length (in time units)

     -> 4 bytes decimal user playback speed = long fixed point rate (normal = 1.0)
     -> 2 bytes decimal user volume = short fixed point level
         (mute = 0.0 ; normal = 1.0 ; QUICKTIME MAX = 3.0)
     ...

    // 语法
    class MovieHeaderBox extends FullBox(‘mvhd’, version, 0) {
      if (version==1) {
      unsigned int(64) creation_time;
      unsigned int(64) modification_time;
      unsigned int(32) timescale;
      unsigned int(64) duration;
      } else { // version==0
      unsigned int(32) creation_time;
      unsigned int(32) modification_time;
      unsigned int(32) timescale;
      unsigned int(32) duration;
      }
      template int(32) rate = 0x00010000; // typically 1.0
      template int(16) volume = 0x0100; // typically, full volume
      const bit(16) reserved = 0;
      const unsigned int(32)[2] reserved = 0;
      template int(32)[9] matrix =
      { 0x00010000,0,0,0,0x00010000,0,0,0,0x40000000 };
      // Unity matrix
      bit(32)[6] pre_defined = 0;
      unsigned int(32) next_track_ID;
  }
```
所以, 有了如下代码:
```html
<input id="input" type="file" />
```
```js
const input = document.getElementById('input')
input.onchange = (e) => {
  const file = e.target.files[0]
  file
    .slice()
    .arrayBuffer()
    .then((ab) => {
      const duration = getVideoDuration(ab)
    })
}

function getVideoDuration(arrayBuffer) {
  const dv = new DataView(arrayBuffer)
  const MVHD = 0x6d766864
  const uint32length = 4
  const maxOffset = dv.byteLength
  
  let offset = 0
  let timeScale = 0
  let duration = 0
  while (offset + uint32length < maxOffset) {
    const val = dv.getUint32(offset)
    if (val === MVHD) {
      const version = dv.getUint8(offset + 1)
      if (version === 0) {
        const createTime = dv.getUint32(offset + 2 * uint32length)
        const modifyTime = dv.getUint32(offset + 3 * uint32length)
        timeScale = dv.getUint32(offset + 4 * uint32length)
        duration = dv.getUint32(offset + 5 * uint32length)
      } else {
        const createTime = Number(
          `0x${dv.getUint32(offset + 2 * uint32length)}${dv.getUint32(offset + 3 * uint32length)}`
        )
        const modifyTime = Number(
          `0x${dv.getUint32(offset + 4 * uint32length)}${dv.getUint32(offset + 5 * uint32length)}`
        )
        timeScale = dv.getUint32(offset + 6 * uint32length)
        duration = dv.getUint32(offset + 7 * uint32length)
      }
      const canPlayTime = duration / timeScale
      return canPlayTime
    }
    offset += 2
  }
  return 0
}
```