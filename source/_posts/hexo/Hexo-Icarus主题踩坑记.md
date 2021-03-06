---
title: Hexo Icarus主题踩坑记
date: 2021-01-30
tag:
    - icarus
    - 主题
    - 加载慢
categories:
    - Hexo
        - 主题
thumbnail: /imgs/illustration/Hexo-Icarus主题踩坑记.jpg
---

# 性能优化

## 部分文件加载慢或加载失败

如下图，icarus默认使用的图标库是`fontawesome`，由于众所周知的原因，`fontawesome`的cdn是访问不到的。当查看页面时，直到这个all.css资源请求超时或返回404之前，页面会显示空白。

<!-- more -->

![fontt-awesome图标库加载失败](/imgs/illustration/2020-01-30-icarus-fq.jpg)

解决这个问题的办法也很简单，那就是换一个静态资源cdn。icarus中cdn的配置在`_config.icarus.yml`中，以下为默认的配置：

```yml
providers:
    # Name or URL template of the JavaScript and/or stylesheet CDN provider
    cdn: jsdelivr
    # Name or URL template of the webfont CDN provider
    fontcdn: google
    # Name or URL of the fontawesome icon font CDN provider
    iconcdn: fontawesome
```

其中`fontawesome`、`google`很可能都无法访问，可以改为使用`loli`的cdn服务：

```yml
providers:
    cdn: loli
    fontcdn: loli
    iconcdn: loli
```

推而广之，其他模板也是类似。当页面加载过慢时，可以先打开**开发者工具**，观察*network*面板，是否有无法加载的css、js资源。如有，就研究下如何使用替代的可访问资源。

