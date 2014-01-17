---
layout: post
title: "ubuntu 软件推荐与安装"
date: 2014-01-03 20:14
comments: true
categories: ubuntu linux
---

可能有人觉得linux下的软件不够多,或者没有windows那么好用,但我要告诉你,linux的软件多得吓人,游戏也多得吓人,不信你往下看就知道了,在这里我们不比windows和linux这两个操作系统,也不找linux下的软件来代替windows,我介绍几个我认为很漂亮又很有用的软件

我的环境是ubuntu 13.04

### 输入法fcitx

其他输入法我没怎么用过,我只用它,它对我来说足够了,而且安装设置都很简单

<!-- more -->

``` bash
sudo apt-get install fcitx-table-all fcitx
```

在ubuntu dash那里输入input,进入input method configuration(输入法配置)

{% img /images/ubuntu_softwares/unity-dash-sample.png %}

{% img /images/ubuntu_softwares/fcitx_config.png %}

重启之后按ctrl+shift就能用了

### 听歌神器DMusic深度音乐播放器

这款深度音乐播放器很好用,界面也美,能装百度,豆瓣插件,功能强大,又有歌词显示,超级美的一个软件

``` bash
sudo add-apt-repository ppa:noobslab/deepin-sc
sudo apt-get update
sudo apt-get install deepin-music-player
```

{% img /images/ubuntu_softwares/dmusic.png %}

插件设置如下

{% img /images/ubuntu_softwares/dmusic_plugins.png %}

默认是没有百度音乐插件的,要自己安装

``` bash
sudo apt-get install cython libwebkitgtk-dev git
git clone https://github.com/sumary/pyjavascriptcore.git
cd pyjavascriptcore
sudo python setup.py install
git clone https://github.com/sumary/dmusic-plugin-baidumusic.git
cd dmusic-plugin-baidumusic
cp -r baidumusic ~/.local/share/deepin-music-player/plugins/
```

启动它在ubuntu dash里输入DM就能找到了,或者在终端里运行deepin-music-player也可以

### 视频软件DPlayer深度影音播放器

用这款视频软件因为我是喜欢上面那款深度音乐播放器,相信这款视频软件也很优秀

``` bash
sudo add-apt-repository ppa:noobslab/deepin-sc
sudo apt-get update
sudo apt-get install deepin-media-player
```

{% img /images/ubuntu_softwares/dplayer.png %}

ubuntu dash里输入DP或终端里运行deepin-media-player就能用了

### 聊天工具qq2012

这款是用wine运行的,我使用这么久基本上不会出现什么问题

[狠点这里下载](http://test.packages.linuxdeepin.com/deepin/pool/non-free/d/deepinwine-qqintl/)

注意是下载那个叫wine-qqintl_0.1.3-2_i386.deb的文件,下载后双击安装就可以了

{% img /images/ubuntu_softwares/qq.png %}

要使用的话在ubuntu dash里输入wine就能找到它了

### 办公软件wps for linux

必须是它了,没错的!表格处理,word档处理,ppt全靠它了

[狠点这里下载吧](http://community.wps.cn/download/)

直接安装就行了

{% img /images/ubuntu_softwares/wps_ppt.png %}

### 截图软件shutter

这个软件可以选择全屏或者区域截图,功能强大还支持编辑功能

``` bash
sudo apt-get install shutter
```

{% img /images/ubuntu_softwares/shutter.png %}

可编辑功能如下

{% img /images/ubuntu_softwares/shutter_edit.png %}


### todo软件nitro

程序员可能会喜欢这种todo软件,这款软件真的设计得很漂亮

``` bash
sudo add-apt-repository ppa:cooperjona/nitrotasks
sudo apt-get update
sudo apt-get install nitrotasks
```

{% img /images/ubuntu_softwares/nitro.png %}

### 定制工具ubuntu-tweak

这款神器可以用来修改ubuntu的一些配置,可以清一些无用的垃圾，基至可以装软件

而且它是国人开发的,值得骄傲

``` bash
sudo apt-get install ubuntu-tweak
```

{% img /images/ubuntu_softwares/ubuntu-tweak.png %}

最喜欢的还是那个janitor的功能,一下子就把浏览器缓存清得干干净净,还把一些下载的无用deb包也顺便清了,大快人心啊

{% img /images/ubuntu_softwares/ubuntu-tweak-clean.png %}

### 轻量级的软件中心App Grid

里面有大把软件游戏,自己慢慢找吧

``` bash
sudo add-apt-repository ppa:appgrid/stable
sudo apt-get update
sudo apt-get install appgrid
```

{% img /images/ubuntu_softwares/appgrid.png %}

### 好用的包安装器GDebi

用它来安装下载的deb包最合适不过了,它会先给你检验这个包能不能包装的

``` bash
sudo apt-get install  gdebi
```

{% img /images/ubuntu_softwares/gdebi.png %}

### 下载管理器uget

一个图形化的wget,可暂停之后再下载,支持队列,批量下载等,界面也很酷

``` bash
sudo apt-get install uget
```

{% img /images/ubuntu_softwares/uget.png %}

