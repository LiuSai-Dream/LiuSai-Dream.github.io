---
layout: post
title: Andriod 设备标识符
category: Android
---

本文叙述了一些Android平台设备相关标识符以及其优缺点。

## Device ID（IMEI / MEID）

- 优点：
1. 持久性，在设备清除数据和恢复出厂设置后仍然存在且值不改变 

- 缺点：
1. 非手机设备，没有telephony模块，无法获得该标识符
2.  权限，需要READ_PHONE_STATE权限
3. 一些手机厂商的实现存在buggy，例如会返回“0”或者“*”

- 结论：
1. 具有很好的唯一性
2. 并不是总可得到
3. 可通过校验，防止一些虚假IMEI / MEID


## UUID

- 优点：
1. 可以进行唯一的标识，适用于APP下载量的统计

- 缺点：
1.纯软件设置，与硬件无关，因此无法标识设备

- 结论：
1. 很好的标识唯一性的方法
2. 无法标识硬件设备


## Mac Address

- 优点：
1. 普通人不易修改

- 缺点：
1.能够修改
2. 并且不是所有的设备都有wifi
3. 如果具有wifi，但是关闭，仍然有可能得不到该地址

- 结论：
1. 可作为辅助，不能完全依赖


## Serial Numer

- 优点：
1. 没有telephony模块的设备需要提供该值， 一些手机设备也会提供

- 缺点：
1.在Android2.3之后才提供

- 结论：
1. 对于无telephony模块的设备可以使用该值


## Sim Serial Number

- 优点：
1. 唯一识别sim卡

- 缺点：
1. 手机也有不插卡的情况

- 结论：
1. 可作为辅助


## Android_Id

- 优点：
1. 

- 缺点：
1.设备清除设备后会重置
2. 某手机厂商生产的所有设备具有相同的android_id
3. 在Android2.2中不可靠
4. 在android4.2之后，设备启用多用户功能后，每个用户具有不同的Android_Id.


推荐的做法：
获取可获得的标识符，根据需求，从两个维度进行分析：1. 从物理设备维度，使用device_id, Serial number进行标识； 2. 从应用本身的维度，使用UUID统计独立安装次数。

当然比较好的做法是拼接若干个标识符。


参考：

- [Google Developer] ： http://android-developers.blogspot.sg/2011/03/identifying-app-installations.html
- http://caizhitao.com/2015/11/27/android-uids-IMEI-MEID-PESN-ESN/
- [推荐] ：http://www.loyea.com/?p=350
- [借鉴极光的唯一标识符] ：http://blog.jiguang.cn/registrationid/