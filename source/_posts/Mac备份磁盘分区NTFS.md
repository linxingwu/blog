---
title: Mac备份磁盘分区NTFS
date: 2018-12-11 14:21:55
tags:
---

买了一个移动硬盘准备用作mac的备份盘，插上之后发现选择备份盘之后会被格式化为mac特有的文件系统，随后在windows上就无法访问了。我这个人比较抠门，觉得浪费了一个移动硬盘，于是想办法让硬盘在win和mac上都可以使用。
一开始的思路：

 * Mac上自带的磁盘管理工具

 这个工具可以分区，如果我把移动硬盘分为两个区，其中一个区用来专门给mac备份，另一个格式化为win可读的文件系统格式这样就可以。但是实际操作起来发现，mac磁盘管理工具问题不小，分区之后没有办法格式化为ntfs或者其他win支持的格式(ExFat)。都会失败。这条路根本走不通。当然也有人这一步直接就成功了。可能是osx版本不同导致的问题吧，没有必要进一步研究。
 
 * Windows 磁盘管理工具
 
 经过实验，这个方法可行。具体步骤是：  
 右击"我的电脑"-"管理"-"磁盘管理"-选择移动硬盘，右击"压缩卷"，这样就能开辟出来一个新分区，格式化为ntfs，这个新的分区以后就可以在windows上使用，充当正常的移动硬盘。
 
  ![压缩卷](http://pjme92tyf.bkt.clouddn.com/%E5%8E%8B%E7%BC%A9%E5%8D%B7.png)

 
 随后插上mac，见证奇迹的时候到来了，这个时候mac上会显示有两块硬盘，选择刚刚没有格式化的那一个分区作为备份盘。然后启动时间机器，把这个分区格式化为mac的文件系统，并备份文件。  
 
  ![Mac](http://pjme92tyf.bkt.clouddn.com/mac%E6%A0%BC%E5%BC%8F%E5%8C%96.png)

 
 至此，已经达到了我最初的目的，在Mac上可以看到两个盘符，在win上只能看到自己ntfs的那一个盘符。因为在windows上识别不了mac的文件系统，在mac上却可以识别windows的文件系统。
 
 
 因为我之前在mac上设置过时间机器的备份盘，导致新插上硬盘时间机器不会蹦出来要求选择备份盘。这个时候要在"系统偏好设置"-"时间机器"里主动选择磁盘。选择磁盘之后时间机器还不会主动备份，这时候在时间机器偏好设置里勾上"在菜单栏中显示时间机器"然后在菜单栏的标志里选择"立即备份"
 
 
 题外话,鸟哥的Linux私房菜中提到过装windows和linux双系统的时候引导区也是这个原理，必须先装windows，然后再装linux。
 
 题外话，我在网上搜索解决办法的时候发现有些人提到过，苹果因为和微软的版权问题，屏蔽了mac对ntfs的写功能，导致只能读，有人通过修改/etc/fstab 文件来试图达到让mac可以写ntfs的目的。我尝试过这个办法，但是时间机器不管那么多，选择备份盘之后，一律要求先格式化。这条路也行不通。
