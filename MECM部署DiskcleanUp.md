#MECM部署DiskcleanUp

##记录一次捣鼓MECM的过程,不是一件很麻烦的事情,但是估计所有Helpdesk都会遇到的头疼问题。

###现状：
- 所有电脑通过软件中心安装质量更新和功能更新,部分OU的账号属于本地管理员组,另外一部分不属于.
- 各公司管理员交付电脑时未充分考虑员工使用情况,采购的电脑磁盘较小,系统文件夹全都在系统分区下；365Apps（Outlook）默认文件夹也都在系统分区.
- 员工的操作习惯较差,喜欢把业务用文件保存在[桌面]文件夹；下载的文件不定期地删除；发送文件通过邮件附件形式发送.

###问题：
- 因为上面的这些原因,使得C分区剩余空间会被快速得使用掉,最终导致下一次功能更新甚至质量更新无法正常进行.

###需求：
- 各公司规定不一样,不更改原有GPO（就算让所有账号都在本地管理员组,不运行磁盘清理的人永远不会执行）
- 让员工无需操作就可以自动删除不需要的文件.
- 最后让所有接入网内的电脑能够正常地通过软件中心进行质量更新和功能更新.

###思路：
######可删的系统文件
- %SystemRoot%\TEMP	(虽然没多少Windows软件会写缓存到这里,不过这个路径下的文件理论上来说没必要长期存在)
- %SystemRoot%\ccmcache	(MECM从分配点下载过来的部署包文件,反正可以反复下载)
- %SystemRoot%\SoftwareDistribution	(Windows更新过程中会把更新文件解压到这里,没了再下载,微软的排错也是删除这里)
- $Windows.~BT	(Windows功能更新解压的路径,没了再下载/解压)
- 还原点	(客户端操作系统本身坏的可能性不大,所以没有存在的必要而且默认禁用)


###解决：
按照上面的顺序依次
######1. %SystemRoot%\TEMP:

因为就是个给软件放垃圾的地方,直接删掉理论上来说就可以
Windows命令提示符删除文件夹的的操作是 [rd | Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/rd)
参数就三个,路径 /S /Q 全都需要

![dir](https://s3.bmp.ovh/imgs/2022/02/936208fbdf9aef3e.png)<br>
***D分区下有个test1,里面有个test2***

![rd](https://s3.bmp.ovh/imgs/2022/02/0529aa4e28273922.png)<br>
***如上图,执行完后就删掉了 test2 这个文件夹***

考虑到其他需要删除的文件可能需要本地管理员 或者 需要操作注册表（后面的操作）,因此通过PowerShell执行策略的 bypass会方便一些,之后的所有操作都会尽量使用PowerShell的语句
看看rd在powershell里有没有同样的功能
```
Get-help rd
```
![gethelp](https://s3.bmp.ovh/imgs/2022/02/9ce64572d7072234.png)<br>
有的,在PowerShell里rd是remove-item的别名,但是用法不一样了.明显可以发现参数多了.<br>
看看用法[Remove-Item (Microsoft.PowerShell.Management) - PowerShell | Microsoft Docs](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/remove-item?view=powershell-7.1)<br>
需要一个路径 -path <br>
递归一下下面所有的文件夹 -Recurse <br>
顺便强制一下防止弹确认 -Force <br>
应该足够了
![remove](https://s3.bmp.ovh/imgs/2022/02/83f901acad2de63b.png)<br>
<br>***还是test2 ,执行一下***
![remove](https://s3.bmp.ovh/imgs/2022/02/0f9a32c4627cd499.png)<br>
<br>***可以,没了***


用法搞定
```
Remove-Item -Path "%SystemRoot%\TEMP" -Recurse -Force 
```
测试一下实际路径

![remove](https://s3.bmp.ovh/imgs/2022/02/bfda479286f974a2.png)<br>
***报错了,因为该路径不存在***

powershell引用环境变量的时候不能直接引用,得带上$env: 更换一下,顺便TMP这个文件夹留着,下面的文件夹都不要,改一改

```
Remove-Item -Path "$env:SystemRoot\TEMP\*" -Recurse -Force
```
![remove](https://s3.bmp.ovh/imgs/2022/02/31ef27b4f1ab62de.png)<br>
***可以是可以，就是报错(被占用的文件)太多***

我不要看到,那就再加个 **-ErrorAction silentlycontinue** 最后再试一试

```
Remove-Item -Path "$env:SystemRoot\TEMP\*" -Recurse -Force -ErrorAction silentlycontinue
```
![remove](https://s3.bmp.ovh/imgs/2022/02/7b1183b3f95a8eba.png)<br>
***成了***


######2. 同理 %SystemRoot%\ccmcache 也可以这么操作

######3. 这个不行 %SystemRoot%\SoftwareDistribution

因为Windows更新(wuauserv) 这个服务会一直使用这个文件夹
得在操作删除前停止这个服务 **Stop-Service -Name wuauserv -Force**
然后再启动服务 **Start-Service -Name wuauserv**
停服务后面最好再加个等待5秒 防止有的客户端太老性能差

######4. $Windows.~BT

直接删除也可以的.但是在整理这个脚本过程中发现
Windows功能更新过程中会有很多没有必要存在的文件被创建,为了解决这个问题Windows自带的磁盘清理功能是非常强大的[cleanmgr | Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cleanmgr)
除了一些账号本身产生的垃圾以外,包括清理功能更新产生的文件在内,前面的一些操作完全可以被磁盘清理覆盖进去.
![cleanmgr](https://s3.bmp.ovh/imgs/2022/02/5955e1a5f76566d0.png)


看一下用法
[cleanmgr | Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cleanmgr)
需要指定(其实就是所有)删除项,还需要不弹出用户确认项(静默)
**/sageset:n /sagerun:n /verylowdisk**
关于指定项,用法页面里没有详细介绍,但是另外一个微软页面([Automating Disk Cleanup tool - Windows Server | Microsoft Docs](https://docs.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/automating-disk-cleanup-tool))中有介绍

>/sageset:n - This switch displays the Disk Cleanup Settings dialog box and creates a registry key to store the settings you select. The n value is stored in the registry and allows you to specify different tasks for Disk Cleanup to run. The n value can be any integer value from 0 to 65535. To get all the available options when you use the /sageset switch, you may need to specify the drive letter that contains the Windows installation.
>For more information, see Registry key information.

再跳转到具体的注册表介绍页看看怎么指定 [Automating Disk Cleanup tool - Windows Server | Microsoft Docs](https://docs.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/automating-disk-cleanup-tool#registry-key-information)

>Each of the modified registry sub keys may contain a REG_DWORD type registry value StateFlagsNNNN, where NNNN is the number n specified in the switch. For example, after you run the cleanmgr /sageset:9 command, a registry value Stateflags0009 is added. The registry value can be set as one of the following values.<br>If the option box is not selected, the value is 00000000.<br>**If the option box is selected, the value is 00000002**.


到对应的注册表看一看,还是挺多的,既然都被列在cleanmgr里,那都是不会影响操作系统正常运行的文件,可删.

![reg](https://s3.bmp.ovh/imgs/2022/02/1b1fa028b25855b4.png)

```
$REG = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches"
set-itemproperty -path "$REG\Windows Upgrade Log Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Windows ESD installation files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Windows Error Reporting Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Windows Defender" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\User file versions" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Upgrade Discarded Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Update Cleanup" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Thumbnail Cache" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Temporary Sync Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Temporary Setup Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Temporary Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\System error minidump files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\System error memory dump files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Setup Log Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\RetailDemo Offline Content" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Recycle Bin" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Previous Installations" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Old ChkDsk Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Offline Pages Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Language Pack" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Internet Cache Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\DownloadsFolder" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Downloaded Program Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Diagnostic Data Viewer database files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Device Driver Packages" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Delivery Optimization Files" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\D3D Shader Cache" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Content Indexer Cleaner" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\BranchCache" -name StateFlags0001 -type DWORD -Value 2
set-itemproperty -path "$REG\Active Setup Temp Folders" -name StateFlags0001 -type DWORD -Value 2
cleanmgr /sagerun:1 /VERYLOWDISK
```
这不行,太难看了.缩短点
思路就是获取VolumeCaches下面所有目录,然后每个目录里新建一个键
由于经验比较浅,本应该很容易的一件事情结果我在这个坑里爬了将近4小时.下面就只记录几个关键的瞬间

第一步：
```
$REGS = "Windows Upgrade Log Files" ,"Windows ESD installation files" ,"Windows Error Reporting Files" ,"Windows Defender" ,"User file versions" ,"Upgrade Discarded Files" ,"Update Cleanup" ,"Thumbnail Cache" ,"Temporary Setup Files" ,"Temporary Sync Files" ,"Temporary Files" ,"System error minidump files" ,"System error memory dump files" ,"Setup Log Files" ,"RetailDemo Offline Content" ,"Recycle Bin" ,"Old ChkDsk Files" ,"Previous Installations" ,"Offline Pages Files" ,"Language Pack" ,"Internet Cache Files" ,"DownloadsFolder" ,"Downloaded Program Files" ,"Diagnostic Data Viewer database files" ,"Device Driver Packages" ,"Delivery Optimization Files" ,"D3D Shader Cache" ,"Content Indexer Cleaner" ,"BranchCache" ,"Active Setup Temp Folders"
foreach($REG in $REGS){
  set-itemproperty -path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\$REG" -name StateFlags0002 -type DWORD -Value 2
}
```
先反向确定需要什么样的变量才能成功操作

第二步：

```
$Registory = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches"
$Full = (get-childitem $Registory).Name
```

拿到对应的路径,并且转换成字符
方便进行字符处理

第三步：
```
$Registory = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches"
$Full = (get-childitem $Registory).Name
foreach($single in $Full){
  $dir = $single.split('\')[7]
}
```
处理字符,拿到第一步中成功测试的所需变量
最终就是下面这段

```
$Registory = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches"
$Full = (get-childitem $Registory).Name
foreach($single in $Full){
  $dir = $single.split('\')[7]
  set-itemproperty -path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\$dir" -name StateFlags0001 -type DWORD -Value 2
}
cleanmgr /sagerun:1 /VERYLOWDISK
```

既然用到了cleanmgr,那对应的磁盘清理这个软件里还有一些别的删除功能吗,看一下
![cleanmgr](https://s3.bmp.ovh/imgs/2022/02/441d8896a1a52e21.png)




程序和功能就是常说的卸载,下面的系统还原和卷影复制这个得注意一下.
首先这是个什么功能,贴上微软的简介页 [Volume Shadow Copy Service | Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)
简单地理解成Windows提供的快照就可以.一般人可能觉得我或者公司员工没用这个服务,但恶心的一点是,微软开启这个服务对应的程序(进程?)叫做创建还原点,而且有的第三方软件创建备份时就是使用这个服务.理论上来说Windows 客户端没有这种本地备份需求（文件可以通过Onedrive等同步盘,用户备份可以做在on-premise的AD上）.虽然默认不启用,但是考虑到还原点这个名词让人感觉很安心,不能控制员工自己手动开启,考虑到几乎是没有意义的浪费本地磁盘,所以可以删
![cleanmgr](https://s3.bmp.ovh/imgs/2022/02/4f7f462b848a79f0.png)

管理工具的简介 [vssadmin | Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/vssadmin)
一共就5个参数,全都删掉且不要交互
**vssadmin delete shadows /all /quiet**


整合一下
YaoNiMing3k.ps1
```
Remove-Item -Path "$env:SystemRoot\TEMP\*" -Recurse -Force -ErrorAction silentlycontinue
Remove-Item -Path "$env:%SystemRoot%\ccmcache\*" -Recurse -Force -ErrorAction silentlycontinue

Stop-Service -Name wuauserv -Force
Start-Sleep -Seconds 5
Remove-Item "$env:SystemRoot\SoftwareDistribution" -Recurse -Force - ErrorAction SilentlyContinue
Start-Service -Name wuauserv

$Registory = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches"
$Full = (get-childitem $Registory).Name
foreach($single in $Full){
  $dir = $single.split('\')[7]
  set-itemproperty -path "HKLM:\Software\Microsoft\Windows\CurrentVersion\Explorer\VolumeCaches\$dir" -name StateFlags0002 -type DWORD -Value 2
}
cleanmgr /sagerun:2 /VERYLOWDISK

vssadmin delete shadows /all /quiet
```
至此脚本部分就已经完成了.试一试效果

![ps1](https://s3.bmp.ovh/imgs/2022/02/730d4515f6e268b9.png)
***Before***

![ps1](https://s3.bmp.ovh/imgs/2022/02/02bb8dcea14c21e2.png)
***After***

还是有一些没用的文件存在的。
当然直接运行也是可以的,但是我相信我们集团员工不可能所有人都愿意/有能力去执行.况且这些语句完全没有做例外处理,万一手动执行的过程中遇到过error,员工看到个error我就百口难辨了.
既然本文是MECM部署DiskcleanUp,那么最后一步就是把脚本通过MECM部署到对应的集合.并且之所以用PowerShell的写法是因为用MECM(SCCM)去部署PowerShell非常的无脑,强力推荐!


那么接下来就进入MECM上的操作,远程桌面到主站点,打开控制台

![ps1](https://s3.bmp.ovh/imgs/2022/02/f695a8c2157e6b5c.png)
***把脚本放好位置***

![deploy](https://s3.bmp.ovh/imgs/2022/02/6d234131e49f1b7c.png)
***创建部署包***


路径需要是UNC格式

就是个只包含一个脚本的部署包,没有程序

完成后
  把部署包Distribute一下到所有的分配点上

接下来创建任务序列

就是个运行脚本的序列,从空模板里选中脚本就可以

下一步跳过 我全都自己编辑


编辑,Add一个 运行脚本的序列


选择上面创建的部署包,脚本名那里填脚本的文件名,执行策略确认是Bypass (很重要,不修改客户端本身执行策略的同时,也能让客户端执行本次非签名脚本)

接下来deploy 这个任务序列

选一下要部署的集合


对于连[磁盘清理]这个自己家里电脑也一样可以操作的步骤都不愿意进行操作的员工来说,让他们打开[软件中心]点更不现实了.改成 [必须]




有关计划,今年不会去软件中心里操作,明年也不可能操作,所以干脆静默循环地去执行.
当然不希望执行一次就结束,选 [总是重试]


既然决定静默了,取消勾选任务序列进程显示
到了安排的时间后,运行任务序列



放个设备到对应集合里,等一等
等的过程中先看看原本C分区还剩下多少



软件中心里也来了

点进去看一看,因为第一次到达这个设备,部署包里的脚本还没下载过来所以运行不了脚本,第一次失败.
之前设置的重试策略是 [总是重试] 所以等一等吧



到点了,再看看C分区
成了,至此MECM部署DiskcleanUp结束

----

最终的目的是让电脑可以避免因员工使用习惯造成系统分区被浪费 导致无法进行功能更新
除了部署清理以外另外一个就是从根本上杜绝员工写文件到系统分区。
下一步就计划是在OSD阶段把对应文件夹移动到其他非操作系统分区(当前时间点操作会可能会影响业务连续性)
