---
title: HomeIDC搭建记录
tags: ['网络','电子垃圾']
abbrlink: 42f9829b
date: 2024-02-08 16:32:00
---

# 基础网络环境介绍
家中拉了两家运营商的三条宽带：联通家宽×1、联通专线×1、电信家宽×1，同时由于整栋楼只有我一人用联通，因此独享整个OLT（也很搞笑的一根联通进户光纤拨成功我两条联通宽带）
下图是家中基础网络拓扑
![HOMEIDC拓扑](https://data.xchub.cn/HOMEIDC拓扑.png)
<!-- more -->
# 硬件部分
## 服务器（软路由）硬件
最初是想要做到链路聚合同时负载均衡，因此CPU选择了2× E5-2689，主板则是浪潮的服务器板，带ipmi方便远程管理和调试；原本整了48G的内存，因为散热风道问题被我直接干炸20G，现在只能说凑合用着了。单独增加了一张光卡直通进RouterOS，用于和光交换机互联。
不过最后做的是策略路由，服务器的性能还是有点浪费，CPU日常跑不到3%，内存也就用12G，只有电费嘎嘎高。
## 交换机
本想一步到位上Mikrotik的CRS309，没想到设备停产了，原价1900的交换机现只要再加4000元运费就可以抱回家。贫穷如我选择一半不到的TPLINK ST-5008F v2，电口交换机则是Htroy赠送的24口网件。
### TP-Link的奇葩问题
1. console几乎是不可用的状态，就成功连进去过一次，聊胜于无。web界面还可以，能完成基础的配置。
2. 端口自协商速率如果更改，会同时更改相邻的端口，即：1口和2口会同时更改端口速率，以此类推
3. 1G和2.5G不共存，即端口仅能支持1G/10G自协商和2.5G/10G自协商。虽然可以理解，但是还是多少有点遗憾，当然2.5G我也一直觉得是个有点畸形的速率。
## PONStick
光猫的性能实在太过于遗憾，而且光猫的功能也非常臃肿，就选择了使用猫棒，闲鱼开车后50/根的大型灵车（也算是目前网络环境中最不稳定的一环，容易过热）
广东地区联通和电信均校验Loid、光猫SN和光猫MAC，在自适应配置中更改即可，双宽带及以上接入的情况下，需要将不同宽带的vlan错开，不然会爆炸的。
Update：Loid记得在安装时提前问装维小哥拿，目前电信光猫管理员密码已确认会自动变更，原默认管理员密码`nE7jA%5m`无法进入。

# 软件部分
由于设备的性能还是挺强的，那么不虚拟化肯定是浪费了，在ESXI驱动有问题的情况下，最终还是回归PVE的怀抱。
路由器系统部分原本则是信心满满地使用OpenWRT，用不了2周原地推倒重建，改用Router OS，这下倍感丝滑。

# 配置部分
常规拨号和vlan配置就暂且略过。
## DDNS
家宽有公网IP的选手会发现家宽的公网并不固定，因此想要随时能访问家里的设备还是需要DDNS自动将更新的公网IP同步到域名托管机构。
我的域名目前挂靠在CF，原先是从Prokbun购买，以下是Router OS的DDNS脚本（感谢Htory）
Cloudflare的ddns，空的相关参数可以在F12中找到：
```cfg
:global cfu do={\
    :local date [/system clock get date];\
    :local time [/system clock get time];\
    :local cfi "";\
    :local cfr "";\
    :local cfe "";\
    :local cfk "";\
    :local cfd "";\
    :local currentIP [/ip address get [/ip address find interface=] address];\
    :local cfa [:pick $currentIP 0 [:find $currentIP "/"]];\
    :local cfp false;\
    /tool fetch mode=https\
    http-method=put\
    url="https://api.cloudflare.com/client/v4/zones/$cfi/dns_records/$cfr"\
    http-header-field="content-type:application/json,X-Auth-Email:$cfe,X-Auth-Key:$cfk"\
    http-data="{\"type\":\"A\",\"name\":\"$cfd\",\"content\":\"$cfa\",\"ttl\":120,\"proxied\":$cfp,\"comment\":\"LastReport:$date $time\"}"\
    output=none\
}
:delay 1
$cfu
```

Porkbun的ddns：
```cfg
:global porkbun do={\
    :local record "";\
    :local domain "";\
    :local sk "";\
    :local ak "";\
    :local name "";\
    :local currentIP [/ip address get [/ip address find interface=] address];\
    :local address [:pick $currentIP 0 [:find $currentIP "/"]];\
    /tool fetch mode=https\
    http-method=post\
    url="https://porkbun.com/api/json/v3/dns/edit/$domain/$record"\
    http-header-field="content-type:application/json"\
    http-data="{\"secretapikey\":\"$sk\",\"apikey\":\"$ak\",\"type\":\"A\",\"name\":\"$name\",\"content\":\"$address\",\"ttl\":\"600\"}"\
    output=none\
}
:delay 1
$porkbun
```

写完脚本后设置定时任务：
```cfg
/system script run xxxx
```


## Hairpin NAT
在给NAS对外开放的服务套上域名后，发现在内网环境中无法直接通过域名访问到，后续发现是NAT造成的来回路径不一致，即：访问的地址是公网地址，但是服务器回你的时候是私网地址，所以路径不一致，会话建立不起来。
具体解决可以参考[这篇博文](https://vqiu.cn/rot-hairpin-nat-example/)


## 旁路网关
部分服务需要覆盖多数设备但是主路由又不适合运行，可以将这部分服务改到旁路网关中，将该旁路网关设置为下联设备的网关，旁路网关自身的网关则设置为主路由，且均可设置在同一网段内，这样可以在不大幅度更改原有网络结构的情况下快速实现新功能。