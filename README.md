# 前言

之前由于项目原因接触到了音视频开发，前期经过仔细调研，选择了FFmpeg作为开发工具，随后经历了许多常用功能开发，也遇到了性能优化、逐字节bug排查等很多意思的挑战。

现在决定将一些颇具意义的过程记录下来，并在此基础上继续实现一些有用有趣的扩展。这些工作的最终呈现形式可能是一个流媒体播放器（或者叫做流媒体客户端），优秀的开源流媒体框架和播放器很多，如ZLMediaKit、VLC、ffplay，这个播放器更多地可以看作是我个人探索和实践的记录，既方便本人进行C/C++最佳实践，也可以帮助一些刚接触音视频开发的同学迅速上手。涉及到的功能可能和国内工业界需要在音视频传感器上做二次开发的场景比较契合，希望能抛砖引玉，为音视频社区贡献自己的一份力，和大家共同成长。

计划是每个功能都配以单独的博客和工程，功能之间没有较多的联系；功能整合的播放器作为一个工程，它需要追求较好的框架和性能；会有单独章节对FFmpeg源码进行解析。本系列默认你了解数字信号处理、数字图像处理、音视频原理、H264原理等前置知识，博客并不会把这些知识详细讲解（音视频领域是一个综合而庞大的体系，如果展开可能是大学一两年的课程），只会穿插一些重要知识点的回溯。后期会根据情况不断对博客内容进行修改和优化。

目前框架是Qt+FFmpeg（Qt5.12.8 mingw73_64，FFmpeg4.4.1），以windows版本为主。如果有你想看到的但我没写到的功能，或者遇到问题，请直接加微信（JoggingJack）联系我（博客留言效果不好，我不会经常check），**your feedback will be highly valued，任何反馈对我来说都非常重要**。

