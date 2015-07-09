---
layout: post
category : XSLT
title: 用 XSLT 把 XML 转成 HTML 的一些经验
description : 记录一些使用 XSLT 把 XML 转成 HTML 时遇到的一些问题和解决办法
tags : [XSLT]
---

最近项目中需要把几个 `HTML` 文件合并成一个文件，每个 Tab 页显示一个 `HTML`, 同时并且提取出里面的信息来生成 `summary page`。 

解决办法是用脚本依次读取这几个 `HTML` 文件，提取信息处理后生成一个 `XML` 文件，然后再用 `Xalan-c 1.10`  通过 `XSLT 1.0` 文件转成最终的 `HTML`。`Tabs` 和 `summary page` 的实现使用了 `Bootstrap`。

第一次用 `XSLT` 和 `Bootstrap`, 总的来说感觉用 `XSLT` 生成 `HTML` 还是挺方便的。 `Bootstrap` 官方文档写得很好，用起来也简单，不用懂 `javascript` 和 `CSS` 也能让你的 `HTML` 发生脱胎换骨的变化。

下面记录的是我在使用中遇到的一些问题和解决办法。


嵌入 `javascript` 和 `CSS`
---
可以用 `<xsl:text disable-output-escaping="yes">...</xsl:text>` 把 `Bootstrap js/CSS` 文件和依赖的 `jQuery js` 嵌入到 `<head>` 中，用`<xsl:comment>` 也可以。

    <xsl:template match="/">
        <html lang="en">
            <head>
                ...
                <xsl:text disable-output-escaping="yes"> 
                    <![CDATA[
                    <style>
                        ...bootstrap.min.css...
                    </style>
                    <script type="text/javascript"> 
                        ...jquery.min.js...
                    </script>
                    <script type="text/javascript"> 
                        ...bootstrap.min.js...
                    </script>
                    ]]>
                </xsl:text>
                ...
            </head>
            <body>
                ...
            </body>
        </html>
    </xsl:template>

直接把这几行加到 `<head>` 中不就行了，为什么要把整个 `js/CSS` 文件都嵌进去呢？因为用链接的话需要联网才能用，还是整个嵌入保险些。

    <link rel="stylesheet" href="http://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.11.3/jquery.min.js"></script>
    <script src="http://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>

`<br/>` => `<br>`
---
一般我们转 `HTML`的话都是这样写 `<xsl:output>` 的： **`<xsl:output method="html"/>`**，但这样的话 `XSLT` 文件中的 `<br/>` 会被转成 `<br>`。

`<br>` 在 `HTML` 中是合法的，但去不是合法的 `XHTML/XML element`。因为我需要在脚本中把 `HTML` 文件中的某部分当成 `XML` 来解析，就必须用 `<br/>` 这种格式。

解决办法是修改 `<xsl:output>`，按 `XML` 格式来输出，这样 `<br/>` 会转成 `<br />`。

	<xsl:stylesheet version="1.0"
        xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

        <xsl:output method="xml" indent="yes"
            doctype-public="-//W3C//DTD XHTML 1.0 Strict//EN"
            doctype-system="DTD/xhtml1-strict.dtd"/>
        ...
    </xsl:stylesheet>

为什么在脚本中不直接当成 `HTML` 而是用 `XML` 来解析呢？虽然有些 `HTML` 的解析库，但都是第三方的需要先申请还要安装，太繁琐，而 `XML` 的解析库有现成的，用的也比较熟了。

`<textarea />`
---
`XSLT` 中的

    <textarea style="width:100%;height:100%;resize:none"></textarea>

在转成 `HTML` 后变成了

    <textarea style="width:100%;height:100%;resize:none" />
但是 `<textarea>` 在 `HTML` 中不属于 `selft-closing tag`，也就是说 `<textarea/>` 是非法的。怎么办？

直接在在 `XSLT` 文件中加个空格 `(<textarea>  </textarea>)` 解决不了问题，我们需要再次用到 `<xsl:text>`:

    <textarea style="width:100%;height:100%;resize:none">
        <xsl:text> </xsl:text>
    </textarea>

转换成 `HTML`:

    <textarea style="width:100%;height:100%;resize:none" id="2108"> </textarea>

`<xsl:apply-templates>`
---
假设有下面的 `XML`:

    <root>
        <a>
            <a_1>1</a_1>
        </a>
        <b>
            <b_1>2</b_1>
        </b>
    </root>

在 `XSLT` 文件中如果想先遍历 `<a>` 和 `<b>` 结点，然后回过头来再次遍历 `<a>` 结点，能做到吗？

先看下下面的尝试：

##### select + mode

    <?xml version="1.0" encoding="UTF-8"?>
    <xsl:stylesheet version="1.0"
        xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

        <xsl:output method="html"/>

        <xsl:template match="/">
            <html lang="en">
                <body>
                    <xsl:apply-templates select="a" mode="first"/>
                    <xsl:apply-templates select="b"/>
                    <xsl:apply-templates select="a" mode="second"/>
                </body>
            </html>
        </xsl:template>

        <xsl:template match="a" mode="first">
            <p>Match a first</p>
        </xsl:template>

        <xsl:template match="b">
            <p>Match b</p>
        </xsl:template>

        <xsl:template match="a" mode="second">
            <p>Match a second</p>
        </xsl:template>
    </xsl:stylesheet>

期望的是得到：

    <html lang="en">
    <body>
    	<p>Match a first</p>
    	<p>Match b</p>
    	<p>Match a second</p>
    </body>
    </html>

实际上得到的是：

    <html lang="en">
    <body></body>
    </html>

##### 只用 mode

    <?xml version="1.0" encoding="UTF-8"?>
    <xsl:stylesheet version="1.0"
        xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

        <xsl:output method="html"/>

        <xsl:template match="/">
            <html lang="en">
                <body>
                    <xsl:apply-templates mode="first"/>
                    <xsl:apply-templates select="b"/>
                    <xsl:apply-templates mode="second"/>
                </body>
            </html>
        </xsl:template>

        <xsl:template match="a" mode="first">
            <p>Match a first</p>
        </xsl:template>

        <xsl:template match="b">
            <p>Match b</p>
        </xsl:template>

        <xsl:template match="a" mode="second">
            <p>Match a second</p>
        </xsl:template>
    </xsl:stylesheet>

结果：

    <html lang="en">
    <body>
        <p>Match a first</p>

            2


        <p>Match a second</p>

            2


    </body>
    </html>

上面的方法都不行。这是因为 `XSLT` 转换时是按顺序一次性遍历 `XML` 结点的，不会遍历多次。

还尝试了其他方法，比如用 `<xsl:copy>` 把 `<a>` 结点的内容先保存起来后面再遍历。但这种方法保存起来的文本再次遍历时是没法用 `<xsl:for-each>` / `<xsl:apply-templates>` 去遍历子结点的。`XSLT 2.0` 倒是支持这样做，但我们用的 `Xalan-c 1.10` 只支持 `XSLT 1.0`，官方说要到 `Xalan-c 2.0` 才会支持 `XSLT 2.0`。

##### 怎么解决呢？
办法很简单，既然只能按顺序遍历一次，那就把 `<a>` 拷贝一份放在 `<b>` 结点下，反正这个 `XML` 只是个中间文件，生成 `HTML` 后就可以删除的，里面的格式自己随便定义。

`XML` 文件：

    <root>
        <a> 
            <a_1>1</a_1>
        </a>
        <b>
            <a_copy>
                <a_1>1</a_1>
            </a_copy>
            <b_1>2</b_1>
        </b>
    </root>

`XSLT` 文件：

    <?xml version="1.0" encoding="UTF-8"?>
    <xsl:stylesheet version="1.0"
        xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

        <xsl:output method="html"/>

        <xsl:template match="/">
            <html lang="en">
                <body>
                    <xsl:apply-templates/>
                </body>
            </html>
        </xsl:template>

        <xsl:template match="a">
            <p>Match a first</p>
        </xsl:template>

        <xsl:template match="b">
            <p>Match b</p>
            <xsl:apply-templates/>
        </xsl:template>

        <xsl:template match="a_copy">
            <p>Match a second</p>
        </xsl:template>

        <xsl:template match="b_1">
        </xsl:template>

    </xsl:stylesheet>


生成的 `HTML`：

    <html lang="en">
    <body>
        <p>Match a first</p>
        <p>Match b</p>
            <p>Match a second</p>


    </body> 
    </html>
