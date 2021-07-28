---
title: Html in markdown
weight: 30
menu:
  notes:
    name: Html in markdown
    identifier: markdown-html
    parent: notes-html
    weight: 30
---

<!-- markdown 图片加标注 -->
{{< note title="markdown 图片加标注">}}

<div>
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">HTTP下载方式</div>
    </center>
</div>

```html
<center id="http">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
    width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">HTTP下载方式</div>
</center>
```



{{< /note >}}


{{< note title="Markdown 排列图片">}}

<div>
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">HTTP下载方式</div>
    </center>
	<center >
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/nD9GQ5mOpVv1ESR.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">BT下载方式</div>
    </center>
</div>

```html
<center id="http">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
    width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">HTTP下载方式</div>
</center>

<center id="BT">
	<img style="border-radius: 0.3125em;
	box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
	width="200px" height="200px" src="https://i.loli.net/2021/07/26/nD9GQ5mOpVv1ESR.png">
	<br>
	<div style="color:orange; border-bottom: 1px solid #d9d9d9;
	display: inline-block;
	color: #999;
	padding: 2px;">BT下载方式</div>
</center>
```

{{< /note >}}


{{< note title="Markdown 图片横排">}}

<div style="display: flex;justify-content: space-around;">
    <center id="http">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">HTTP下载方式</div>
    </center>
	<center id="bt">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/nD9GQ5mOpVv1ESR.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">BT下载方式</div>
    </center>
</div>


```html
<div style="display: flex;justify-content: space-around;">
    <center id="http">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/QVMgJblTeGuY6fO.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">HTTP下载方式</div>
    </center>
	<center id="bt">
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);"
        width="200px" height="200px" src="https://i.loli.net/2021/07/26/nD9GQ5mOpVv1ESR.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">BT下载方式</div>
    </center>
</div>
```

{{< /note >}}