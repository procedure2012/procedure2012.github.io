---
title: 自制词典
tags: 技术
categories:
  - - 科普
  - - 随笔
date: 2021-04-07 19:01:06
---


>子贡问为仁。子曰：“工欲善其事，必先利其器。居是邦也，事其大夫之贤者，友其士之仁者。”
《论语 卫灵公》

这几天在做雅思的时候碰到了一个问题：因为没有纸质版我只能用pdf做题，但是pdf还是影印版全是图片，没有办法直接复制粘贴单词或者做标注。只能手动查词再手动添加到生词本里。整个流程繁琐且大量重复，效率低到令人发指。所以我一咬牙一跺脚——“您想了个更好的法子？”——我不做雅思了！

开玩笑开玩笑😂大量重复性工作当然要写个程序一劳永逸的解决问题，先来看看我的需求：

+ 陌生单词能在查完词典后不仅能显示详细的查询结果，还能自动保存到生词本里，不需要我任何的额外操作。
+ 生词本里只有我需要的单词，要包括发音、释义、例句，且排列美观，还能规划科学的单词复习计划。
+ 不要用手机，屏幕太小看着难受，键盘太小用着难受。要通过电脑操作，最好完全通过键盘操作。
+ 这个东西能够同步到多个不同平台的终端上。
+ 这个东西最好能扩展到德语上面实现同样的效果。
+ 德语的话，要能在查词结果中给出单词词性和各种变形格式。
+ 最好是不要钱

总之，在我看pdf或者纸质书的时候，任何一个我不认识的单词都可以输入到这个东西里，它会给我显示我想要的查询结果，并且帮我把这个词以我希望的形式保存进生词本里，我只需要在空闲时间按照这个生词本规划好的学习计划复习单词即可。

首先看看有没有现成的解决方案。各种背单词软件都有查词+背词的组合功能，但它们都集中在手机上。不要说让我搓玻璃板了，笔记本自带键盘我都不想用。更不要提看了就头疼的小屏幕……（我还特别想说一下平板，这个东西我感觉很鸡肋，什么都能干什么都干不好：娱乐买一个switch+上课纸笔做笔记，成本一点不比平板高，玩又玩的舒服，上课又不会被分散精力；在家看剧就用电脑屏幕大看着舒服，上床就老实睡觉不要熬夜，平板完全没有存在的价值。）扯远了，现成的app主要问题还是我用着不顺手，而那些漂亮的界面对我没有任何价值。而且现在我的人工流程已经比较完善了，结果也很好，把流程自动化就能够比现有app更好的满足我的需求。

人工流程是这样：发现生词-->到在线词典搜索-->把查询结果登记到生词本里。整体流程非常直观，第一步不需要自动化，第二步用一个简单的爬虫就可以实现，第三步只要找一个支持批量导入的生词本就可以了。最后我选了一个支持html+CSS自定义显示格式，并且能添加语音的生词本（[Anki](https://apps.ankiweb.net/)）。用python爬虫查询单词并保存结果，一天结束后把结果直接批量导入anki就可以了。我只需要输入一个个的单词，并在最后结束是点两下鼠标导入当天的成果就okay了。
<iframe width=100% height="480" src="//player.bilibili.com/player.html?aid=289948635&bvid=BV1pf4y1W7qZ&cid=319913936&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

在弄完英文词典后我又顺手写了个德语词典，整体思路完全一致，但是略微做了一些改变。中德在线词典做得实在是太拉跨，但是英德词典又没有完整的动词变位，所以我用了英德词典的结果+中德词典的动词变位拼在一起，我只需要输入一次单词就能得到所有我想要的结果，以前还要两个网页翻来翻去。虽然现在有些单词的读音还是有问题，不过已经基本满足了我的需求。
<iframe width=100% height="480" src="//player.bilibili.com/player.html?aid=844960415&bvid=BV1G54y1b7yd&cid=319914143&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

整个项目放在了我的[github](https://github.com/procedure2012/MyDict)上面，如果有人感兴趣可以下载下来试一试，我自己还觉得挺好用的（就是bug多了点😂）。如果大家有什么改进的想法也可以告诉我，希望以后能更新更多的语言版本上去。