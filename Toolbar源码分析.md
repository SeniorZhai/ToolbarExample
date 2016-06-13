Toolbar作为ActionBar的完美替代品，具有低版本支持，高度自定义的特性。现在让我们了解下Toolbar的源码实现吧。

##特点
首先Toolbar继承于[ViewGroup](http://developer.android.com/intl/es/reference/android/view/ViewGroup.html)，可以像使用一个普通的View一样使用Toolbar，可以放在任何位置。
![](./device.png)
##自定义属性
```xml
<declare-styleable name="Toolbar">
        <attr name="titleTextAppearance" format="reference" />
        <attr name="subtitleTextAppearance" format="reference" />
        <attr name="title" />
        <attr name="subtitle" />
        <attr name="android:gravity" />
        <attr name="titleMargins" format="dimension" />
        <attr name="titleMarginStart" format="dimension" />
        <attr name="titleMarginEnd" format="dimension" />
        <attr name="titleMarginTop" format="dimension" />
        <attr name="titleMarginBottom" format="dimension" />
        <attr name="contentInsetStart" />
        <attr name="contentInsetEnd" />
        <attr name="contentInsetLeft" />
        <attr name="contentInsetRight" />
        <attr name="maxButtonHeight" format="dimension" />
        <attr name="collapseIcon" format="reference" />
        <attr name="collapseContentDescription" format="string" />
        <attr name="popupTheme" />
        <attr name="navigationIcon" format="reference" />
        <attr name="navigationContentDescription" format="string" />
        <attr name="android:minHeight" />
        <attr name="logo" />
        <attr name="logoDescription" format="string" />
        <attr name="titleTextColor" format="color" />
        <attr name="subtitleTextColor" format="color" />
</declare-styleable>
```
可以看到,属性主要围绕title、subtitle、content的属性设置

##重要的子View
- mMenuView
- mTitleTextView
- mSubtitleTextView
- mNavButtonView
- mLogoView

##重要的方法
- getPopupTheme/setPopupTheme 设置/获取Menu的Theme
- onRtlPropertiesChanged 设置RTL
- setLogo 设置Logo
- isOverflowMenuShowing 更多(弹出菜单)是否显示
- showOverflowMenu 显示OverflowMenu(弹出菜单)
- hideOverflowMenu 隐藏OverflowMenu(弹出菜单)
- setTitle/getTitle 设置/获取标题
- setSubtitle/getSubtitle 设置/获取副标题
- setNavigationIcon/getNavigationIcon 设置navigation button的图标
- setOverflowIcon 设置弹出菜单的icon

作为一个ViewGroup，最重要的两个方法当然是onMeasure和onLyout方法。
###Measure方法
[onMeasure](https://gist.github.com/SeniorZhai/661dfba5f933d25f5c8732ca810a5e0f#file-measure)方法中，先后对mNavButtonView、mCollapseButtonView、mMenuView、mExpandedActionView、mLogoView进行measure
```
...
if (shouldLayout(mNavButtonView)) {
    // 根据widthMeasureSpec、heightMeasureSpec测量子View
    measureChildConstrained(mNavButtonView, widthMeasureSpec, width, heightMeasureSpec, 0, mMaxButtonHeight);
    navWidth = mNavButtonView.getMeasuredWidth() + getHorizontalMargins(mNavButtonView);
    height = Math.max(height, mNavButtonView.getMeasuredHeight() + getVerticalMargins(mNavButtonView));
    // 合并子元素的测量状态
    childState = ViewUtils.combineMeasuredStates(childState, ViewCompat.getMeasuredState(mNavButtonView));
}
...
```
值得注意的是onMesure方法中对Menu菜单的测量
```
    int menuWidth = 0;
    if (shouldLayout(mMenuView)) {
        measureChildConstrained(mMenuView, widthMeasureSpec, width, heightMeasureSpec, 0, mMaxButtonHeight);
        menuWidth = mMenuView.getMeasuredWidth() + getHorizontalMargins(mMenuView);
        height = Math.max(height, mMenuView.getMeasuredHeight() + getVerticalMargins(mMenuView));
        childState = ViewUtils.combineMeasuredStates(childState,
        ViewCompat.getMeasuredState(mMenuView));
    }
```
```
    private void measureChildConstrained(View child, int parentWidthSpec, int widthUsed,
            int parentHeightSpec, int heightUsed, int heightConstraint) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        int childWidthSpec = getChildMeasureSpec(parentWidthSpec,
                getPaddingLeft() + getPaddingRight() + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        int childHeightSpec = getChildMeasureSpec(parentHeightSpec,
                getPaddingTop() + getPaddingBottom() + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        final int childHeightMode = MeasureSpec.getMode(childHeightSpec);
        if (childHeightMode != MeasureSpec.EXACTLY && heightConstraint >= 0) {
            final int size = childHeightMode != MeasureSpec.UNSPECIFIED ?
                    Math.min(MeasureSpec.getSize(childHeightSpec), heightConstraint) :
                    heightConstraint;
            childHeightSpec = MeasureSpec.makeMeasureSpec(size, MeasureSpec.EXACTLY);
        }
        child.measure(childWidthSpec, childHeightSpec);
    }
```

###Layout方法
[onLayout](https://gist.github.com/SeniorZhai/661dfba5f933d25f5c8732ca810a5e0f#file-layout)方法中,先后对mNavButtonView、mCollapseButtonView、mMenuView、mExpandedActionView、mLogoView进行layout
值得注意的是,官方的View是要对RTL进行支持的,所以在Layout的时候要考虑RTL的问题,如果有需要,我们也应该在自定义View的时候考虑RTL的排列。
```
if (shouldLayout(mNavButtonView)) {
    if (isRtl) {
        right = layoutChildRight(mNavButtonView, right, collapsingMargins, alignmentHeight);
    } else {
        left = layoutChildLeft(mNavButtonView, left, collapsingMargins, alignmentHeight);
    }
}
```
> 注:[关于RTL](https://developer.android.com/about/versions/android-4.2.html#RTL)

###ExpandedActionViewMenuPresenter
ExpandedActionViewMenuPresenter实现MenuPresenter接口,用于构建弹出菜单。
MenuPresenter从MenuBuilder对象读取菜单项并生成相应的菜单项子视图，创建的菜单项子视图被添加到菜单容器视图中，MenuPresenter也提供对MenuBuilder对象及其菜单项的获取及其它操作。
```
    private class ExpandedActionViewMenuPresenter implements MenuPresenter {
        MenuBuilder mMenu;
        MenuItemImpl mCurrentExpandedItem; //对应菜单项的具体实现类

        @Override
        public void initForMenu(Context context, MenuBuilder menu) {
            // 当menu变动时重置ExpandedActionView
            if (mMenu != null && mCurrentExpandedItem != null) {
                mMenu.collapseItemActionView(mCurrentExpandedItem);
            }
            mMenu = menu;
        }

        @Override
        public MenuView getMenuView(ViewGroup root) {
            return null;
        }

        @Override
        // 更新ActionMenuView 的子View
        public void updateMenuView(boolean cleared) {
            if (mCurrentExpandedItem != null) {
                boolean found = false;

                if (mMenu != null) {
                    final int count = mMenu.size();
                    for (int i = 0; i < count; i++) {
                        final MenuItem item = mMenu.getItem(i);
                        if (item == mCurrentExpandedItem) {
                            found = true;
                            break;
                        }
                    }
                }

                if (!found) {
                    collapseItemActionView(mMenu, mCurrentExpandedItem);
                }
            }
        }

        @Override
        public void setCallback(Callback cb) {
        }

        @Override
        public boolean onSubMenuSelected(SubMenuBuilder subMenu) {
            return false;
        }

        @Override
        public void onCloseMenu(MenuBuilder menu, boolean allMenusAreClosing) {
        }

        @Override
        public boolean flagActionItems() {
            return false;
        }

        @Override
        // 展开时调用
        public boolean expandItemActionView(MenuBuilder menu, MenuItemImpl item) {
            ensureCollapseButtonView();
            if (mCollapseButtonView.getParent() != Toolbar.this) {
                addView(mCollapseButtonView);
            }
            mExpandedActionView = item.getActionView();
            mCurrentExpandedItem = item;
            // 添加ExpandedActionView到菜单项下
            if (mExpandedActionView.getParent() != Toolbar.this) {
                final LayoutParams lp = generateDefaultLayoutParams();
                lp.gravity = GravityCompat.START | (mButtonGravity & Gravity.VERTICAL_GRAVITY_MASK);
                lp.mViewType = LayoutParams.EXPANDED;
                mExpandedActionView.setLayoutParams(lp);
                addView(mExpandedActionView);
            }

            removeChildrenForExpandedActionView();
            requestLayout();
            item.setActionViewExpanded(true);

            if (mExpandedActionView instanceof CollapsibleActionView) {
                ((CollapsibleActionView) mExpandedActionView).onActionViewExpanded();
            }

            return true;
        }

        @Override
        // 折叠时调用
        public boolean collapseItemActionView(MenuBuilder menu, MenuItemImpl item) {
            if (mExpandedActionView instanceof CollapsibleActionView) {
                ((CollapsibleActionView) mExpandedActionView).onActionViewCollapsed();
            }

            removeView(mExpandedActionView);
            removeView(mCollapseButtonView);
            mExpandedActionView = null;

            addChildrenForExpandedActionView();
            mCurrentExpandedItem = null;
            requestLayout();
            item.setActionViewExpanded(false);

            return true;
        }

        @Override
        public int getId() {
            return 0;
        }

        @Override
        public Parcelable onSaveInstanceState() {
            return null;
        }

        @Override
        public void onRestoreInstanceState(Parcelable state) {
        }
    }
```

###CustomView
在onLayout方法中我们可以看到针对一系列CustomView的处理,Toolbar也有LayoutParams内部类
Toolbar并没有直接设置CustomView的方法,我们要通过ActionBar进行设置
```
getSupportActionBar().setCustomView(mCustomView, params);
```

