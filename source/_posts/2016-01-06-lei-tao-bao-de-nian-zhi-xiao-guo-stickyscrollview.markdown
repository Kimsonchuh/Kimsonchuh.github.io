---
layout: post
title: "类淘宝的粘滞效果-StickyScrollView"
date: 2016-01-06 10:01:29 +0800
comments: true
categories: Android
---
###类淘宝的粘滞效果-StickyScrollView

####简介
对于许多电商的App都在详情页中使用了一种粘滞的效果，这种效果是由淘宝首先发起使用的，后来许多电商争相效仿。之前做的一个水果商城的相关项目中也使用了这种效果，但是当时是参考了别人开源的代码，一个自定义View，[ScrollContainer](http://www.2cto.com/kf/201408/326934.html)。该类可以实现类似于淘宝App那种粘滞的效果，但是体验不是很好，所以这里就干脆自己重新编写一个。

####目录结构
* 效果描述
* 实现思路
* 实现代码
* 代码分析

#####效果描述
如果没有对这种效果了解过，可以打开淘宝或京东的客户端中产品的详情页体验一下。具体效果就是一个页面中，主要分成两部分，这里称为上部分和下部分；用户进入该页面时只显示上部分，下部分只是一个网页，使用webview（此时webview还没有进行加载数据）。当用户滑动上部分到屏幕底部时，会提示向上滑动以查看更具体的详情，当用户向上滑动到距离时，下部分会自动弹到页面顶部并且加载数据。而用户向上滑动小于指定距离时，则会返回原来的位置。而下部分弹回上部分也是同样的原理。

#####实现思路
根据[ScrollContainer](http://www.2cto.com/kf/201408/326934.html)的实现思路，该类是继承RelativeLayout，也是有两个子view，上部分是一个ScrollView，下部分是一个webView。当用户滑到ScrollView到底部时，判断是否满足跳转页面的条件，如果满足则记录距离和滑动速度，并触发监听，然后该类进行刷新，重新调用measure和onlayout方法，然后根据之前记录的距离和速度进行刷新页面，从而达到滑动的效果。但是这样做的缺点是用在绘制页面的资源消耗较大，而且对于滑动速度的自定义程度小，所以我重新设计了一个实现方案。我这里实现的思路与上面所述的ScrollContainer基本相同，不同的是使用了scroller滑动类对滑动进行控制，该类可以实现对滑动速度的自定义，给开发者提供了更大的扩展性。

#####实现代码

```java
public class StickyScrollView extends ScrollView {
    public static final String TAG = StickyScrollView.class.getSimpleName();
    public static final int PAGE_TOP = 0;
    public static final int PAGE_BOTTOM = 1;
    public static final double PERCENT = 0.4;
    public static final int ANIMATION_DURATION = 180;
    public static final int TOUCH_DURATION = 150;

    private ViewGroup mChildLayout;
    private View mTopChildView;

    private Context mContext;
    private OnPageChangeListener onPageChangeListener;

    private boolean isScrollAuto;           //判断是否自由滚动
    private Scroller mScroller;             //滑动类
    private int screenHeight;               //屏幕高度
    private int offsetDistance;             //topview的高度与屏幕的高度差
    private int topChildHeight;             //topview的高度
    private boolean isTouch;                //用户是否在触控屏幕
    private int currentPage;                //值为0时屏幕显示topview，值为1时屏幕显示bottomview
    private long downTime;                  //用户按下屏幕的时间戳
    private long upTime;                    //用户抬起时的时间戳
    private int downY;                      //用户按下屏幕的y坐标
    private int upY;                        //用户抬起的y坐标
    private boolean isPageChange;           //页面是否切换

    public StickyScrollView(Context context) {
        super(context);
        this.mContext = context;
    }

    public StickyScrollView(Context context, AttributeSet attrs) {
        super(context, attrs);
        this.mContext = context;
    }

    public StickyScrollView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        this.mContext = context;
    }

    public void setOnPageChangeListener(OnPageChangeListener onPageChangeListener) {
        this.onPageChangeListener = onPageChangeListener;
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);

        mChildLayout = (ViewGroup) getChildAt(0);
        mTopChildView = mChildLayout.getChildAt(0);
        topChildHeight = mTopChildView.getMeasuredHeight();
        screenHeight = getMeasuredHeight();
        offsetDistance = topChildHeight - screenHeight;
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                isTouch = true;
                downY = (int) ev.getY();
                downTime = System.currentTimeMillis();
                if (mScroller != null) {
                    mScroller.forceFinished(true);
                    mScroller = null;
                }
                break;
            case MotionEvent.ACTION_MOVE:
                break;
            case MotionEvent.ACTION_UP:
                isTouch = false;
                upY = (int) ev.getY();
                upTime = System.currentTimeMillis();
                //用户手指在屏幕上的时间
                long duration = upTime - downTime;

                //这里要确保点击事件不失效
                if (Math.abs(upY - downY) > 50) {
                    Log.e(TAG, ">>>ISN_T CLICK:" + Math.abs(upY - downY));
                    if (currentPage == PAGE_TOP) {
                        //下面的判断已经能确定用户是否往上滑
                        if (getScrollY() > offsetDistance) {
                            mScroller = new Scroller(mContext);
                            if (getScrollY() < (screenHeight * PERCENT + offsetDistance) && duration > TOUCH_DURATION) {
                                isPageChange = false;
                                scrollToTarget(PAGE_TOP);
                            } else {
                                //切换到下界面
                                isPageChange = true;
                                isScrollAuto = duration < TOUCH_DURATION ? true : false;
                                scrollToTarget(PAGE_BOTTOM);
                                currentPage = PAGE_BOTTOM;
                            }
                            return false;
                        }
                    } else {
                        if (getScrollY() < topChildHeight) {
                            mScroller = new Scroller(mContext);
                            if (getScrollY() < (topChildHeight - screenHeight * PERCENT) || duration < TOUCH_DURATION) {
                                //切换到上界面
                                isPageChange = true;
                                isScrollAuto = duration < TOUCH_DURATION ? true : false;
                                scrollToTarget(PAGE_TOP);
                                currentPage = PAGE_TOP;
                            } else {
                                isPageChange = false;
                                scrollToTarget(PAGE_BOTTOM);
                            }
                            return false;
                        }
                    }
                }
                break;
        }
        return super.dispatchTouchEvent(ev);
    }

    /**
     * 滚动到指定位置
     */
    private void scrollToTarget(int currentPage) {
        int delta;
        if (currentPage == PAGE_TOP) {
            delta = getScrollY() - offsetDistance;
            mScroller.startScroll(0, getScrollY(), 0, -delta, isScrollAuto == true ? ANIMATION_DURATION : (int) (Math.abs(delta) * 0.4));
        } else if (currentPage == PAGE_BOTTOM) {
            delta = getScrollY() - topChildHeight;
            mScroller.startScroll(0, getScrollY(), 0, -delta, isScrollAuto == true ? ANIMATION_DURATION : (int) (Math.abs(delta) * 0.4));
        }
        postInvalidate();
    }

    @Override
    public void computeScroll() {
        // 调用startScroll的时候scroller.computeScrollOffset()返回true
        super.computeScroll();
        if (mScroller == null) {
            return;
        }
        if (mScroller.computeScrollOffset()) {
            this.scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            postInvalidate();
            if (mScroller.isFinished()) {
                mScroller = null;
                if (onPageChangeListener != null && isPageChange) onPageChangeListener.OnPageChange(currentPage);
            }
        }
    }


    @Override
    protected void onScrollChanged(int l, int t, int oldl, int oldt) {
        super.onScrollChanged(l, t, oldl, oldt);
        //滚动时的监听,当用户触屏滑动时不监听，t == getScrollY
        if (currentPage == PAGE_TOP) {
            if (getScrollY() > offsetDistance && !isTouch) {
                if (mScroller == null) {
                    //用于控制当滑动到分界线时停止滚动
                    scrollTo(0, offsetDistance);
                } else {
                    scrollToTarget(PAGE_TOP);
                }
            }
        } else if (currentPage == PAGE_BOTTOM) {
            if (getScrollY() < topChildHeight && !isTouch) {
                if (mScroller == null) {
                    scrollTo(0, topChildHeight);
                } else {
                    scrollToTarget(PAGE_BOTTOM);
                }
            }
        }
    }

    /**
     * 切换页面完成后的回调
     */
    public interface OnPageChangeListener {
        void OnPageChange(int currentPage);
    }

}

```

#####代码分析
StickyScroll类是继承了ScrollView，一个LinearLayout作为子View，而LinearLayout中包含两个
代码中主要功能的实现集中在以下几个方法中。
1. `onLayout` 复写该方法是为了获取子view中的高度
2. `dispatchTouchEvent` 效果实现的主要逻辑。在用户抬起手指时判断滑动到原来的位置或者是切换页面，并且在切换的时候禁用自动滚动，优化体验。   
3. `scrollToTarget` 根据传入的参数启动Scroller，滚动到指定的偏移量。
4. `computeScroll` 判断是否已经滚动结束，如果是在切换页面的情况下滚动结束，则调用回调方法，通知页面切换。
5. `onScrollChanged` 根据情况是StickyScrollView实现自动滚动或者是滚动到指定的位置。


####总结
实现StickyScrollView这种具有粘滞效果的自定义主要的难点在于如何计算滑动的距离以及自动滚动。只要把这部分的逻辑实现好了之后，基本功能就实现了，剩下的也就是功能优化的修修补补了。最终的实现效果如下：
![](https://github.com/Kimsonchuh/StickScrollView/blob/master/images/stickyScrollview.gif)