---
{"dg-publish":true,"permalink":"/Article/simpread-iOS 截屏防护 - 担心 App 内容被截屏泄露吗？这个开源库就是你要的/"}
---

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/469526162)

前言
--

想必很多同学都遇到过想要防止截屏的场景，但通过现有的 API 只能监听到截屏完成的通知。

试了一些主流应用，发现很多都想去防止截图，但是最终实现的效果并不理想，只能在截图完成后去做一些提示，甚至访问相册删除图片。看起来好像是一个不好解决的问题。

经过一段时间的探索，我封装了一个十分轻量便捷的工具。希望能有些帮助，欢迎交流！

一、 常见方案
-------

### 1.1 系统通知

`UIApplication.userDidTakeScreenshotNotification` 这个系统通知是在完成截屏动作后，系统给到 `App` 的，在收到这个通知后做处理，并不能达到防护的效果。

微信的付款吗就是这样的实现，在截屏后提供一个警示内容。

![](https://pic4.zhimg.com/v2-fc838ef725c2c3d101569c6c9eafc623_r.jpg)

### 1.2 小红书

在使用小红书的截屏的时候顶部状态栏会添加一个小红书的水印如下图：

![](https://pic4.zhimg.com/v2-8cca4f6928e7f02d35a16c7f9d47a507_r.jpg)

它是怎么实现的呢？我有两个猜测：

**第一种：水印一直在，刘海后面**

因为非刘海屏没有这个水印，所以估计是这样，但是通过切换到任务管理器状态并没有看到有那个水印。

![](https://pic4.zhimg.com/v2-5f242b98e73fda477b57827c7e847c23_r.jpg)

**第二种：通过什么黑科技加上的？**

不是第一种，难道真的是又什么黑科技？

**继续验证**

我觉得肯定是第一种，觉得任务切换的状态下是不是那个水印被系统隐藏掉。想道还有 QuickTime 可以投屏来验证。

![](https://pic3.zhimg.com/v2-8ea76f74af7f283f2c28d9b9e2fec4aa_r.jpg)

破案，果然是藏在刘海后面的，没有继续深入的价值了 ‍♂️。

### 1.3 ScreenShieldKit

![](https://pic1.zhimg.com/v2-ba61b48aab4b30ae1d08252ecd8717e4_b.gif)

在寻找解决方案的时候发现了这样一个 SDK [ScreenShieldKit](https://link.zhihu.com/?target=https%3A//screenshieldkit.com/)

> Contact us for more information and to receive a free evaluation SDK!  
> 这句话一看就是收钱的了，抱着能白嫖绝不三连的宗旨，先略过暂不深入研究。（开玩笑，如果你觉得我的方案不错，本文求赞评转 + Star！）

### 1.4 爱奇艺

爱奇艺使用的是截图后直接读取用户相册删除图片的方式进行的，不给权限，截图就不会被删除。

![](https://pic3.zhimg.com/v2-19be520010dd611f512299ece20a571a_r.jpg)

### 1.5 UITextField

当我们将一个 `UITextField` 的 `isSecureTextEntry` 设置为 `true` 的时候，会隐去输入的文案用 `圆点` 替代。并且在进行录屏或者截屏的时候都会被系统隐去。

下面用我的个人项目 [梦见账本](https://link.zhihu.com/?target=https%3A//apps.apple.com/cn/app/id1498426607)

如下图我正在用 `QuickTime` 进行录屏：

![](https://pic1.zhimg.com/v2-858851f6116b6d957c8d91e5bcaab7dc_r.jpg)

那么我们是否可以使用这种特性呢？

二、 RyukieSwifty/ScreenShield
----------------------------

基于 `UITextField` 的效果我实现了一个 `ScreenShieldView` 可以很方便的进行使用。

[GitHub: RyukieSwifty/ScreenShield](https://link.zhihu.com/?target=https%3A//github.com/RyukieSama/Swifty)

_**如果觉得不错的话，欢迎留个⭐️哦**_

### 2.1 使用方式

**Cocoapods 导入**

```
pod 'RyukieSwifty/ScreenShield'

```

**使用**

例如对整个控制器的 `View` 进行截屏防护：

```
import UIKit
import RyukieSwifty

class TransactionAddViewController: UIViewController {
    // MARK: - Life
    override func loadView() {
        view = ScreenShieldView.create()
    }
    ...
}

```

### 2.2 如何实现的

![](https://pic2.zhimg.com/v2-efab02443992e22f35af586dfd1ac245_r.jpg)

按照向里面添加子视图的方式验证具体的原因，最后发现这个效果是由一个私有类实现的 `_UITextLayoutCanvasView` 。他本质上也是一个 `UIView` ，所以理论上只要我们能够创建出来一个它，那么就可以将想要保护的内容添加进去。

由于它是私有类，无法直接创建，并且如果直接通过字符串区创建也担心有审核风险，于是我通过下面的方式来创建。

```
private func makeSecureView() -> UIView? {
    let field = UITextField()
    field.isSecureTextEntry = true
    let fv = field.subviews.first
    fv?.subviews.forEach { $0.removeFromSuperview() }
    fv?.isUserInteractionEnabled = true
    return fv
}

```

> 当前手头没有更多的系统版本设备可以测试，已经验证的 `13.7 ～ 15.3.1` 是没问题的。  
> 不排除会存在某些版本上 `UITextField` 的结构发生了调整导致失效的问题，如果有，欢迎在 [GitHub](https://link.zhihu.com/?target=https%3A//github.com/RyukieSama/Swifty) 提 `issue` 。

总结
--

不知道各位有没有其他的实现方式，有的话欢迎一起交流。

如果本文有帮到你，欢迎赞评转 + Star！

关于我
---

*   [技术文章归档](https://link.zhihu.com/?target=https%3A//ryukiedev.gitbook.io/wiki/)
*   [Github](https://link.zhihu.com/?target=https%3A//github.com/RyukieSama)

个人项目：

[‎扫雷：无尽天梯 - Elic​apps.apple.com/cn/app/id1488204246](https://link.zhihu.com/?target=https%3A//apps.apple.com/cn/app/id1488204246)[‎梦见账本 - 极速智能记账​apps.apple.com/cn/app/id1498426607](https://link.zhihu.com/?target=https%3A//apps.apple.com/cn/app/id1498426607)[‎隐私访问记录 - 系统性分析应用隐私权限访问活动洞见隐私泄露](https://link.zhihu.com/?target=https%3A//apps.apple.com/cn/app/id1590992377)