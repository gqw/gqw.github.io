---
title: Linux环境配置
weight: 30
menu:
  notes:
    name: Linux环境配置
    identifier: notes-linux-env
    parent: notes-linux
    weight: 30	
---

<!-- markdown 图片加标注 -->
{{< note title="gcc 源文件编译">}}
```bash
# 下载，解压
tar -xf gcc-11.1.0.tar.xz
cd gcc-11.1.0.tar
# 安装必要编译工具
sudo apt install build-essential
# 安装依赖库
sudo apt install libgmp-dev
# 生成makefile文件
./configure
# 编译安装
make -j 10
sudo make install
```
{{< /note >}}


{{< note title="从PPA安装">}}

```bash
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt install gcc-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 110 --slave /usr/bin/g++ g++ /usr/bin/g++-11 --slave /usr/bin/gcov gcov /usr/bin/gcov-11
```
{{< /note >}}