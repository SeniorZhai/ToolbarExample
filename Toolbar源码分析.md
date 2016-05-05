Toolbar作为ActionBar的完美替代品，具有低版本支持，高度自定义的特性。现在让我们了解下Toolbar的源码实现吧。

#继承关系
首先Toolbar继承于`ViewGroup`，所有Toolbar是可以嵌套子View的。
##重要的子View
mMenuView、mTitleTextView、mSubtitleTextView、mNavButtonView、mLogoView
![](./device.png)
