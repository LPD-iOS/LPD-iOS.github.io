---
layout: post
title: 'SSH-Keygen 中生成的 Randomart Image 是什么'
date: '2017-03-27'
header-img: "img/post-bg-weexios.jpg"
tags:
     - SSH-Keygen
author: '小鱼周凌宇'
---

# randomart image 出现在哪里

通常我们在生成 SSH Key 的时候会用到 `ssh-keygen` 命令，在生成结束后，会输出类似如下的内容，这个 randomart image 是什么呢？

```
The key's randomart image is:
+--[ RSA 2048]----+
|       o=.       |
|    o  o++E      |
|   + . Ooo.      |
|    + O B..      |
|     = *S.       |
|      o          |
|                 |
|                 |
|                 |
+-----------------+
```

<!-- More -->
# 为什么会有 randomart image

相比超长字符串，人们更容易接受图形。让我们对比两幅图片的差异比对比两个超长字符串也要容易的多。这就是为什么现在大家使用二维码，而不是复制粘贴 URL 的原因。

Randomart image 通过将 Key 转换成有规律的图片，让人可以更加容易的、快速的对比 Key 的异同。

# 趣闻

在[《The drunken bishop: An analysis of the OpenSSH
fingerprint visualization algorithm》](http://aarontoponce.org/drunken_bishop.pdf)中，作者通过一段有趣的故事来表达 randomart image 生成的过程：

> Peter 主教发现自己在一个封闭的矩形房间内，四面都是墙壁，而地板上又铺满了黑白交替矩形的瓷砖。Peter 主教突然开始头疼——大概应为之前喝了太多的酒——于是开始随意的走动起来。准确的说，他是按照对角走位的方式，就好像国际象棋上的主教一样。当他遇到墙壁的时候，如果他踩着黑瓷砖，就走向白瓷砖，如果踩着白瓷砖就走向黑瓷砖。每次动作之后，他都会在瓷砖上放置一个硬币，记录他踩过这里一次。走了 64 步之后，用完了所有的硬币，Peter 突然醒了过来。多么奇怪的梦！

# 如何生成

看了上面的故事，来说一说 randomart image 具体是如何生成的。

我们知道 OpenSSH Key 的指纹是一个 MD5 校验和，同事可以用 16 进制表示出来，类似这样：

```
fc:94:b0:c1:e5:b0:98:7c:58:43:99:76:97:ee:9f:b7
```

当然，也可以用二进制来表示。

于是按照 Peter 的走路方式，我们定义如下走位：

1. "00" 表示向西北（左上）移动
2. "01" 表示向东北（右上）移动
3. "10" 表示向西南（右下）移动
4. "11" 表示向东南（坐下）移动

就像这样：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-01.jpg)


我们来画个图，用于表示黑白相间的瓷砖房，并每个格子上都编号，一开始的时候，Peter 在房间的中间：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-02.jpg)

中间的 76 号就是 Peter 最初的位置。我们可以把 Peter 所在的格子表示为：

```
// 用坐标系表示
P = x + 17y
```


那么，从 76 号向四个方向移动 P 值变化：

1. "00" 进入 58 号格子，数值 -18
2. "01" 进入 60 号格子，数值 -16
3. "10" 进入 92 号格子，数值 +16
4. "11" 进入 94 号格子，数值 +18

对于碰墙的情况，做如下规则：

将房间的每个格子做分类，四个角分别为 a、b、c、d，靠着四面墙的分别为 T、R、B、L，其余部分为 M。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-03.jpg)

对于每一种情况：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-04.jpg)

解释一下上图，以 a 栏为例，如果下一步像左上移动，但是继续走就撞墙了，所以实际移动为 `不移动`；如果向右上移动，实际移动为 `向右移动`；如果向左下移动，实际移动为 `向下移动`；如果像右下移动，实际移动就是 `向下移动`。

那么 Peter 投掷硬币是怎样体现的？实际上，是做如下统计，每个格子，如果没有被踩过，则不做表示；如果被踩过一次，记录为 `.`；如果被踩了两次，记录为 `o`...具体如下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-05.jpg)

比较特别的是 `S` 和 `E`，`S` 和 `E` 分别标记起始和终止位置（所以 76 号格子永远是 S）。

实际操作一下，对于 `fc:94:b0:c1:e5:b0:98:7c:58:43:99:76:97:ee:9f:b7`，我们将其转换成二进制：

```
11010100:11010100:11111101:11001010:...:00111011:11100100:10111010:11101001
```

将二进制二位一组，按照下表做好走路的顺序：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-06.jpg)

按照上面的顺序，踩到格子的顺序如下。

76, 58, 76, 94, 112, 94, 78, 62, 78, 60, 42, 60, 76, 60, 42, 24, 42, 26, 10, 26, 44, 26, 8, 26, 42, 24, 40, 24, 40, 22, 40, 58, 42, 24, 40, 24, 8, 26, 8, 7, 8, 9, 25, 9, 25, 41, 25, 43, 27, 45, 29, 13, 29, 45, 63, 79, 97, 115, 133, 117, 133, 151, 135, 152, 151

最后做成统计，填好图，成品就如下。是不是就和我们平常在控制台得到的输出一致？

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_ssh-keygen%20%E4%B8%AD%E7%94%9F%E6%88%90%E7%9A%84%20randomart%20image%20%E6%98%AF%E4%BB%80%E4%B9%88-07.jpg)

# 相关

> [What is randomart produced by ssh-keygen?](https://superuser.com/questions/22535/what-is-randomart-produced-by-ssh-keygen)
> [OpenSSH Keys and The Drunken Bishop](https://pthree.org/2013/05/30/openssh-keys-and-the-drunken-bishop/)
> [The drunken bishop: An analysis of the OpenSSH
fingerprint visualization algorithm](http://aarontoponce.org/drunken_bishop.pdf)

*最后一点说明：其主要原理都发表在 [The drunken bishop: An analysis of the OpenSSH
fingerprint visualization algorithm](http://aarontoponce.org/drunken_bishop.pdf)*

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)



> 如有任何知识产权、版权问题或理论错误，还请指正。
>
> 转载请注明原作者及以上信息。
