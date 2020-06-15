dubbo调用在2.7.5版本以下checkAndUpdateSubConfigs导致的性能问题的解决

一、背景

toc 的监控在日常环境测试时，发现日常环境在大量JOB需要调度的情况下，发现三台机器调度的瓶颈大概为8k每分钟，也就是一台机器每秒处理44个，平均每个JOB耗时22毫秒。这还是在已经分片的情况下。

如果只是 dubbo 调用就要这么久肯定不正常。于是进行排查。

二、数据

理论上最耗时的点应该是在远程调用的过程，也就是网络延迟。在日常环境用arthas抓取方法耗时，发现耗时的过程反而是在

![652BE8F3-6A34-467F-BCE0-5362B40E1E93](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/652BE8F3-6A34-467F-BCE0-5362B40E1E93.png)

这个方法上。

用 arthas 抓的图如图所示。

![9C1437EC-09DE-4039-9800-C68D1591701D](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/9C1437EC-09DE-4039-9800-C68D1591701D.png)

可以看到 config.get 这个方法远大于或者接近实际调用过程。理论上对这个点进行优化可以至少提升一倍的性能。

三、解决过程

继续分析 get 方法，其内部耗时主要还是在 checkAndUpdateSubConfigs(); 这个方法。

![A0BCC225-A330-45BB-AC2F-4C73EB38AABB](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/A0BCC225-A330-45BB-AC2F-4C73EB38AABB.png)

这个方法主要是获取刷新各种配置。而且由于我们没有用新版的dubbo配置中心，导致每调一次GET会打一次warn日志![42EDB6B8-5D43-4E59-BD73-D5F22C7CB69B](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/42EDB6B8-5D43-4E59-BD73-D5F22C7CB69B.png)

于是考虑要不在get 做个缓存来解决。但是在解决之前也查一下有没有其他人碰到这个问题，结果还真有。他们解决的办法是升级dubbo 版本。apache 2.7.5 版本

就解决了这个性能问题，将 checkAndUpdateSubConfigs 丢到 init 方法里，只在 init 的时候调用一次。

![1FFB477A-6B66-4741-8918-854868F7642B](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/1FFB477A-6B66-4741-8918-854868F7642B.png)

不过由于升级 dubbo 核心版本没那么容易，于是暂时就在 get 外层做了个缓存来解决。

下图是修改后的耗时，可以看到耗时基本只剩调用这块。具体的性能提升多少等待后续的测试。

![8015F365-6DDC-4F48-BAB3-60BAF289208A](https://cdn.jsdelivr.net/gh/xingcici/oss@master/uPic/8015F365-6DDC-4F48-BAB3-60BAF289208A.png)