---
title: wsl
weight: 30
menu:
  notes:
    name: wsl
    identifier: notes-linux-wsl
    parent: notes-linux
    weight: 30
---

<!-- markdown 图片加标注 -->
{{< note title="wsl2 默认root登录">}}
```bat
[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss\{a7b6183f-cc3e-4388-a97a-502a98efca90}]
"DefaultUid"=dword:00000000
```
设置 DefaultUid 为0
{{< /note >}}


<!-- markdown 图片加标注 -->
{{< note title="wsl2 限制CPU和内存大小">}}

创建 %UserProfile%\\.wslconfig 添加如下内容
```bat
[wsl2]
memory=4GB # Limits VM memory in WSL 2 to 4 GB
processors=4 # Makes the WSL 2 VM use two virtual processors
```
{{< /note >}}