---
layout: post
title:  image process
date:   2015-06-18 17:50
categories: coding
---

游戏临近测试，但是包的大小一个版本比一个版本要大，为了减小包的SIZE，最近研究了一下图片处理这块。由于是2D的游戏，包的大小主要由.png图片影响着，打算从这块入手。

1.几种常见的图片格式介绍：

	* .bmp: Windows位图,最常被Microsoft Windows 程序以及其本身使用的格式。可以使用无损的数据压缩，但是一些程序只能使用未经压缩的文件。

	* .png: 无损压缩位图格式。起初被设计用于代替在互联网上的GIF格式文件。与GIF的专利权没有关联。色深:1, 2, 4, 8, 16, 24, 32, 48, 64,阿尔法通道（8- or 16-bit）。

	* .jpeg/jpg: 在网络上广泛使用于存储相片。使用有损压缩，质量可以根据压缩的设置而不同。压缩算法：离散余弦变换, RLE, 哈夫曼。色深：8位（灰阶）, 12位, 24位。没有阿尔法通道。

2.由于.png是无损压缩，使用的是zlib压缩，再次对些压缩也不会有质的减小。只能在可接受的范围内减少图片的分辨率。

	* 最初我们选择一个比较偷懒的方法来处理，使用opencv（一个很庞大的库，20M+）来对图片进行处理。使用下面的API对图片进行图片的缩放：

		void cvResize( const CvArr* src, CvArr* dst, int interpolation=CV_INTER_LINEAR );

	src: 输入图片

	dst: 输入图片

	interpolation：

		* CV_INTER_NN - 最近邻差值

		* CV_INTER_LINEAR - 双线性差值 (缺省使用)

		* CV_INTER_AREA - 使用象素关系重采样。当图像缩小时候，该方法可以避免波纹出现。当图像放大时，类似于CV_INTER_NN

		* CV_INTER_CUBIC - 立方差值

	在对比几种插值算法与缩放因子产生的效果后，我们决定用CV_INTER_LINEAR算法与黄金分割点进行把.png图片统一缩小，然后游戏运行加载图片时，再用同样的方法把图片还原为原来的大小。这样可以把图片减小到原图片三分之一。
	运行效果，对于场景背景，还是可以接受的，但是UI、角色及动画已经无法让人接近，再加上opencv这样厚重的库，所以到此就放弃此方法啦。

	* 既然.png到.png这条路走不通，下面就寻求其他的方法。.jpeg/jpg虽然是有损压缩，但是效果还是比较能接受的。于是想是不是可以用.jpeg/jpg+alpha进行打包，然后到游戏中还原为.png

	于是使用强大的图片处理工具：ImageMagick对相关的.png图片进行处理，如果下：

		@echo off
		set exevar="E:\pngtojpg\ImageMagick-6.9.1-4\convert.exe"
		for /f "usebackq tokens=*" %%d in (`dir /s /b *.png`) do (
			%exevar% "%%d" -background black -alpha remove "%%~dpnd.jpg"
			%exevar% "%%d" -background black -channel A -alpha extract "%%~dpnd_alpha_mask.png"
		)

	把每一张.png图片分解为一张.jpeg/jpg和一张alpha.png的图片，然后在游戏中修改相关的加载接口，把.jpeg/jpg+alpha.png合成.png来使用。这样原来background 140M+的文件，转换后只有
	40M+的大小，对于UI、角色及动画这样处理效果也是可以接受的。-background black，如果不加这个选择，图片中过渡比较厉害的地方会产生白边，这点还是不能让人接受，加了会产生黑边，
	但是黑边看着比较自然，所以还是可以的。（ps, 处理前包为200M+，处理后100M左右）
	
3.对于.png变换为.jpeg/jpg+alpha具体减少率是因图片而异的，不同的alpha值.png转换后的比率也是不同的。

虽然画面质量降低了，但是如果不是专业的美术进行对比，还是很难发现其差别的。关键包的大小已经减少到基本满意的大小啦！