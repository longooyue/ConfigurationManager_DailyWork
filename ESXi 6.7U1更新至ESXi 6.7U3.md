# ESXi 6.7U1更新至ESXi 6.7U3

**回忆一次ESXi更新的过程,如果使用OEM定制镜像来尝试更新肯定会遇到更新不了的情况,虽然VMWare提供了解决办法,但是文章有错.**

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




直接通过关键词应该能搜出来很多经验文章,这里还是直接引用VMWare的文档,虽然是U2的文档,但是U3理论也是通用的.
https://docs.vmware.com/en/VMware-vSphere/6.7/vsphere-esxi-672-upgrade-guide.pdf
其中67页,有这么一句
```
Important If you are updating ESXi from a zip bundle in a VMware-supplied depot, either online 
from the VMware Web site or downloaded locally, VMware supports only the update method 
specified for VMware-supplied depots in the topic Upgrade or Update a Host with Image Profiles
```
对应 topic 在68页,也有这么一句
```
Important If you are upgrading or updating ESXi from a zip bundle in a VMware-supplied depot, 
either online from the VMware website or downloaded locally, VMware supports only the update 
command esxcli software profile update --depot=depot_location --profile=profile_name.
```
文档中提到,只支持该方法  esxcli software profile update --depot=depot_location --profile=profile_name  进行更新,原因也很简单,关键词一下ESXi更新报错,就能找到一堆文章.里面基本用的都是vib install 等方法进行更新的.
![Bing](https://i.328888.xyz/2023/02/22/gFuM8.png)<br>

其实在VMWare 的KB https://kb.vmware.com/s/article/56145 中也有对应的解决方法(其实就是详细介绍 esxcli software profile update --depot=depot_location --profile=profile_name 的使用方法,在这里进行介绍 )

在KB中给的方法1中

- Using profile update command:
  - __To get the profile name run the below command,__
    __esxcli software profile get__

    __Example:__
    __[root@esx1:~] esxcli software profile get__
    __HPE-ESXi-6.7.0-Update3-iso-Gen9plus-670.U3.10.4.5.19__
    __Name: HPE-ESXi-6.7.0-Update3-iso-Gen9plus-670.U3.10.4.5.19__
    __Vendor: Hewlett Packard Enterprise__
    __Creation Time: 2020-02-22T06:07:54__
    __Modification Time: 2020-02-22T06:08:10__
    __Stateless Ready: True__
    __...__ <br>
    
这部分我个人认为是是有些多余的.顶多算是一个检查
因为实际更新过程中,需要的profile name并不是通过这个方法获得的.
在后面的更新步骤中,KB给的例子如下
 
Example:
 esxcli software profile update -p ESXi-6.7.0-2018xxxxxxx-standard -d /vmfs/volumes/datastore1/update-from-esxi6.7-6.7_update01.zip

__Note: To get a list of available profiles within a path use the command below:__
```
esxcli software sources profile list -d <location of ZIP file> 
```

下面实际演示一下我操作过程

```Bash
PS C:\WINDOWS\system32> ssh root@172.21.1.81
Password:
The time and date of this login have been sent to the system logs.

WARNING:
   All commands run on the ESXi shell are logged and may be included in
   support bundles. Do not provide passwords directly on the command line.
   Most tools can prompt for secrets or accept them from standard input.

VMware offers supported, powerful system administration tools.  Please
see www.vmware.com/go/sysadmintools for details.

The ESXi Shell can be disabled by an administrative user. See the
vSphere Security documentation for more information.
[root@Localhost:~] ls /vmfs/volumes/ESXI_LOCAL/iso/
ESXi670-202210001.zip
SW_DVD9_Win_Server_STD_CORE_2019_1809.18_64Bit_ChnSimp_DC_STD_MLF_X22-74329.ISO
VMware-VIM-all-6.7.0-15132721.iso
ubuntu-22.04.1-live-server-amd64.iso
[root@Localhost:~] esxcli software sources profile list -d /vmfs/volumes/ESXI_LOCAL/iso/ESXi670-202210001.zip
Name                              Vendor        Acceptance Level  Creation Time        Modification Time
--------------------------------  ------------  ----------------  -------------------  -------------------
ESXi-6.7.0-20221001001s-no-tools  VMware, Inc.  PartnerSupported  2022-09-21T13:36:44  2022-09-21T13:36:44
ESXi-6.7.0-20221004001-standard   VMware, Inc.  PartnerSupported  2022-09-21T13:36:44  2022-09-21T13:36:44
ESXi-6.7.0-20221001001s-standard  VMware, Inc.  PartnerSupported  2022-09-21T13:36:44  2022-09-21T13:36:44
ESXi-6.7.0-20221004001-no-tools   VMware, Inc.  PartnerSupported  2022-09-21T13:36:44  2022-09-21T13:36:44
[root@Localhost:~] esxcli software profile update -p ESXi-6.7.0-20221001001s-standard -d /vmfs/volumes/ESXI_LOCAL/iso/ESXi670-202210001.zip
```

整理了一下操作步骤,一共就下面这三行
```
2023-02-21T09:39:35Z shell[2239481]: [root]: esxcli software sources profile list -d /vmfs/volumes/ESXI_LOCAL/iso/ESXi670-202210001.zip
2023-02-21T10:04:53Z shell[2239481]: [root]: esxcli software profile update -p ESXi-6.7.0-20221001001s-standard -d /vmfs/volumes/ESXI_LOCAL/iso/ESXi670-202210001.zip
2023-02-21T10:06:11Z shell[2239481]: [root]: reboot
```