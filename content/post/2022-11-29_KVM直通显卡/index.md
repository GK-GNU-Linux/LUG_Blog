---
title: "KVM虚拟机GPU直通实验，为游戏助力"
description: 给KVM虚拟机加双翅膀
image: neofetch.png
date: 2022-11-29T02:45:32+08:00
categories:
    - 技术周报
---
作者: Twac

## 介绍

KVM(Kernel-based Virtual Machine)是一个开源的系统虚拟化模块，选择KVM的原因是因为自己的主力系统的Linux。但是出于不可抗力的原因（臭打游戏的）也不能完全脱离Windows，起初笔记本装了双系统，但是来回切换还是太麻烦了，虽然Linux有Wine、Proton等一些兼容层，但是效果还是不尽人意。

于是就想在Linux上运行一台Windows虚拟机，但是KVM虚拟机配合QEMU的QXL图形渲染还是达不到（畅玩游戏）要求的。所以要利用KVM PCIe直通功能，让虚拟机独占高性能显卡，直接加载显卡对应的驱动，和显卡直接通信，给它加一双翅膀，拥有和物理机几乎同样的渲染性能。

实验环境：

![neofetch](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/neofetch.png)


- 虚拟机镜像：Windows 10 LTSC 2019
- 主机：神舟战神G7-CT7NK
- 套件：libvirt + qemu + virt-manager(GUI)

## 查看笔记本架构

目前主流的笔记本架构是`MUXed`和`MUXLess`。相比`MUXLess`来说，`MUXed`架构的笔记本更容易直通，因为它采用的是核显/独显切换工作方式。而`MUXLess`架构是核显输出，独显渲染。
```shell
$ lspci | grep VGA #查看架构
```
如果独显以3D Controller开头，那么属于`MUXLess`；如果独显以VGA Controller开头，并且由于一个VGA核显输出(Intel/AMD)，那么属于`MUXed`。

![lspci](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/lspci.png)

## 开启VT-d虚拟化以及IOMMU模块

BIOS开启VT-d，并在宿主机内核参数添加 `iommu=on,iommu=pt` ，其中 `iommu=pt` 为了防止不能直通的设备造成错误。  

```shell
$ dmesg | grep -e DMAR -e IOMMU  #检查 dmesg 以确认 IOMMU 已经被正确启用
```
 
>注意： 
> - IOMMU 是 Intel VT-d 和 AMD-Vi 的通用名称。
> - VT-d 指的是直接输入/输出虚拟化(Intel Virtualization Technology for Directed I/O)，不应与VT-x(x86平台下的Intel虚拟化技术，Intel Virtualization Technology)混淆。VT-x 可以让一个硬件平台作为多个“虚拟”平台，而 VT-d 提高了虚拟化的安全性、可靠性和 I/O 性能。

![iommu](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/iommu.png)

## 添加vfio模块屏蔽GPU

将`vfio_pci vfio vfio_iommu_type1 vfio_virqfd`添加到initramfs中，重新生成内核并重启。然后运行以下脚本，查看设备id.

```bash
#!/bin/bash #查看是否生效
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;
```
可以看到group1对应的是本机独显设备
![iommu_group](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/group.png)

将以上设备id添加到`/etc/modprobe.d/vfio.conf`中启用vfio-pci驱动隔离GPU，编辑内容如下： 

```config
/etc/modprobe.d/vfio.conf
----------------------------------------------------------------------
options vfio-pci ids=8086:1901,10de:2191,10de:1aeb,10de:1aec.10de:1aed
```

## 创建虚拟机

使用virt-manager(GUI)创建一台虚拟机：  
- 架构：Q35
- 固件：UEFI
- CPUs：host-passthrogh
- 内存：8192(8G)
- 启动盘&数据盘：驱动使用virtio
- 网络：NIC/NAT，驱动依然virtio

默认先用QEMU的QXL渲染，安装好[virtio驱动](https://github.com/virtio-win/virtio-win-pkg-scripts)

## 显卡直通

先确定一下显卡 `device-id` 和 `vendor-id` 和 PCI 总线 ID 是否一致，如果不一致需要将 ID 修改为一致的，终端运行

```shell
$ lspci -nnk | egrep -A3 "VGA|3D" #检查ID
```

![subSystem](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/subSystem.png)

如图我的独显的vendor-id是`10de`，device-id是`2191`;  
PCI系统总线的 vendor-id 是`1558`，device-id 是`8550`

拿到ID后在virt-manager(GUI)直通独显设备，注意要将IOMMU组里的全部设备直通进去

![addPCIe](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/addPCIe.png)

> 由于NVIDIA今年放开了虚拟化限制，提取显卡vbios是一个可选项，不需要绕过检测。

此时如果开机，查看设备管理器会多出一个`Microsoft基本显示适配器`，说明已经识别到设备了，但是点开一看报了代码`31`，无法加载驱动。需要在虚拟机xml配置PCIe总线id。

首先在xml配置的第一行`<domain>`标签配置qemu命名空间（详情看文末虚拟机xml配置）`<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">`

然后在虚拟机xml配置尾部添加前面记录的PCIe总线id`(注意进制转换)`
```xml
<qemu:override>
    <qemu:device alias="hostdev0">
        <qemu:frontend>
            <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="5464"/>
            <qemu:property name="x-pci-sub-device-id" type="unsigned" value="34128"/>
        </qemu:frontend>
    </qemu:device>
</qemu:override>
```
如果你的qemu版本低于7.0那么用如下配置即可

```xml
<qemu:commandline>
    <qemu:arg value='-set'/>
    <qemu:arg value='device.hostdev0.x-pci-sub-vendor-id=5464'/>
    <qemu:arg value='-set'/>
    <qemu:arg value='device.hostdev0.x-pci-sub-device-id=34128'/>
</qemu:commandline>
```

做完这些重启虚拟机，驱动应该是正常运行，此时算是大功告成。直通显卡后，开启虚拟机后要切换独显输出，因为宿主机用的是核显，此时可以选择外接一个显示器（笔记本的HDMI/DP接口使用的是独显），或者利用`libvirt hooks`钩子函数让独显抢占当前笔记本显示器。

至此显卡基本直通到此结束，还是趟了很多坑的。小小记录一下。
![效果](https://file.feldan1.top:9443/d/misc/blog/2022-11-29_KVM%E7%9B%B4%E9%80%9A%E6%98%BE%E5%8D%A1/img/view.png)

>## 参考链接
>1. [ArchWiki - PCI passthrough via OVMF (简体中文)](https://wiki.archlinux.org/title/>PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
>2. [ArchWiki - Intel_GVT-g](https://wiki.archlinux.org/title/Intel_GVT-g)
>3. [ArchWiki - Libvirt](https://wiki.archlinux.org/title/Libvirt)
  
## XML完整配置

```xml
<!-- 第一行`<domain>`标签配置qemu命名空间`<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">` -->
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm" 
  <name>win10</name>
  <uuid>7c121caf-30db-4a3a-82f6-d59ca87b802f</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">12582912</memory>
  <currentMemory unit="KiB">12582912</currentMemory>
  <vcpu placement="static">8</vcpu>
  <os>
    <type arch="x86_64" machine="pc-q35-7.0">hvm</type>
    <loader readonly="yes" type="pflash">/usr/share/edk2-ovmf/x64/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
    </hyperv>
    <vmport state="off"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="4" threads="2"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="writeback" discard="ignore"/>
      <source file="/home/twac/VM/storage/win10.qcow2"/>
      <target dev="vda" bus="virtio"/>
      <boot order="1"/>
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="writeback" discard="ignore"/>
      <source file="/home/twac/VM/storage/DATA.qcow2"/>
      <target dev="vdb" bus="virtio"/>
      <address type="pci" domain="0x0000" bus="0x0a" slot="0x00" function="0x0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/twac/VM/ISO/Win10_21H2_Chinese(Simplified)_x64.iso"/>
      <target dev="sdb" bus="sata"/>
      <readonly/>
      <boot order="2"/>
      <address type="drive" controller="0" bus="0" target="0" unit="1"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/home/twac/VM/ISO/virtio-win-0.1.208.iso"/>
      <target dev="sdc" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="2"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="9" port="0x18"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="10" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="10" port="0x19"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x1"/>
    </controller>
    <controller type="pci" index="11" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="11" port="0x1a"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x2"/>
    </controller>
    <controller type="pci" index="12" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="12" port="0x1b"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x3"/>
    </controller>
    <controller type="pci" index="13" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="13" port="0x1c"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x4"/>
    </controller>
    <controller type="pci" index="14" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="14" port="0x1d"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x5"/>
    </controller>
    <controller type="pci" index="15" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="15" port="0x8"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </controller>
    <controller type="pci" index="16" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x0b" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:98:77:e9"/>
      <source network="default"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x0c" slot="0x00" function="0x0"/>
    </interface>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <sound model="ich6">
      <address type="pci" domain="0x0000" bus="0x10" slot="0x01" function="0x0"/>
    </sound>
    <audio id="1" type="none"/>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <driver name="vfio"/>
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <rom bar="on" file="/var/tmp/1660ti.rom"/>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
      </source>
      <rom bar="on" file="/var/tmp/1660ti.rom"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x2"/>
      </source>
      <rom bar="on" file="/var/tmp/1660ti.rom"/>
      <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x3"/>
      </source>
      <rom bar="on" file="/var/tmp/1660ti.rom"/>
      <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="usb" managed="yes">
      <source>
        <vendor id="0x046d"/>
        <product id="0xc332"/>
      </source>
      <address type="usb" bus="0" port="1"/>
    </hostdev>
    <hostdev mode="subsystem" type="usb" managed="yes">
      <source>
        <vendor id="0x05ac"/>
        <product id="0x024f"/>
      </source>
      <address type="usb" bus="0" port="2"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x00" slot="0x1f" function="0x0"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x10" slot="0x02" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x00" slot="0x1f" function="0x3"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x10" slot="0x03" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x00" slot="0x1f" function="0x4"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x10" slot="0x04" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x00" slot="0x1f" function="0x5"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x10" slot="0x05" function="0x0"/>
    </hostdev>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
  <qemu:override>
    <qemu:device alias="hostdev0">
      <qemu:frontend>
        <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="5464"/>
        <qemu:property name="x-pci-sub-device-id" type="unsigned" value="34128"/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
</domain>
```












