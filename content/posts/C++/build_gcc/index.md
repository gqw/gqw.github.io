---
date: 2021-07-30
title: "Linux 编译GCC 11"

menu:
  sidebar:
    name: "Linux 编译GCC 11"
    identifier: linux-build-gcc-11
    parent: C++
    weight: 10
hero: gnu-gcc.jpg	
---

```bash
# 下载，解压
tar -xf gcc-11.1.0.tar.xz
cd gcc-11.1.0.tar
# 安装必要编译工具
sudo apt install gcc g++ make
# 安装依赖库
sudo apt install libgmp-dev
# 生成makefile文件
./configure
# 编译安装
make -j 10
sudo make install
```
