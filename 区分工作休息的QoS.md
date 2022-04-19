# 利用WindowsServer实现基于时间的QoS更新

**记录一次捣鼓QoS的过程,不是一件很麻烦的事情,但是估计使用MPLS VPN组网的公司的Helpdesk都会遇到的头疼问题.**

****
## 目录
* [现状](#现状)
* [问题](#问题)
* [需求](#需求)
* [思路](#思路)
* [解决](#解决)
    * 制作脚本
    * 优化脚本
    * 完成脚本
    * 测试脚本
* [结束语](#结束语)

## 现状：
   - 主站点将部署包Distribute到全国多数办公区域都有分配点服务器(Distribution Point)
   - 有DP的办公区域内的电脑从DP获取部署包 没有DP的办公区域内的电脑从主站点获取部署包
   - 有DP的办公区域带宽基本都在4M及以上 没有DP的办公区域带宽基本都在1M及一下

## 问题：
   - 由于没有DP的办公区域内的电脑从主站点获取部署包,在完成deploy后,电脑只要完成评估和更新策略后会立刻从主站点下载部署包
   - 虽然该办公区域内的电脑台数不多,但是一台电脑带宽太小24H都不一定下载完一个功能更新.更不要说2台及以上同时下载的情况
   - 下载不能正常完成本身对业务没有影响,但是下载过程中对带宽的占用就十分影响业务了

## 需求：
   - 考虑到设备安全,质量更新功能更新每个月/每半年必须进行
   - 确保工作时间不影响基础业务(邮件,共享文件夹等)
   - 确保休息时间全速下载

## 思路：
   - Windows本身带QoS功能,但是不能基于时间对QoS进行更新
   - Windows本身的任务计划程序可以基于时间进行文件操作
   - 把两者结合一下就可以

## 解决：
**根据常识,Windows所有的设置都保存在注册表里,展示出来的GUI只是作为更新注册表的入口.
有关QoS策略的注册表保存在这个地址HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\QoS\ <br>**
但是直接修改注册表并不会让QoS生效,必须通过组策略这个入口来进行操作,
对组策略的更新会立刻生效而不需要执行 **gpupte /force**<br>


**那么首先看看QoS在哪里改,要怎么改**<br>
### 摸索QoS的实现

要先找到QoS在哪里<br>
本地组策略编辑器->计算机设置->Windows设置->基于策略的QoS<br>
![](https://s3.bmp.ovh/imgs/2022/02/51e31b1f42e69b28.png)<br>

接下来看看能不能通过文件/注册表的形式去代替在GUI上操作,完成QoS的更新,通过关键词摸索出来,这个所谓的"文件"是Registry.pol ([Registry Policy File Format | Microsoft Docs](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/policy/registry-policy-file-format))

新建一个QoS 看一看Registry.pol 和 注册表的变化

![](https://s3.bmp.ovh/imgs/2022/02/b334287503b584ed.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/18f573ded0df9f59.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/88741081d4c17425.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/c77d554b14d3c2ed.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/45e4418d1c929b4f.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/6883ed109f7db53e.png)<br>
![](https://s3.bmp.ovh/imgs/2022/02/42b398bb0b83029b.png)<br>
***Before 89.6MB/s***
![](https://s3.bmp.ovh/imgs/2022/02/19931459b3ab60e8.png)<br>
***After 1.59MB/s***
![](https://s3.bmp.ovh/imgs/2022/02/dae4bb07d0b474c8.png)<br>

![](https://s3.bmp.ovh/imgs/2022/02/0c3666264111e11c.png)<br>