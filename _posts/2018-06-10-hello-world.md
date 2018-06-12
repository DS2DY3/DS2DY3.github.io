---
title: 小白搭建gitpage
layout: post
category: 札记

---

之前已经有一个[Idiotory - whyidoit](https://srxqds.github.io/)，建的时候完全是一头雾水，主题也不是简约风的，色彩比较多，这点调整起来就困难，然后很多样式支持的也不好，进而影响自己写博客的积极性（好吧，借口！），基于这些因素，就一直想从头建一个gitpage，拖了很久（2年？），昨天加完班晚上11点到家，开始折腾到晚上9点`commit`了137次才弄完，，早上只躺了1个小时，成文记录下。

## 主题挑选

一直想找个极其精简的博客模板，类似王垠的[当然我在扯淡](http://www.yinwang.org/#)就很喜欢简单又不是设计感，但对设计和web编程也只是“七窍通了六窍”，只有找别人开源的。能google到的我全都筛选了一遍，像[awesome-jekyll-themes](https://github.com/planetjekyll/awesome-jekyll-themes)，[Github Pages themes](https://pages.github.com/themes/)等，如果你需求可以去找下。最终比较喜欢[Tale](https://chesterhow.github.io/tale/)和[whiteglass](https://yous.be/whiteglass/)两个风格的，最后采取以**whiteglass**为基础模板然后主页用**Tale**的`post`样式。当然自己也改造了很多。


## 动手改造

模板敲定了，接下来就是添砖加瓦，先整合了**Tale**的`pos`界面，然后加入`tag`和`category`的支持，最后还加上了访问人数的统计。

由于不想安装**Jekll**环境（总是被这种无脑的想法所困），所以都是修改了，提交github再看是否成功，浪费了很多时间，工欲善其事必先利其器啊！

## 后续计划

有了页面，还差一个好的编辑器，两者缺一不可，所以后面想法是基于[stackedit](https://github.com/benweet/stackedit)改造为一个离线的客户端编辑器。主要是感觉**stackedit**有两个不能满足我的需求：
- 不支持远程和本地工程目录
- 插入图片不够方便
- 多了一些不需要的功能
- ...

在没有更好的替代品出现时，只有自己动手才能"follow my heart"。

虽然gitpage的搭建本来就很傻瓜式，但是我还是折腾了很久，对于web一点经验都没有，从头开始（呼应下标题）。通过这个过程，用到了之前走马观花式学的东西，也学了很多，也参考了很多网上资源，下面罗列备注下，以表感谢：

1. [Jekyll 的配置和基础][1]
2. [Liquid 模板语法][2]
3. [CSS和HTML基础][3]
4. [Jekyll Tags on Github Pages][4]
5. [访问统计-不蒜子][5]
6. [Tale模板][6]
7. [whiteglass模板][7]


[1]: https://jekyllrb.com/docs/quickstart/ "Jekll guide"
[2]: https://shopify.github.io/liquid/ "Liquid Template Lanague"
[3]: http://www.apress.com/9781430237808 "Pro HTML5 and CSS3 Design Patterns"
[4]: http://longqian.me/2017/02/09/github-jekyll-tag/ "Jekyll Tags on Github Pages"
[5]: http://busuanzi.ibruce.info/ "不蒜子"
[6]: https://chesterhow.github.io/tale/
[7]: https://yous.be/whiteglass/
[8]: http://www.minddust.com/post/tags-and-categories-on-github-pages/







