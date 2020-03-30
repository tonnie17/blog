+++
title = "我眼中最好用的编程笔记本：Notion"
date = 2019-12-07T00:00:00+08:00
tags = ["notion"]
categories = [""]
draft = false
+++

最近在寻找一个工具来将以前的笔记和书签统一整理，刚好找到一款叫「Notion」的软件，使用了两天，感觉比较满足我的需求，于是打算分享我为什么选择Notion来作为我的笔记应用。

以前我用的比较多的笔记应用是Evernote，对于Evernote来说，它的优点有很多，首先它的界面简洁美观，而且功能非常稳定，全平台同步，使用起来大部分时间下很流畅，基本感受不到有卡顿的地方，另外要说的是Evernote的剪藏功能做的十分出色，在浏览网页时很方便地就能把网页格式化整理到笔记中。虽然Evernote作为一个笔记应用来说已经十分出众，但还是没有能解决我的痛点：

- 不支持多级目录，在Evernote中没有文件夹的概念，而是`笔记本组-笔记本-笔记`的关系，而我个人记录的习惯是把笔记归类地整整有条，并支持多层嵌套，而在Evernote中则需要用标签来归类笔记，很难做到我想要的效果
- 排版格式较为单一，在一些进行富文本编辑的场景下不太好控制样式
- 免费版不支持Markdown，而且代码显示格式丑陋，好消息是目前的Evernote版本已经添加了对Markdown的支持，而且用起来效果还不错

后面我又尝试过另外一些笔记应用，但都没有符合自己的需求，后来我使用了几天Notion，发现它刚好能解决我的需要，说说个人认为它最大的优点：

- 可以嵌套任意层级的Page，使用嵌套Page可以在Notion中实现知识的整理和分类
- 强大和类型丰富的Block，内容即组件，可以灵活地实现想要的显示效果
- 全平台同步，本身是一个Web App，因此可以做到全平台显示效果保持统一
- 自带了许多Template，可以快速成型一个页面
- 内置了页面历史和团队协作功能
- 导入功能，可以从Evernote和Google Doc等地方导入笔记

当然，主要对我最有帮助的还是前面两个功能，作为一个程序员，经常会遇到的事就是遇到一些问题要各种翻阅资料，比如经常需要去查命令手册，基础语法和一些特定问题的解决办法，这时候像我这种懒的程序员就容易手忙脚乱，东找西搬，那么如果有一个工具来把这些常用知识进行整理分类的话，遇到问题后查询解决的效率也就会变得事半功倍了。

在Notion中，「Page」成为一个取代文件夹的存在，可以建立多个Page，然后对这些Page按树状进行排列，就可以实现跟文件夹一样的效果，而且Page之间可以相互嵌套，也可以共同组成另一个Page。

如下图所示，在菜单栏的左下方就是一些创建好的Page，每个Page可以添加自己的图片，这里默认可以选择里面提供的emoji，还能选择Page的cover（也就是右上角的背景图），在右下角中展示的是Page的内容，可以看到在这个Page的下面嵌套了7个Page，因此这7个Page会变为Page Link出现在父Page的内容中，可以说在Notion中Page同时扮演了笔记本和文件夹两种角色。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4fcdbde93033b7140deaf3f56ec06931_1440w.jpg)
Page

这也是我认为为什么Notion的分类功能更加直观的原因，每个Page可以按照其内容给它设置一个合适的icon，当要去查询笔记时我就能只看到图标就快速能找到想要的Page，提高查找效率。

在新建一个Page时，还可以使用Notion自带的Tempalte功能从它提供好的模板列表中挑选合适的模板来创建新页面，这里面的模板基本覆盖了日常使用的大多数场景，比如笔记、任务、待办事项、项目管理和会议纪要等等，对于新手来说入门是十分友好的，如果对入门Notion感到迷茫，就从这里开始吧。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-b1d33be7863d81b9e25a6157091e6222_1440w.jpg)
Template

下面这个阅读列表，就是从自带的模板创建出来的，现在我更习惯用Pocket来标记网页书签和文章，稍后再筛选将它们同步到这个列表中，在Notion中这种表格属于「Database」视图中的一种，在Notion的Database中可以对视图内容进行搜索、增删、过滤筛选等操作，而且Notion提供了Table、Board、Calendar、List和Gallery五种视图，可以很方便在页面中完成各种视图之间的切换，以下就是同一份数据在Table和Board两种视图下的展现形式：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4a67fb7ac5ff7438c1ee7e8a25a7923b_1440w.jpg)
Table（表格）

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-b3ecb52b2ca4a7cec5ad002e385e3a11_1440w.jpg)
Board（看板）

在Notion中，所有内容都是一个个的「Block」，包括文字、图片、代码块甚至是上面提到的Database，都属于一种类型的Block，Notion中所有内容都是由Block构成的，自身提供了多种类型的Block，在Page中输入斜杠 `/` 会弹出Block类型列表提供选择和搜索。每个Block都是一个模块化的组件，可以自由被拖拽，插入到页面的各种位置，如下图的两段文字、两个代码块以及一张图片就使用了Block的拖拽实现了类似分栏的效果，充分利用了页面的显示空间：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-dce6c04b27014231aee4c599c969fe10_1440w.jpg)
Block

这种分栏效果也尤其适合，当你有一些Page，但你需要把他们组织整理起来的时候，比如做一个导航用的索引目录，这时候Block的拖拽功能往往很有帮助，你可以选中Block的内容，对他们进行**加粗/染色/高亮/评论/提及他人**等操作。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1c7daf13fe0915a5df1cbee727171c49_1440w.jpg)
Block

因为有了这种形式，我可以在Typora上完成文字的编辑，再把Markdown复制粘贴到Notion中，再对内容进行排版加工，调整至自己想要的样式。在Notion中，不仅可以支持基础类型的Block，而且提供了很多高级的Block类型，可以在页面中内嵌文件，音频，视频，以及Google Map，Google Drive，Github Gist等内容，如下面这个页面就内嵌了一个本地上传的PDF，把光标移到Block里面可以进行上下滚动浏览。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5ff6ed71d36533c120aa6954cc59fbd1_1440w.jpg)
Embed PDF

值得一提的是，在Notion中列表的功能非常好用，提供了Bulleted List（无序列表），Numbered List（数字列表），Toggle List（折叠列表），To-do List（待办列表）四种常规的列表样式，在写笔记时，可以通过拖拽Block的方式把段落移动到列表的层级之下，使用折叠列表还可以将大片内容收叠起来，使页面空间更加清晰。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-44b0f742688901a5e2e19744c994a6a1_1440w.jpg)
列表

一个普通的页面链接，在Notion中也有三种显示形式：文本链接，Web书签和内嵌页面（当然不是所有页面都可以进行内嵌的），用的比较多的还是前面两种类型，Web书签尤其适合用于来做一些网页摘要和链接引用。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-6d45d648dcce52194b445e8f603b936e_1440w.jpg)
链接

在Notion中有个比较实用的功能叫「Template Button」，你可以配置一个按钮，再把一个已有的Block拖拽到Button配置的Template中，这时候，当你点击这个按钮时，就会在按钮的附近出现一个你刚刚配置过的Block，假如你有很多页面都有一个通用的模板格式，你就可以把这个页面配置到Template Button，只需要轻点按钮，就能把模板创建出来。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-46445b92c6f00f4bbba474a11244df7c_1440w.jpg)
配置「Template Button」

说了那么多不能忘了笔记应用最重要的功能之一：「全文检索」，Notion的搜索功能在我感受里只是凑合能用的水平，搜索的速度，准确性和对中文的支持都给人体验不好，经常会出现搜索一个明明存在的词组却无法用搜索过滤出来，因此对中文检索有强需求的是不太推荐使用Notion的，相比来说我更倾向把它用作一个记录整理的工具。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-49efa0dfa76fd950f7512547d04ef93b_1440w.jpg)

除此之外，个人认为Notion还有以下缺点：

- 免费版只支持1000个Block（当然可能是我的缺点）
- 同步速度慢，客户端不好用，而且离线状态下没有使用体验
- 导出功能比较薄弱
- 不支持中文

但总的来说，我认为Notion还是值得一试的笔记应用，因为它的概念以及许多功能对比起其他笔记应用来说都是比较新意和有意思的，而且作为一个内容整理应用，用它很适合来记录和管理知识，构建Knowledge Base。

以上就是我使用了几天Notion的体验，只介绍了其冰山一角的功能，如果你也有兴趣使用Notion，欢迎使用我的[邀请链接](https://link.zhihu.com/?target=https%3A//www.notion.so/%3Fr%3Db14df3808dc349eabe9325300a77a87c)，注册后可以获取$10的优惠，免费体验2个月。
