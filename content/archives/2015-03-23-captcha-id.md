+++
categories = ["code"]
date = "2015-03-23T16:00:06+08:00"
draft = false
slug = "captcha-id"
title = "浅谈验证码的设计和破解"

+++

关于验证码，这是一个古老的课题，从最原始的数字文本发展到今天的文字图像，验证码的破解和反破解一直没有停止过。
最近12306网站也是换成了图像选择的验证码，由于其背后存在的巨大商业利益，也有人在第一时间进行了分析破解，有人验证说12306的图像验证码是动态生成的。如果是这样的，还是有点吊的，不过如今的图像识别技术确实也很厉害了。有人通过百度的图像识别API就搞定了。不扯了，今天为什么记录这篇文章呢，因为自己做的一个系统暴露在了公网上，而且用的是公司内部的域账号接口做的验证，如果不加验证码的话可能会被人暴力破解通过VPN进入公司内网，那么问题来了，什么样的验证码可以提高破解难度呢？

![](/images/2015/1427097472.png)

###二值化

拿到验证码后，第一步就是把这张图片进行二值化，难度是找到二值化的阈值，有意思的这个阈值影响着正常用户的识别难度，也就是说，只要用户可以识别出这个验证码，我们就可以找到一个阈值把这张图片二值化。阈值找的越精确，识别度越高，后面的工作越简单，阈值的寻找需要用到数学中的一些有意思的算法，感兴趣的同学可以研究下。

二值化后，我们要对这些数据进行降噪，去掉一些干扰信息。

![](/images/2015/1427097546.JPG)

###降噪
图中的红圈中的1即为干扰信息，去掉这些信息还是比较容易的，我们计算这个点上临近8个点之和，如果小于某个阈值即为干扰点。图上三个点周围8个点和都为0，可以判定这个点是干扰点。

###切割
下一步，经过降噪处理后，我们得到了相对纯净的二值化数据，接下来是最为关键和最有难度的切割字符了。我们先垂直切割再水平切割，针对上图的验证码其实算法也并不难，和降噪本质上相似，但是上图这个是比较好切割的，四个字符没有连接的部分，如果有连接的部分，就要花费一番功夫去分析切割了。


###识别
切割完后，就要去识别字符了，在试验中发现，如果字符有扭曲、旋转、缩放、粘连、样式，识别的成功率会大大降低。其实那些干扰线和花哨的背景，并没有提高多少破解难度，我们看看Google的验证码：

![](/images/2015/1427097566.png)

最后，我们知道了什么样的验证码是比较难破解的，我们只是在技术上探讨了下，并没有利用OCR等技术来提高破解率。于是开始着手给系统加上验证码：

![](/images/2015/1427097913.png)

