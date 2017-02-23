---
layout: post
title: 零碎见识或者经验吧
date: 2016-02-09
tags: Note
---

1. 如果要在另一台电脑上更新博客，得先删除.deploy_git，我也不知道为什么，折腾了一两个小时，终于弄好啦
2. rm -rf 删除的文件或文件夹是有办法恢复的，如果文件系统是ext,可以使用debugfs和extundelete恢复。
具体过程没看，先知道有这么回事吧

------------
2.23  
很多命令平时用的少，但每次到用的时候就忘了，今天整理两个   
1. `grub-mkconfig  -o filename`   
2.  windows引导修复，`bcdboot  system driver    efi driver`,对于efi的卷标可以在diskpart中设置
--------
2.24
1. `fdisk partition -l` 能显示分区类型
2. `partx -s /dev/sda` 能显示uuid
都能显示卷号，扇区的起始位置、扇区大小和容量

---
3.6
1. byzanz-record -d 30 -x 0 -y 0 -w 1366 -h 768 file.gif
2. ffmpeg -i input.mp4 -ss start.time -t stop.time -acopy copy -vcopy copy output.mp4
time format: 00:00:00
also how much seconds eg: 90

---
4.4
1. postgresql /etc/postgresql/9.4/main/pg_hba.conf
2. 创建用户并设置密码
3. 导入sql文件
