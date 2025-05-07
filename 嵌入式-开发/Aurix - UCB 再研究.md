---
title: Aurix - UCB 再研究
categories: 嵌入式-开发
---
## 琐碎知识点
1. 每个UCB都有自己的访问控制权限，而访问控制由`PROCONx registers`中的内容决定，其中的内容在启动时从UCB中加载。因此，空片子首次写入UCB时很重要
2. 每个UCB分为ORIG UCB，COPY UCB
3. UCB模块有多组UCBs， 例如UCB_BMHD， UCB_SSW，UCB_USER，UCB_HSM，UCB_SWAP等

## UCB_BMHD - Table-35
1. 如果ORIG/COPY BMHD的confirmation code 均为ERRORED，则该BMHD不会被评估（安装）
2. ORIG/COPY中的内容应该一致
3. BMHD会在启动的时候被SSW固件评估
4. BMI域{0-15}：
	- 0 -> HWCFG Pin enabled(0), disabled(1)
	- 3:1 -> Start-up mode
5. BMHDID域{16-31}： B359H
6. STAD域{32bit}: 该地址应该始终处于PFlash中，且字对齐