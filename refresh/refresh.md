# 下拉刷新与上划加载

## 1.关于滑动时的一些方法

**滑动对象**
scrollTo和scrollBy用于滑动View的内容，而不是改变View本身所处的位置。所以，单独的View滑动很少见，更多的是ViewGroup调用scroll方法滑动子控件的位置。比如，使用TextView对象调用scrollTo或者ScrollBy方法，会发现TextView里面的文本内容的位置发生改变，而TextView本身所处的位置没有变化。

**getScrollX（）和getScrollY（）**
返回值mScrollX和mScrollY分别表示距离起始位置的X轴或Y轴方向上的偏移量，而不是View在X轴或Y轴方向上的坐标值，用于记录偏移增量的两个变量。所以，mScrollX和mScrollY的初始值为0和0。

**scrollTo（int x， int y）**
从源码中可以看出，scrollTo移动的目标位置x和y值是以初始化的mScrollX和mScrollY为参考点的，只要位置发生偏移，就对mScrollX和mScrollY赋新值。注意，这里的（x，y）理解为针对初始化值的偏移量，比如，我想移动到（100,100）这个位置，那么偏移量就是（0,0）-（100,100）=（-100，100），所以调用的时候就是view.scrollTo（-100, -100），这样才能达到我们想要的偏移效果。

**scrollBy（int x， int y）**
从源码中可以看到，scrollBy依旧调用了scrollTo方法：scrollTo(mScrollX + x, mScrollY + y)，只是参考点不再是初始化时的（0,0），相对于当前位置的偏移量。

### 小结

1. **scrollTo方法中的参数会被保存为mScrollX和mScrollY，这2个参数用作getScrollX和getScrollY的返回。**参考如下：
2. **当scrollTo中的参数为正数时，x方向为向左移动，y方向为向上移动，负数时相反**



> https://www.cnblogs.com/virtual-young/p/4578424.html

以上参考链接中包含getRawX和getX的区别，getRawX是相对于整个屏幕窗口；而getX是相对于父视图。



## 2.判断是否还能下滑，即是否能显示子视图中上方的内容

**1. Ultra-Pull-To-Refresh判断**

> 

```java
public static boolean canChildScrollUp(View view) {
    if (android.os.Build.VERSION.SDK_INT < 14) {
        if (view instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) view;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                    .getTop() < absListView.getPaddingTop());
        } else {
            return view.getScrollY() > 0;
        }
    } else {
        return view.canScrollVertically(-1);
    }
}
```

**2. SwipRefreshLayout判断** 

>

```java
/**
 * @return Whether it is possible for the child view of this layout to
 *         scroll up. Override this if the child view is a custom view.
 */
public boolean canChildScrollUp() {
    if (android.os.Build.VERSION.SDK_INT < 14) {
        if (mTarget instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) mTarget;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                            .getTop() < absListView.getPaddingTop());
        } else {
            return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
        }
    } else {
        //Negative to check scrolling up, positive to check scrolling down.
//第二个参数direction负值判断用户是否可以上滑，正值判断用户是否可以下滑
        return ViewCompat.canScrollVertically(mTarget, -1);
    }
}
```



![](E:\Project\Android-Learn\src\refresh_1.jpg)



```java
x = left + translationX 

y = top + translationY
```

![](E:\Project\Android-Learn\src\refresh_2.png)