**在本文开始之前，先做一个简短声明：本文提到的所有与 ShadowSocks 以及 clashx 仅仅作为技术讨论！！**

##### **引子**

故事要从升级 MacOS 的系统开始，工作快下班的时候，也不知道怎么想的，把工作机升级一把系统，简简单单的点击一下，没想到要付出几天的代价。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84eecdd5cba641c089bc0518ea5fdca6~tplv-k3u1fbpfcp-watermark.image?)

在升级之后，发现不能够使用 Google 搜索了，我以为是账号过期了，但是发现并不是。再然后，我回家打开了自己的电脑，使用 shadowsocks 很丝滑，然后，然后！！我就作死的把我的订阅全部删除，重新订阅——订阅失败。至此，我工作的电脑和我的电脑，都不能 Google 搜索了。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07cf3e0d32254c968b9c0523fccba3ca~tplv-k3u1fbpfcp-watermark.image?)

我人麻了。

经过了两天的不断调试，我放弃了。修不好，有些可以使用，但是版本太过老旧。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3eb50a2d29a4444aad56743678a0329e~tplv-k3u1fbpfcp-watermark.image?)

这一款可以继续使用，但是，我觉得很难受，可没有订阅了，PAC 好像也不生效。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22167574208549c48d08d6b3cbcd9ccd~tplv-k3u1fbpfcp-watermark.image?)

**转角遇到爱**

之后发现了 ClashX，真的蛮好用的。哈哈哈！

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73fced9ef7654799945c703c5b50904d~tplv-k3u1fbpfcp-watermark.image?)

#### **正式教学**

本人使用的版本是最新的，[下载地址](https://github.com/yichengchen/clashX/releases)

##### （一）选择配置->托管配置->管理：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1eda13aba6a465c9a00b6472549a88a~tplv-k3u1fbpfcp-watermark.image?)

##### （二）添加自己的订阅地址：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/104dc20592ab40fcbf3d92010dad8667~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cd8b1ac83814b8189ab1cc8af41c9f8~tplv-k3u1fbpfcp-watermark.image?)

大功告成！！

##### （三）自定义规则，打开配置文件夹，选择对应的文件夹

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/557db32abb9f4324ac4082fddf4b082a~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/665fc4e7712842d7822875d253b3a0ea~tplv-k3u1fbpfcp-watermark.image?)

##### （四）添加配置

这里只是举个例子

```yaml
DOMAIN-SUFFIX,s3.amazonaws.com,Proxy
```

DOMAIN-SUFFIX 表示域名匹配，Proxy 是你们自己的代理规则

#### 小结

没事别乱升级。

##### 参考文章：

[clash 配置自定义规则](https://tomorrow505.xyz/clash%E9%85%8D%E7%BD%AE%E8%87%AA%E5%AE%9A%E4%B9%89%E8%A7%84%E5%88%99/)

[clash rules](https://github.com/Loyalsoldier/clash-rules)
