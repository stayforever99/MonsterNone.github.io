---
title: OpenWrt Mwan3多线多拨
date: 2019-06-20 21:23:57
tags:
    - OpenWrt
    - 教程
    - 路由器
summary: 从理论上增加可用带宽。
desc: 
    - OpenWrt
    - 多拨
    - Mwan3
    - 路由器
    - 负载均衡
---

最近给Phicomm K2刷了OpenWrt (原来OpenWrt驱动有问题，一直用的官改) ，折腾了好一会儿没弄到什么实用的功能 (不实用功能包括IPv6) 。正好家中有两条宽带，一条联通宽带、一条广电宽带，广电的一直吃灰但好歹花了钱也要用上2333。

```
斐讯K2 + 官方OpenWrt

联通宽带50M，双11办的，￥11.11；

广电宽带10M，和电视一起优惠的，￥50。
```

### 安装Mwan3
首先要给OpenWrt装上Mwan3，官方固件不自带 (好像？忘了。。) ，移植的固件一般都有。

在Luci `系统` - `软件包` 里面，搜索 `mwan`，安装 `luci-app-mwan3`、`mwan3`、`luci-i18n-mwan3-zh-cn` (中文支持，可选) 

{% asset_img 搜索.png 搜索.png %}

> 如果你从来没有安装过可能需要首先更新软件包列表，可以在 `配置` 中先改成国内的源：
>  
> 将 `发行版软件源` 中的每一行 `/releases/版本/targets/......` 前的网址改为 `http://mirrors.tuna.tsinghua.edu.cn/lede` 
>
> 例如：`src/gz openwrt_core http://mirrors.tuna.tsinghua.edu.cn/lede/releases/18.06.2/targets/ramips/mt7620/packages`

> 如果你的OpenWrt是英文版，那顺便安装下 `luci-i18n-base-zh-cn`

都安装好了之后刷新下界面，在 `网络` 里面会出现 `负载均衡`

{% asset_img show.png show.png %}

### 配置VLAN

首先把需要多拨的两根 (三根、四根、无数根) 网线插上。一根插在原本的WAN口上，另外一根插在LAN口。

打开 `网络` - `交换机` 界面，这时候就可以清楚地看到我们的两根网线。

> 我这里是多插了一个树莓派

{% asset_img vlan.png vlan.png %}

这时候通过原本显示的WAN口和LAN口可以轻松分辨出端口对应的宽带。例如我这里WAN是联通，LAN1是广电。

如果你分不清谁是谁的话，可以拔掉，刷新，记下 `端口状态`，插上，对比新的状态。

好，知道了谁是谁之后就简单了。

新建一个VLAN规则，名字就写3。

**这时候到了重头戏，划分VLAN。**

网上一些教程写的不是很清楚，一头雾水；这里我教你一个简单的原则。

***所有的实际LAN口在同一个 (LAN口专用的) VLAN中 `未标记`***

***所有的实际WAN口各自独立在不同的VLAN中 `未标记`***

***每一条VLAN的CPU端口永远为 `已标记`***

也就是说：

**把第二 (三、四...) 条宽带插的端口从LAN口的VLAN中移除 (设置为 `关`)**

**新建一个VLAN，在此VLAN中第二条宽带的LAN口为唯一的 `未标记` 端口**

解释一下：

1. 首先由于防火墙的需要，LAN口和WAN口不能放在一起，LAN口可以互联
2. 由于相同的协议存在相互干扰，所以WAN口不能放在一起 (我也不是很清楚，在实际使用中我把两个WAN口PPPoe和DHCP放在一起没发生问题，就先这么理解吧) 


就是这么简单。具体操作如下：

> 原有的规则中
>> WAN为 `未标记`，其他为 `关` 的那条，对应原本WAN口，无需改动
>
>> WAN为 `关`，其他为 `未标记` 的那条，对应原本的LAN口，将插宽带的LAN口从中移除，设置为 `关`

> 新建的规则中
>> CPU (eth0) 设置为 `已标记`
> 
>> 你插在LAN口的宽带的那个端口，设置为 `未标记`
>
>> 其他口都默认关就行

最终如图：

> LAN1 和 WAN 是宽带，LAN4是树莓派 (LAN口设备) 

{% asset_img vlan-2.png vlan-2.png %}

### 设置拨号/DHCP连接网络

设置好了VLAN之后，需要将两个新的WAN口联网。

`网络` - `接口` ，编辑原来的WAN，设置一个原WAN口宽带的联网方式；

在 `高级设置` 中，设置网关跃点为 `10`

{% asset_img jump.png jump.png %}

新建一个接口，名称随意，例如 `WAN2`，协议设置为第二条宽带的联网方式，`包含以下接口` 设置为新建的那条VLAN (`eth0.3`) 

{% asset_img new.png new.png %}

然后对应设置一下新接口的联网；网关跃点设置为20 (每一条宽带的跃点都不要相同)。

保存&应用。回到接口概况看下两个宽带是不是都成功联网了 (分配到了IP地址) 。

{% asset_img interface.png interface.png %}

### Mwan3设置

都成功联网了之后，就可以设置Mwan3负载均衡使用不同的网络了。

在 `网络` - `负载均衡` - `接口` 中，先把所有的删了。

新建，名称和你宽带1的接口名称一致 (在 `网络` - `接口` 里面显示的名字)；

里面的配置没什么特别要注意的，根据名称自己看情况设置，记得把 `启用` 设为 `是` 就好。

同理新建宽带2的接口WAN2。

{% asset_img mwan.png mwan.png %}

看一下跃点数和你之前设置的跃点数是否一致，一致就说明接口名称填写正确了。

在 `成员` 中，删除所有，新建，名称可以随意了，例如 `wan1_m1` `wan2_m1`。

选择对应的接口，跃点数留空，比重可以根据你的宽带不同的带宽来填写。比如50M的宽带比重为5，10M的比重为1。

{% asset_img member.png member.png %}

在 `策略` 中，删除所有，新建一个作为负载均衡使用，名称为balance (或者其他)。把你新建的成员加上。备用设为 `默认 (使用主路由)`

{% asset_img stra.png stra.png %}

在 `规则` 中，删除所有，新建一个，名称随意，如果你不知道啥是啥那就把地址/端口都留空；`协议` 设为 `tcp` (大部分负载都是TCP)；`分配的策略` 选择你刚才新建的负载均衡策略；`粘滞模式` 设为 `关` ；其他默认就可。

{% asset_img rule.png rule.png %}

## 终于设置完了！

在 `状态` - `负载均衡` 里面看下几个WAN口是不是都在线；`详细` 中看下 `policies` 是不是想要的效果。

{% asset_img status.png status.png %}

### 测试一下

主要是测试一下你有没有把原本好用的网络搞坏233333

```bash
# pi @ raspberrypi in ~ [23:38:24] 
$ python3 speedtest.py
Retrieving speedtest.net configuration...
Testing from China Unicom Liaoning (112.x.x.x)...
Retrieving speedtest.net server list...
Selecting best server based on ping...
Hosted by China Telecom Wuhan Branch (Wuhan) [646.43 km]: 31.35 ms
Testing download speed................................................................................
Download: "57.37 Mbit/s"
Testing upload speed......................................................................................................
Upload: 9.94 Mbit/s

# pi @ raspberrypi in ~ [23:39:11] 
$ python3 speedtest.py
Retrieving speedtest.net configuration...
Testing from China Unicom Liaoning (112.x.x.x)...
Retrieving speedtest.net server list...
Selecting best server based on ping...
Hosted by China Telecom Wuhan Branch-2 (Wuhan) [646.43 km]: 31.619 ms
Testing download speed................................................................................
Download: "76.07 Mbit/s"
Testing upload speed......................................................................................................
Upload: 8.06 Mbit/s
```