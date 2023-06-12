---
title: linux 常用命令
date: 2016-01-02
tags: ["Note"]
---
vsftpd:
  /etc/vs....conf

useradd -d /home/ftp

passwd username
pwd

## 系统相关

- sysctl
- nc
- htop, top, free
- pvs, vgs, lvs
- xfs_growfs
- tcpdump

## 用户相关

- useradd -d /home/ftp
- passwd username

## 编辑相关

- awk
- sed

## vim相关

- j,k,h,l 移动
- i,a 插入
- y,p 复制粘贴，:+ y, +p 将内容复制到+寄存器，将+寄存器的内容粘贴
- r 替换
- C+f, C+b 上下翻屏

## tmux相关

- C+b c 创建
- C+b w 列表
- C+b o 切换
