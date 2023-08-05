# 手机运行Docker: 从修改内核到刷入原生Linux


<meta name="referrer" content="no-referrer" />



## 背景

最近收拾东西翻出了抽屉里吃灰的小米6。小米6当年可以说是神机一部，最好的835遇到了最好的MIUI9。如今放在抽屉里吃灰实在可惜，想着拿来做点什么让它继续发挥余热。

随后就萌生了一个想法：在手机上跑Docker，这样的话就可以用到很多打包好的Docker应用了。后来发现有这个想法的人不止我一个，有很多大佬已经实现了。原生安卓无法直接运行Docker的原因是：安卓虽然基于Linux，但是内核阉割了很多东西，很多Docker相关的的内核选项没有开启，所以需要通过刷机来进行修改。

本人总共尝试了两种方案：一种是重新编译安卓内核，开启对应选项。不过在本人的小米6，Linux内核4.4版本，LineageOS 19.1上失败了；另一种是直接刷入原生的Linux系统，成功启动了Docker。在这里跟大家分享一下刷机的过程。

## 方案一：重新编译安卓内核

如果可以通过修改安卓内核来开启Docker应该是最理想的方案：这样可以在保留手机原有功能架构的基础上来提高我们的可玩性。老外写过一篇详细的教程，可以按照这个来：https://gist.github.com/FreddieOliveira/efe850df7ff3951cb62d74bd770dce27

不过很遗憾，最后这种方案失败了，一直出现报错。找了半天也没有找到解决办法，希望知道原因的小伙伴告知我一下。

### 准备工作

首先要找一份第三方维护的你的手机内核的源码，如lineageOS，PixelExperience等。这些内核代码热度较高，更新频繁，有什么bug马上就被修复了，编译的时候成功率较大。

另外注意，如果是小米手机，最好不要用小米官方github上的内核。本人亲身体会，编译过程不仅一堆BUG，刷入系统后还开不了机。后来看到看雪的帖子，很多人也遇到了同样的情况：https://bbs.pediy.com/thread-262263.htm

经过一番查找对比，最后选择以lineageOS维护的小米6(sagit)的内核源码作为基础：https://github.com/LineageOS/android_kernel_xiaomi_msm8998



```plain
git clone https://github.com/LineageOS/android_kernel_xiaomi_msm8998 --depth=1
```

sagit是小米6的手机代号，这个代号独一无二，可以百度搜一下自己手机的对应代号。

### 修改内核

我们首先不做修改，编译一次看报不报错。

```plain
cd ./android_kernel_xiaomi_msm8998
sudo apt install build-essential openssl pkg-config libssl-dev libncurses5-dev pkg-config minizip libelf-dev flex bison  libc6-dev libidn11-dev rsync bc liblz4-tool  
sudo apt install gcc-aarch64-linux-gnu dpkg-dev dpkg git

export ARCH=arm64
export SUBARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make O=out sagit_defconfig
make O=out -j$(nproc)
```



#### error：CROSS_COMPILE_ARM32 not defined or empty

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668348685447-ef8940c1-9d33-48e4-bea7-6fabbf87ba3b.png)

 kernel config 里面删掉 CONFIG_COMPAT_VDSO

#### error: statement with no effect [-Werror=unused-value]

```plain
../drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_cfg.c: In function ‘hdd_cfg_print’:
../drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_cfg.c:6896:43: error: statement with no effect [-Werror=unused-value]


 error: ‘staid’ may be used uninitialized in this function [-Werror=maybe-uninitialized]
  911 |         hdd_dhcp_indication(pAdapter, staid, skb, QDF_RX);
      |         ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  CC      drivers/soc/qcom/early_random.o
  CC      drivers/video/fbdev/msm/mdss_mdp_ctl.o
  CC      drivers/video/fbdev/msm/mdss_mdp_pipe.o
```



　临时解决办法： 增加 -Wno-error=unused-value  -Wno-error=maybe-uninitialized，见一种加一种。最后觉得太麻烦了，直接Makefile里增加-w选项，屏蔽所有警告。



#### 开启内核支持

安装termux。这里为了控制方便我开启了ssh，用电脑连接上去操作。然后下载check脚本看缺少哪些内核选项。

```plain
pkg install tsu
pkg install wget
wget https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh
chmod +x check-config.sh
sed -i '1s_.*_#!/data/data/com.termux/files/usr/bin/bash_' check-config.sh
sudo ./check-config.sh
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668513581148-459e8152-1790-48f2-bb50-d19a28afd938.png)

Generally Necessary下面必须要全部是绿色才有可能安装Docker，这里看到还是有很多内核选项是缺失的。

常规的修改内核选项是需要通过menuconfig，但是缺的太多了，一个个手动开启很麻烦。我们可以用一个偷懒的方法：直接编辑defconfig文件，手动添加进去。可以按照酷安上的一个帖子来操作：

https://www.coolapk.com/feed/38099071?shareKey=YTdhYmE1NWRkZjE3NjM3MzgwZjQ~&shareUid=23564717&shareFrom=com.coolapk.market_12.5.2



```plain
vim arch/arm64/configs/sagit_defcofig
```



加入以下内容，并保存。

```plain
CONFIG_NAMESPACES=y
CONFIG_NET_NS=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_SCHED=y
CONFIG_CPUSETS=y
CONFIG_MEMCG=y
CONFIG_KEYS=y
CONFIG_VETH=y
CONFIG_BRIDGE=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
CONFIG_NETFILTER_XT_MATCH_CONNTRACK=y
CONFIG_NETFILTER_XT_MATCH_IPVS=y
CONFIG_NETFILTER_XT_MARK=y
CONFIG_IP_NF_NAT=y
CONFIG_NF_NAT=y
CONFIG_POSIX_MQUEUE=y
CONFIG_NF_NAT_IPV4=y
CONFIG_NF_NAT_NEEDED=y
CONFIG_CGROUP_BPF=y
CONFIG_USER_NS=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_CGROUP_PIDS=y
CONFIG_MEMCG_SWAP=y
CONFIG_MEMCG_SWAP_ENABLED=y
CONFIG_IOSCHED_CFQ=y
CONFIG_CFQ_GROUP_IOSCHED=y
CONFIG_BLK_CGROUP=y
CONFIG_BLK_DEV_THROTTLING=y
CONFIG_CGROUP_PERF=y
CONFIG_CGROUP_HUGETLB=y
CONFIG_NET_CLS_CGROUP=y
CONFIG_CGROUP_NET_PRIO=y
CONFIG_CFS_BANDWIDTH=y
CONFIG_FAIR_GROUP_SCHED=y
CONFIG_RT_GROUP_SCHED=y
CONFIG_IP_NF_TARGET_REDIRECT=y
CONFIG_IP_VS=y
CONFIG_IP_VS_NFCT=y
CONFIG_IP_VS_PROTO_TCP=y
CONFIG_IP_VS_PROTO_UDP=y
CONFIG_IP_VS_RR=y
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_APPARMOR=y
CONFIG_EXT4_FS=y
CONFIG_EXT4_FS_POSIX_ACL=y
CONFIG_EXT4_FS_SECURITY=y
CONFIG_VXLAN=y CONFIG_BRIDGE_VLAN_FILTERING=y
CONFIG_CRYPTO=y CONFIG_CRYPTO_AEAD=y
CONFIG_CRYPTO_GCM=y
CONFIG_CRYPTO_SEQIV=y
CONFIG_CRYPTO_GHASH=y CONFIG_XFRM=y
CONFIG_XFRM_USER=y
CONFIG_XFRM_ALGO=y
CONFIG_INET_ESP=y
CONFIG_INET_XFRM_MODE_TRANSPORT=y
CONFIG_IPVLAN=y
CONFIG_MACVLAN=y
CONFIG_DUMMY=y
CONFIG_NF_NAT_FTP=y
CONFIG_NF_CONNTRACK_FTP=y
CONFIG_NF_NAT_TFTP=y
CONFIG_NF_CONNTRACK_TFTP=y
CONFIG_AUFS_FS=y
CONFIG_BTRFS_FS=y
CONFIG_BTRFS_FS_POSIX_ACL=y
CONFIG_BLK_DEV_DM=y
CONFIG_DM_THIN_PROVISIONING=y
CONFIG_OVERLAY_FS=y
```

但是注意，这里是有坑的：按照这个帖子中的操作后刷入内核，再跑一遍check脚本会发现还是会有missing，原因是部分内核选项需要依赖另外的选项，而这些另外的依赖没有开启，所以还是missing的状态。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668521463385-3c693dc6-05ac-4ec4-b271-6a09b0ddd153.png)

所以最完整的做法，是加入之后再手动make  menuconfig，看哪些缺失开哪些。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668871779865-8a5c7c84-fa72-45b1-b85a-e7e0f461478e.png)

```plain
rm -rf out
make O=out sagit_defconfig
make O=out menuconfig
make O=out savedefconfig 
make O=out -j$(nproc)
```

经过等待之后，如果出现如下提示，就说明我们的内核已经编译好了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1670144450420-bf4b8c97-8a17-430b-a60d-d9f199c8b7b9.png)

### 刷入内核

刷入内核使用的是Anykernel3，详细可以看这篇教程：https://www.akr-developers.com/d/125

修改完配置文件后，把上一步arch/arm64/boot/Image.gz-dtb放到Anykernel3的根目录下，然后在根目录中执行zip -r <压缩包名.zip> *即可

最后通过twrp：adb sideload xxx.zip刷入内核。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/1599908/1670657586377-64c0fa41-6452-497e-9d03-e2b0ddc290ba.jpeg)

### Termux安装docker

Termux里面再跑一边check脚本，保证Generally Necessary是绿的。然后进入下一步：安装Docker。

通过Termux执行以下命令

```plain
apt update && apt upgrade -y
pkg install root-repo
pkg install golang make cmake ndk-multilib tsu tmux docker
```

然后编译tini

```plain
mkdir $TMPDIR/docker-build
$ cd $TMPDIR/docker-build
$ wget https://github.com/krallin/tini/archive/v0.19.0.tar.gz
$ tar xf v0.19.0.tar.gz
$ cd tini-0.19.0
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PREFIX ..
$ make -j8
$ make install
$ ln -s $PREFIX/bin/tini-static $PREFIX/bin/docker-init
```

编译的时候会有一个报错

```plain
/data/data/com.termux/files/usr/tmp/docker-build/tini-0.19.0/src/tini.c:434:19: error: a function declaration without a prototype is deprecated in all versions of C [-Werror,-Wstrict-prototypes]
void reaper_check () {
                  ^
                   void
```

手动修一下，在报错的地方加一个void，或者make的时候加入gcc参数忽略报错。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668520035648-53ede9bf-afa7-46dc-a9d9-514bc45b9c81.png)

启动Docker

```plain
sudo dockerd --iptables=false &>/dev/null &
```

测试Docker是否正常运行

```plain
sudo docker run hello-world
```



这里我的设备报错了，搜了很久也没找到原因跟解决办法，在老外帖子下面的评论区也有人回复说遇到了一模一样的情况。无奈只好放弃。

报错日志

```plain
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "mqueue" to rootfs at "/dev/mqueue": mount mqueue:/dev/mqueue (via /proc/self/fd/6), flags: 0xe: device or resource busy: unknown.
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1668518091094-93bd308a-bab5-4207-850a-9866ccb558c0.png)



## 方案二：刷入postmarkOS

安卓内核阉割的东西太多了，鬼知道改了什么东西。修改安卓内核的方式行不通，只能转换思路。那么能否在手机上去安装一个完整的Linux呢，当然是可以的，那就是postmarkOS。

postmarkOS是一个运行在手机上的Linux系统，基于Alpine Linux，比Ubuntu Touch更为接近原始的Linux，并且尽量使用Mainline kernel。官网：https://wiki.postmarketos.org/

### 准备工作

之前研究这个项目的时候，官方还没有合并小米6的分支：https://gitlab.com/postmarketOS/pmaports/-/merge_requests/3134， 只能手动本地编译一波。当时查看底层代码后发现postmarketOS官方的安装工具pmbootstrap对本地开发分支不太友好，例如构建rootfs下载驱动的时候，会硬编码从pmos的gitlab项目去拉取驱动，而我们使用的是本地修改的分支，当然是下载不到的。后来又魔改了一波pmbootstrap的代码，将硬编码的部分改为本地获取。不过网上对于pmOS移植这块的文档很少，也可能是我操作的姿势不对，最后勉强build好了一个zip，刷入系统但是开不了机。

不过好消息是两周前官方已经把小米6的分支合并进去了，也就意味着我们不用自己去手动移植了。更详细的教程可以参考这一篇：https://www.cnblogs.com/hongshao/p/14351045.html。

首先在Linux中执行以下命令：

```plain
pip3 install --user pmbootstrap
pip3 install --user --upgrade pmbootstrap
pmbootstrap init
```



![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669040980210-1a44a1b2-a019-4be1-9721-0ced9e43dd1d.png)

### 刷入postmarkOS

在官方的教程中	，默认是通过pmbootstrap flasher 命令直接刷入，但是我选择了制作卡刷包来刷入系统，这样卡刷包可以后期反复使用：https://wiki.postmarketos.org/wiki/Installation_from_recovery_mode

注意一定要加 --recovery-install-partition=data参数，不然刷入后你的分区会非常的小。

```plain
pmbootstrap install --android-recovery-zip --recovery-install-partition=data
pmbootstrap export
cd $(dirname $(readlink /tmp/postmarketOS-export/pmos-*.zip))
```

这里有一个坑，拿到zip文件后这个时候还不能直接adb sideload，会报错：

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1670657558497-6af303b6-3890-4696-b870-79a5c85022e4.png)

正确的做法是要先进入twrp，然后unmount所有的分区，再进行sideload刷入。

```plain
adb sideload pmos-xiaomi-sagit.zip
```

刷完后twrp会有一个提示：挂载data失败。别着急，因为data分区已经被我们修改了，其实系统已经刷入完成了。

重启就可以进入我们刷好的postmarketOS系统了。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/1599908/1670657639783-250752c7-35d9-4598-8e42-3d4d19cc9778.jpeg)

### 安装RNDIS驱动

首先按照官方的教程，开启手机的SSH：https://wiki.postmarketos.org/wiki/SSH。

这里出现了一个问题，开启SSH服务后连接电脑，提示无法识别设备，原因是电脑上没有对应的驱动，需要按照我下面的步骤来安装。



![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669432805883-9ca19ad3-5374-445a-902e-5a03bb60e63c.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669433030881-5bd7368f-4624-44be-b859-73ee62c33857.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669433100949-a43e96cd-59d2-4738-a443-4e67dc237923.png)

安装完毕后就可以正常识别手机设备了，成功连接上了SSH。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669433160908-74efb860-dc1d-4d56-ad3d-d2edd1792ef9.png)

### 通过USB共享网络

当前的系统版本WIFI有BUG不能用，修复的分支官方还没有merge：https://gitlab.com/msm8998-mainline/linux/-/merge_requests/2，所以我们要想一些另外的办法来联网。

按照官方教程 https://wiki.postmarketos.org/wiki/USB_Internet，在Ubuntu中以root权限输入以下命令：

```plain
sysctl net.ipv4.ip_forward=1
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 172.16.42.0/24 -j ACCEPT
iptables -A POSTROUTING -t nat -j MASQUERADE -s 172.16.42.0/24
iptables-save
ufw disable
```

然后ping一下百度试试，这个时候已经可以联网了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669535069380-5b1f17b1-fc9d-4f37-b336-a78c30176809.png)

### Hello from Docker!

终于要进入安装Docker的环节了，首先更新一下系统，然后apk add docker

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669535225500-02bbd504-871f-4cdc-bc8b-5c37e4eff97b.png)

Hello from Docker!  我们终于成功在手机上运行了Docker。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669536312398-8e816a8c-b47d-4965-bd08-f1b3dcb4f6ff.png)

看下htop

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669536576515-9b0f25d4-7521-4595-ba44-8728bbdcc764.png)

空间使用

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1669536597780-7e22867d-a2ac-4185-b59a-f2d8ecb4ee72.png)



## 玩法扩展

小米6的处理器是骁龙835，8核，10nm，2.45GHz，对比树莓派4B CPU BCM2711 4核 28nm 1.5 GHz。旧手机拿来当开发板，这不比树莓派香？触摸屏WIFI蓝牙红外都配齐了。

根据cpubenchmark的跑分对比，835的跑分几乎是树莓派CPU的三倍：https://www.cpubenchmark.net/compare/4297vs3919/BCM2711-vs-Qualcomm-Technologies,-Inc-MSM8998

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1670510071650-657cebd4-73a6-49db-8d27-bef8a1d91dd6.png)

不过美中不足的是小米的手机一直是USB2.0，IO简直是龟速。。。外接个硬盘也就十几M速度。。。跑重IO的业务是不太行了。

现在，我们的小米6摇身一变，变成了一个性能还不错的Linux开发板。我们可以安装php+nginx搭个博客，或者在家里跑Homeassistant等等。

作为一个安全从业者，还是会想是否能在安全场景下有所应用。想象一个场景：在近源渗透的时候，如果拿着电脑或者树莓派，一眼就看起来很奇怪。但是如果我们把这些东西装进手机，这样伪装性就会大大增加，之前也有师傅研究过这个课题：https://www.anquanke.com/post/id/204544。 Kali虽然有NetHunter，但是受限于安卓系统，NetHunter还是很难发挥出原生Kali的所有功能；而安装pmOS后的手机是一个完整的Linux，理论上能够实现所有原生Kali的功能。不过以上只是猜想，具体可行性还有待验证。



## 最后

终于给手机安装上了Docker，不过然后呢？好像也并不会真的用到它。不过刷机享受的就是折腾的过程，~~在手机上跑Docker，不觉得很酷吗？作为一名理工男我觉得这太酷了，很符合我对未来生活的想象，科技并带着趣味。~~
