# 【FFmpeg】ffplay播放rtsp视频流花屏问题

**问题描述：**ffplay播放rtsp视频流时，播放过程中随机出现花屏现象。

**基本流程学习：**阅读ffplay源码，熟悉其播放rtsp视频流的基本流程。

在ffplay源码阅读和分析的基础上，画出了其播放rtsp的函数调用关系，如下图所示：

![](https://images0.cnblogs.com/blog/414008/201308/06112939-d2399797e7ea4e5dae60d16dd48a203b.png)

**avformat\_open\_input**函数根据输入的文件名，与**rtsp\_read\_packet**关联。

**rtsp\_read\_packet**完成每个rtp包的读取和解析，读取主要是利用rtp\_read从缓冲区获取数据，解析主要是根据rtp协议，解析rtp包，得到h264码流数据，由rtp\_parse\_packet完成。

av\_read\_frame读取一帧数据的avpacket包，主要是调用rtsp\_read\_packet读取h264码流数据包，然后由av\_parser\_parse2组成h264 码流包，最终组成一帧数据的avpacket。

**错误测试：**发布不同分辨率的rtsp视频流，测试错误产生的原因。

利用VLC发布视频的rtsp服务，经测试，同一种视频封装格式，分辨率越小，花屏现象越少。

分辨率越小，服务端发送给客户端的数据越小，其花屏现象越少，说明花屏现象与服务端发送的数据量有关。

可能的原因是服务端发送的数据量较大时，客户端缓冲区不足，导致数据丢失的问题，从而引起花屏现象。

**错误验证：**修改ffmpeg源码，输出客户端接收的数据包信息，验证是否存在数据丢失的问题。

源码修改如下图所示，主要是输出RTP包的序号，根据序号判断是否存在丢包问题。

![](https://images0.cnblogs.com/blog/414008/201308/07090233-599fbdd301cd494c89c9dd4694285424.png)

信息输出结果如下图所示，正常情况下，RTP的序号是连续的，而由输出信息可知RTP序号不连续，因而存在丢包的问题。

![](https://images0.cnblogs.com/blog/414008/201308/07090512-2f8a8c5c9d1548ff8b5244b96348fb8a.png)

**解决方法：**增加客户端接收数据的缓冲区，避免丢包现象的产生。

源码修改如下图所示，主要是将UDP\_MAX\_PKT\_SIZE增大了10倍。

![](https://images0.cnblogs.com/blog/414008/201308/07090802-a154c99da7094dc58e5a92f3dd3a36ed.png)

