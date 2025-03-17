# scan 常见问题

> 如果没有找到遇到的问题，请及时在 [issues](https://github.com/timerring/bilive/issues/new/choose) 中提出。

## 关于渲染速率

渲染速率主要与硬件以及弹幕数量有关，测试硬件的基本区间 2核 Xeon(R) Platinum 85 的 CPU 的渲染速率在 3 ~ 6 倍之间，也可使用 Nvidia GPU 加速，项目的测试显卡为 GTX1650，其渲染速率在 16 ～ 20 倍之间。 

弹幕渲染具体时间可通过 `渲染速率x视频时长` 估算。

使用 Nvidia GPU 加速的相关参考：
+ [Using FFmpeg with NVIDIA GPU Hardware Acceleration](https://docs.nvidia.com/video-technologies/video-codec-sdk/12.0/ffmpeg-with-nvidia-gpu/index.html)
+ [使用GPU为FFmpeg 加速](https://yukihane.work/li-gong/ffmpeg-with-gpu)

## 渲染速率为什么下降

长时间地使用 GPU，温度升高可能会导致 GPU 降频，从而导致渲染速率下降。可以通过 `nvidia-smi -q -d CLOCK` 查看 GPU 频率信息。

## requests 请求错误
```
requests.exceptions.ConnectionError: ('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))
```

Reference: https://stackoverflow.com/questions/70379603/python-script-is-failing-with-connection-aborted-connectionreseterror104

解决方案：网络问题，你应该知道怎么做。

## 字体信息错误
```
Glyph 0x... not found, selecting one more font for (Microsoft YaHei, 400, 0)
```
Reference：https://github.com/timerring/bilive/issues/35

解决方案：通常 ffmpeg 无法渲染表情，因此很多情况下关于表情的渲染都会报错，我已经通过一个通用正则滤除了 99% 的表情，当然可能会有其他奇怪的表情和字符，如果报错忽略即可，ffmpeg 会渲染为一个方框，不影响效果。

## 渲染出来的弹幕和字幕为什么都是方框？

这是字体缺失的问题，往往出现在手动部署过程中，我默认采用的是微软雅黑字体，在本项目的 assets 目录下有 msyh.ttf，安装它即可。docker 版本应该不会有这个问题，因为在 dockerfile 中已经完成了该过程。当然如果你想使用别的字体也可以在 DanmakuFactory 指令中转换时指定对应的字体。

## 为什么又改回队列的方式渲染弹幕了

由于通常 nvidia 加速下的弹幕渲染速率为 10 ~ 15x，而 whisper 执行语音识别的速率为 5x，因此之前由 whisper 创建弹幕渲染的方式是完全行得通的，但是对于弹幕非常多的直播间来说(20分钟片段产生15400+条弹幕)，渲染速率会下降到 2 ～ 4x，当新一轮 whisper 处理后的片段进行弹幕渲染时，会进一步使渲染速率下降，堆积超过 3 个并行的弹幕渲染进程，会由于显卡并发编码的数量限制而导致失败，此外还会存在爆显存的风险。因此，为了保证渲染的质量和程序的稳定性，我原改回列队的方式处理弹幕渲染。

## WSL 运行报错 ERROR：［Errno -2］ Name or service not known

主机名解析的问题，wsl 通常网络问题比较多。From [issue 159](https://github.com/timerring/bilive/issues/159)

解决方案：主要两个思路
1. 可以按照内容检查一下网络：https://unix.stackexchange.com/questions/589683/wsl-dns-not-working-when-connected-to-vpn
2. 不用 record.sh 启动，直接在命令行终端里执行 blrec 然后浏览器访问 `http://localhost:2233` （default），按照https://blog.csdn.net/Yiang0/article/details/127780263 ，本机 Windows 可以直接通过 localhost 访问 WSL2，查看一下是否有正常运行的进程。

## 录制没有切片产生

> 首先检查 `src/config` 中的 `AUTO_SLICE` 是否为 `True`，如果为 `False`，则不会进行切片处理。

通常默认设置，如果视频文件大小小于 200 MB，则不进行切片处理，这样做的目的主要是防止一些碎片化的片段也被切片上传，因为对于一个网络不佳或者经常连线的主播，一场直播可能会产生几十个片段，如果每两分钟或者三分钟的连线都要再切片一次，冗余的内容对观众来说观感不是很好，因此权衡之下我设置了一个 threshold 为 200 MB，确保足够长的片段才会被切片。

解决方案：修改 `src/config` 中的 `MIN_VIDEO_SIZE` 数值为你所需要的限制，然后重新执行 `./scan.sh` 即可。

## RuntimeError: CUDA error: no kernel image is available for execution on the device

日志显示你的 ubuntu 的 nvidia gpu 驱动和 cuda 版本可能存在不匹配的问题，不能正确调用 cuda 核心。

解决方案：详细参考[这篇文章](https://zhuanlan.zhihu.com/p/466793485)。