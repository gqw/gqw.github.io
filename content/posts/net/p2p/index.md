---
date: 2021-07-27
title: "一种简单可行的P2SP实践方案"
linkTitle: "p2sp"
author: asura9527 ([@gqw](https://gqw.github.io))
menu:
  sidebar:
    name: 一种简单可行的P2SP实践方案
    identifier: simple-p2sp-realizition
    parent: net
    weight: 10
hero: imgs/hero.jpg
---

## 缘由与目标

随着计算机硬件的发展，游戏的玩法和画质也得到了飞速的发展。游戏内容越来越丰富，画面质量也越来越精细。这也导致了游戏的包体越来越大。PC端游戏基本上10GB起步，《战地》，《易水寒》等游戏基本已经达到100G体量。如此大的体量对于游戏发行方，是一个必须要考虑的问题。用于游戏分发的下载器我们必须考虑以下几个方面：

    1. 可访问， 这是下载器的首要前提，我们必须保证数据能够被联网的任意玩家可以访问到。
    2. 数据可靠，下载的资源必须是正确的内容。
    3. 速度，我们必须尽可能的保证玩家的下载速度。
    4. 稳定，需要能够稳定下载所需资源。
    5. 节省流量，虽然这是个可选项但是流量就意味着金钱，所以下载器还需要尽可能的节约流量。

通常的下载器使用的是Http方式下载，虽然能够保证可靠地数据访问，但是离高速稳定省流量还有较大差距，为了提高速度和稳定性我们通过使用CDN的方案有效解决。但是CDN解决不了节省流量的目标（实际上是增加流量成本的）， 简单的解决方法是使用压缩下载内容，目前项目中使用的下载器就是使用的这种方法。压缩数据的方法效果还是非常有效的，以某款为例，12GB的包可以压缩到7GB, 也就是可以节省40%的流量。

是否可以有更有效的方法解决流量问题呢？很自然的想法是使用P2P加速技术。P2P是指 peer to peer, 即节点到节点数据传输，就是各个客户端之间数据传输。大家常见的P2P技术主要是BT下载技术，但是BT与我们前面使用的技术是由冲突的。我们先看下HTTP与BT的结构图进行简单的对比下：

<div align=center style="display: flex;justify-content: space-around;">
<center id="http">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
    width="300px" height="300px" src="imgs/cs_download.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">HTTP下载方式</div>
</center>

<center id="bt">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
    width="300px" height="300px" src="imgs/bt_download.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">BT下载方式</div>
</center>
</div>

对比可知， HTTP是基于C/S架构的，客户机是严格依赖中心服务器的，BT无中心节点所有数据都是玩家之间传输。HTTP的好处是访问可控，只要客户机能连接到服务器就肯定能获得所需数据，但是服务器必须承担所有流量压力和流量费用。 BT的好处也是显而易见的，所有数据传输都是玩家之间的行为，甚至可以不需要部署服务器，服务器的压力和费用为零，但是BT却有可能违背我们前面所述下载的最基本要求——可访问，玩家节点的连接并不能像HTTP那样是可靠稳定的。

有没有可能结合HTTP和BT技术，充分利用各自的优势并互相弥补对方的缺点呢？实现如下的结构：
<div align=center>
<center id="bt">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
    width="300px" height="300px" src="imgs/p2p_download.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">P2SP下载方式</div>
</center>
</div>

幸运的是答案为YES, 迅雷的P2SP技术就是基于上述思想实现的，说明技术方向是对的，不用再去怀疑和验证技术的可行性。但是需要注意的是，迅雷的P2S我们的需求是有差异的， 迅雷下载器P2S部分的S并不是自己的服务器（而是各个下载站点的服务器），所以它不用考虑这部分的流量压力和费用，而我们作为内容提供方流量和费用必须自己承担。所以迅雷并不关心用户数据是否压缩，但是对于我们内容提供商就必须要考虑通过前面压缩技术获得的40%的利益。

看到这里你可能会想，压缩还不容易吗？直接用压缩软件把下载内容打包下上传到HTTP服务器不就行了吗？是的如果不考虑BT文件的分享率，这样做是解决了问题，但这意味着我们BT分享的内容也是个压缩包，因为我们业务是游戏启动器，也就是说我们不可能将用户下载的游戏安装包一直保留，举个例子，如果我们的游戏原始文件为100GB压缩后为60GB，如果按刚才的方案，用户下载60GB的安装包后，再解压为100GB的原始文件，也就是说必须占用用户160GB的磁盘空间，而这其中的60GB只是为了方便他人下载用的。

如果在用户安装完游戏后再将安装包删除，那此用户通过BT分享的时间也就只有其下载+安装的时间了。如何解决这个矛盾呢？ 我们现在梳理下问题：

1. 通过HTTP下载的内容需要时压缩的数据
2. 通过BT分享的内容必须是原始的内容

通过分析我们可以发现看似不可调和矛盾其实是有解的。那就是需要在下载的时候将HTTP压缩数据原地解压。自此方案的的思路已经基本清晰，但还有一个需要考虑的问题，根据BT的协议标准，BT下载时是将所有数据拼成一个数据块，再将这个数据块进行切块下载的。这就导致一个问题，通常我们压缩数据是按照文件为单位进行压缩的，但是BT下载时是按照切块进行下载的，这会导致HTTP和BT下载内容对应不上，如下图：

<div align=center>
    <center id="bt">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="700px" height="250px" src="imgs/zip_fmt.jfif">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">ZIP格式</div>
    </center>
</div>

<div align=center>
    <center id="bt">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="700px" height="250px" src="imgs/bt_fmt.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">BT格式</div>
    </center>
</div>

BT的下载原理是根据上图将需要下载的内容先按照文件进行拼接再进行切片（Piece），建立多个下载通道，每个通道根据切片的状态选择下载并更新切片状态。在下载时BT将无视文件的概念，对于BT来说每次下载只是一片连续的数据块。这样如果HTTP按照文件从压缩包中提取数据，下载后将很难找到对应的BT文件中Piece的位移，也就无法更新下载Piece的状态。

解决思路是，反过来我们在数据压缩时不再以文件为单位进行压缩，而是按照BT的Piece为单位创建压缩文件，并记录每个Piece在压缩文件中的位移。HTTP下载通道下载时先向BT模块申请需要下载的Piece，再根据Piece的位移从HTTP服务器下载相应内容，最后更新BT Piece状态。


## 具体实施

### 准备阶段

1. 创建种子文件（.torent）
2. 根据种子文件中`piece length`创建压缩文件
3. 修改种子文件添加peice的压缩信息，主要是每个piece压缩后大小和总压缩文件大小。可以根据piece压缩后大写算出每个piece的偏移量
4. 上传压缩文件到CDN
5. 修改种子文件添加CDN中压缩文件HTTP下载地址
6. 上传种子文件到CDN

实际处理中1,2,3,5步骤可以通过修改torrent文件合并处理。

### 下载阶段

修改BT下载器，根据torrent文件中的的压缩文件地址建立特殊下载通道，模拟BT请求下载块，根据BT文件中的压缩块大小计算Piece在压缩文件中的偏移，使用HTTP Range Requests标准下载对应的数据块，下载完后更新对应Piece状态。直至所有文件下载完成。

## 相关工程

[libtorrent](https://github.com/gqw/libtorrent/tree/gqw)

