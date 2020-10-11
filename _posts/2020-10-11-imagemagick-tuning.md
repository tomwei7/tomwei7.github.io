---
layout: post
title: 记一次 ImageMagick jpeg 缩放性能调优
tags: ImageMagick,jpeg
---

由于同事了调岗，接手了一个简单的缩略图服务，大佬看之前的代码过于混(la)乱(ji)大手一挥带着我们用 C++ 重构了这个服务，故事就发生在新服务上线几周后。在一个愉快的周五下班后，正在桌上吃着火锅突然收到告警服务开始超时，赶紧联系同事重启了一波，但是效果甚微。无奈紧急切回来旧服务。

事后进行分析发现是因为有一张非常高分辨率的图片导致的，由于上传时只限制了用户上传的图片体积大小没有检查图片实际分辨率再加上上游服务的不合理重试，几乎让这张图片占满了所有的工作线程，最终导致雪崩。

![罪魁祸首.jpeg](/images/2020-10-11-imagemagick-tuning/df35a6b35caea63c4be3c19947407fe4426a2b3c-small.jpg)

由于图片太大上面的是缩略图，感兴趣的可以点击这里下载原图[罪魁祸首.jpeg](/images/2020-10-11-imagemagick-tuning/df35a6b35caea63c4be3c19947407fe4426a2b3c.jpeg), 原图的大小大小约 24000x17280 总像素超过4亿

### 问题重现

竟然知道了问题是超高分辨率，就需要看看有没有可以优化的地方。图片处理我们主要使用的是 ImageMagick 这个开源库，首先将服务最小化，缩略图的主要逻辑大概就这几行

```c++
Magick::Image im;
im.read("big-img.jpeg");
im.filterType(Magick::LanczosFilter);
im.resize(Magick::Geometry(400, 288));
im.write("thumbnail.jpeg");
```

首先用出问题的图片跑一下试试 (因为一些资源管理问题，ImageMagick 在这里没有开启 openmp，都是单线程工作的)

```
User time (seconds): 9.71
System time (seconds): 0.96
Percent of CPU this job got: 99%
Maximum resident set size (kbytes): 4954472
```

在我自己电脑上大约需要 10s 占用约 4G 内存，实际在公司服务器大约需要 15s 左右 (AMD Yes!)，对于一个在线服务很明显是不能接受的。通过 perf 工具可以看出，CPU 主要都是花在 resize 的过程中。

调整 filterType 可以起到优化的左右，比如将但是调整 filterType 到 PointFilter，处理时间一下子就可以来到 1.4s 内存占用还是 4G，但是图片效果基本就不能接受了

```
User time (seconds): 1.40
System time (seconds): 0.91
Percent of CPU this job got: 99%
Maximum resident set size (kbytes): 4953288
```

![pointfilter-1.jpeg](/images/2020-10-11-imagemagick-tuning/pointfilter-1.jpeg)

而且修改 filterType 可能会导致其他图片最终呈现的效果与约定的不一致，所以 filterType 基本上是不能修改的。

### 从 ImageMagick 入手

为了解决这个问题我又去翻了一下 ImageMagick 的文档与代码，看看是不是已经有对这种超高分辨率的图片做缩略图的优化了，毕竟这个是很常见的需求。果然 ImageMagick 除了 `resize` 还提供了一个 `thumbnail` 的方法，`thumbnail` 顾名思义就是用来生成缩略图的，翻了一下代码 `thumbnail` 主要分为两个过程

1. 对原图进行简单采样及每隔 5 个像素点取一个，等于将原图长宽都缩小到以前的五分之一
2. 进行普通 `resize` 操作

因为第一步的采样过程非常暴力，所有只有到缩略图的大小是原图面积的 10% 以下而且原图的长宽都需要大于25像素，否则都会降级为普通的 `resize` 操作

简单修改下代码进行测试

```c++
Magick::Image im;
im.read("big-img.jpeg");
im.filterType(Magick::LanczosFilter);
im.thumbnail(Magick::Geometry(400, 288));
im.write("thumbnail.jpeg");
```

改用 `thumbnail` 耗时一下子降到到 1.4s 左右，基本和使用 PointFilter 性能差不多，而且最终输出的图片与直接进行 `resize` 的结果从肉眼上基本没有差距，但是仍然会占用约 4G 左右的内存。

```
User time (seconds): 1.44
System time (seconds): 0.90
Percent of CPU this job got: 99%
Maximum resident set size (kbytes): 4912756
```

使用 `thumbnail` 结果

![thumbnail-1.jpeg](/images/2020-10-11-imagemagick-tuning/thumbnail-1.jpeg)

直接 `resize` 结果

![direct-resize.jpeg](/images/2020-10-11-imagemagick-tuning/direct-resize.jpeg)

### 从 jpeg 格式入手

虽然使用了 `thumbnail` 可以解决耗时问题，但是一次处理要占用 4G 多内存还是不可接受的。要避免占用这么多内存就要避免一次把所有像素都放到内存里面，一般会想到的就是通过流式的方式处理数据，这样就能大大降低内存占用了，另一个开源的图片处理库[libvips](https://github.com/libvips/libvips) 就是这样做的，但是将 ImageMagick 改造成流式处理的话这个工作量就过大了。对于 jpeg 这种图片格式还有另外一种方式就是可以在解压像素的时候做到类似缩放的功能，这里就要牵扯到 jpeg 的工作原理了，jpeg 的压缩算法基于[离散余弦变换](https://zh.wikipedia.org/wiki/%E7%A6%BB%E6%95%A3%E4%BD%99%E5%BC%A6%E5%8F%98%E6%8D%A2)，这里打个比方可以理解为 jpeg 将像素存储在一个函数中，当解压的时候输入像素的下标就能获得这个像素原始的数据，比如对于一个 1000x1000 的图片，我希望在解压成一张 500*500 图片，那样在解压时我只要跳过那些我不需要的像素点就好了，当时实际情况更加复杂，jpeg 也不是存储在一个函数中的。这里推荐一个很好的介绍 jpeg 工作原理的视频 [How JPEG works](https://www.youtube.com/watch?v=f2odrCGjOFY)

有了这样的理论基础我们就可以实践了，其实 `libjpeg` 已经提供了这个参数，可以再解压 jpeg 的时候指定缩放的比例，比例有固定的档位在 1~8 之间，也就是说最小只能缩放到原图的长宽的八分之一。 在 ImageMagick 中设置这个参数的方式有点诡异，代码如下，这里指定 `jpeg:size` 其实只是指定了一个期望值实际获得的图片大小会是一个大于目标尺寸的最小值，比如这次测试的图片实际获得的图片大小是 3000x2160。

```c++
Magick::Image im;
MagickCore::ImageInfo* im_info = im.imageInfo();
im_info->options = MagickCore::NewSplayTree(MagickCore::CompareSplayTreeString, (void*(*)(void*)) nullptr, (void*(*)(void*)) nullptr);
MagickCore::AddValueToSplayTree((MagickCore::SplayTreeInfo*)(im_info->options), "jpeg:size", "400x288");
im.read("big-img.jpeg");
im.filterType(Magick::LanczosFilter);
im.resize(Magick::Geometry(400, 288));
im.write("shrink-on-load.jpeg");
```

优化之后耗时降低到了 0.4s 左右，内存也降到了 100M！ 而且最终输出的结果与直接进行 `resize` 的结果从肉眼上基本没有差距

```
User time (seconds): 0.42
System time (seconds): 0.02
Percent of CPU this job got: 99%
Maximum resident set size (kbytes): 104656
```

解压时缩放结果

![shrink-on-load.jpeg](/images/2020-10-11-imagemagick-tuning/shrink-on-load.jpeg)

直接 `resize` 结果

![shrink-on-load.jpeg](/images/2020-10-11-imagemagick-tuning/shrink-on-load.jpeg)

### 题外话

因为 ImageMagick 不支持 SIMD 加速在测试的过程我也尝试过使用支持 SIMD 加速的图片库比如 [pillow-simd](https://github.com/uploadcare/pillow-simd) 处理完整分辨率的图像也基本能控制在 1s 级别，不能利用现代 CPU 的功能是 ImageMagick 的硬伤

### 总结

这次缩略图的优化，对于这种超大分辨率的图片处理速度提升了一个数量级 (10s -> 0.5s) 同时内存占用也降到了一个数量级 (4G -> 100M)。基本上达到了在线服务的能力。这个故事告诉我们没事别去重构老服务😅

### 参考

- [Imagemagick thumbnails/](http://www.imagemagick.org/Usage/thumbnails/)
- [libjpeg-turbo.org/](https://libjpeg-turbo.org/)
- [Imagemagick Single Convert Command Performance](https://stackoverflow.com/questions/42022982/imagemagick-single-convert-command-performance)
- [How JPEG works](https://www.youtube.com/watch?v=f2odrCGjOFY)
