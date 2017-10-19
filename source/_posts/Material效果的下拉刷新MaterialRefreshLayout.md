---
title: Material效果的下拉刷新MaterialRefreshLayout
date: 2016-12-21 15:48:42
tags:
---
Material效果的下拉刷新MaterialRefreshLayout
<!--more-->

效果图
![images](https://github.com/gaoyuyu/CustomRefreshLayoutDemo/raw/master/captures/refresh.gif)

----------
参照[android-cjj/BeautifulRefreshLayout](https://github.com/android-cjj/BeautifulRefreshLayout)修改而来。

核心知识点
 1. WaveView
 2. 自定义FrameLayout
 

####一、WaveView实现涟漪效果
涟漪效果是由贝塞尔曲线绘制而来，其中headHeight是上方矩形的高度，controlX和controlY是贝塞尔曲线控制点X,Y坐标点。
成员属性
```Java
    //屏幕宽度
    private int mWidth;
    //屏幕高度
    private int mHeight;
    //头部矩形高度
    private int headHeight;
    //贝塞尔曲线控制点X坐标值
    private int controlX;
    //控制点Y坐标值
    private int controlY;
    //颜色
    private int waveColor =R.color.colorPrimaryDark;
    //画笔
    private Paint paint;
    //Path
    private Path path;
```
重写构造方法并初始化
```Java
    public WaveView(Context context)
    {
        this(context, null, 0);
    }

    public WaveView(Context context, AttributeSet attrs)
    {
        this(context, attrs, 0);
    }

    public WaveView(Context context, AttributeSet attrs, int defStyleAttr)
    {
        super(context, attrs, defStyleAttr);
        init();
    }
    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public WaveView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)
    {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
```
重写onSizeChanged获取屏幕高度和屏幕宽度
```Java
    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh)
    {
        super.onSizeChanged(w, h, oldw, oldh);
        mWidth = w;
        mHeight = h;
    }
```
接下来就可以开始绘制图形，没什么可说。
```Java
    @Override
    protected void onDraw(Canvas canvas)
    {
        super.onDraw(canvas);
        path.reset();
        path.lineTo(0, headHeight);
        path.quadTo(controlX, headHeight + controlY, mWidth,headHeight);
        path.lineTo(mWidth, 0);
        canvas.drawPath(path, paint);
    }
```
####二、自定义FrameLayout
主要成员变量
```Java
    //子布局
    private View mChildView;

    //头部的高度
    protected float mHeadHeight = 100;

    //贝塞尔曲线控制点Y轴坐标值
    protected float mControlY = 180;

    //刷新的状态
    protected boolean isRefreshing;

    //触摸获得Y的位置
    private float mTouchY;

    //当前Y的位置
    private float mCurrentY;

    //当前头部布局高度
    protected int mCurrentHeaderHeight = 0;
    //子view在Y轴上移动的距离
    protected float offsetY = 0;
    //刷新回调接口
    private OnMaterialRefreshListener onMaterialRefreshListener;
```
其中OnMaterialRefreshListener是刷新回调接口。

布局一开始加载进来，首先执行的是onAttachedToWindow(),在这里需要将头部布局`header.xml`加载进来，并addView()，`header.xml`比较简单，只有一个ImageView（箭头）、TextView（显示文字）、ProgressBar。**注意到有一个`setRefreshing(isRefreshing)`方法也在这里调用了，稍后解释**，这里也对mChildView，也就是通过`addView()`添加进来的头部布局设置了属性动画监听器，主要是为了实时重绘，改变高度。
```Java
    @Override
    protected void onAttachedToWindow()
    {
        super.onAttachedToWindow();
        Log.i(LOG_TAG, "onAttachedToWindow");

        mHeaderLayout = LayoutInflater.from(getContext()).inflate(R.layout.header, null);
        mWaveView = (WaveView) mHeaderLayout.findViewById(R.id.waveview);
        mTip = (TextView) mHeaderLayout.findViewById(R.id.tip);
        mProgressBar = (ProgressBar) mHeaderLayout.findViewById(R.id.progressbar);
        mArrow = (ImageView) mHeaderLayout.findViewById(arrow);


        this.addView(mHeaderLayout);

        mChildView = getChildAt(0);
        //此时getChildCount()为2，因为上面调用了addView()，以及还有一个子view，所以有2个。index为0的View为头部布局
        if (getChildCount() > 2)
        {
            throw new RuntimeException("Can only have a child view");
        }

        if (mChildView == null)
        {
            return;
        }

        setRefreshing(isRefreshing);
        
        ViewPropertyAnimator childViewPropertyAnimator = mChildView.animate();
        childViewPropertyAnimator.setInterpolator(new DecelerateInterpolator());
        childViewPropertyAnimator.setUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                int childHeight = (int) mChildView.getTranslationY();
                mHeaderLayout.getLayoutParams().height = childHeight;
                mHeaderLayout.requestLayout();
            }
        });
    }
```
接下来需要处理触摸事件，响应触摸事件。重写View的`onInterceptTouchEvent`和`onTouchEvent`，其中`onInterceptTouchEvent`是拦截事件，返回true表示触摸事件被拦截，false不拦截；`onTouchEvent`是响应事件，返回true表示该触摸事件得到处理，false不做响应。`canChildScrollUp()`判断是否可以上拉，在SwipeRrefreshLayout的源码中出现过。

在`onInterceptTouchEvent()`方法中，主要对单指按下和移动动作进行处理，单指按下时获取手指到该view所在坐标系的Y轴距离（注意，不是屏幕默认坐标系，是view所在坐标系），同时，在单指移动时获取偏移量distanceY，相应做出拦截。
```Java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev)
    {
        //刷新状态拦截事件,不做任何处理
        if (isRefreshing)
        {
            return true;
        }
        int action = ev.getAction();
        switch (action)
        {
            //单点触摸按下动作
            case MotionEvent.ACTION_DOWN:
                mTouchY = ev.getY();
                break;
            //单点触摸离开动作
            case MotionEvent.ACTION_UP:
                break;
            //单点触摸移动动作
            case MotionEvent.ACTION_MOVE:
                float currentY = ev.getY();
                float distanceY = currentY - mTouchY;
                if (distanceY > 0 && !canChildScrollUp())
                {
                    return true;
                }
                break;
        }
        return super.onInterceptTouchEvent(ev);
    }
    /**
     * 判断是否可以上拉
     *
     * @return boolean
     */
    public boolean canChildScrollUp()
    {
        if (mChildView == null)
        {
            return false;
        }
        if (Build.VERSION.SDK_INT < 14)
        {
            if (mChildView instanceof AbsListView)
            {
                final AbsListView absListView = (AbsListView) mChildView;
                return absListView.getChildCount() > 0
                        && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                        .getTop() < absListView.getPaddingTop());
            }
            else
            {
                return ViewCompat.canScrollVertically(mChildView, -1) || mChildView.getScrollY() > 0;
            }
        }
        else
        {
            return ViewCompat.canScrollVertically(mChildView, -1);
        }
    }
```
最后，在`onTouchEvent()`中，根据偏移量来改变相关控件的显示和隐藏。
**MotionEvent.ACTION_MOVE：**
先取得偏移量distanceY，值得注意的是，在上拉的时候distanceY会出现小于0的值，这里需要处理下，小于0的时候默认取0，否则整个mChildView会被上拉超出屏幕，不是想要的效果。
```Java
                mCurrentY = event.getY();
                float controlX = event.getX();
                float distanceY = mCurrentY - mTouchY;
                distanceY = Math.max(0, distanceY);
```
之后就需要在distanceY在某一个偏移区间变化的时候，动态改变header中各种view的状态，并添加动画效果。
为方便理解，可对照下图：
![images](https://github.com/gaoyuyu/CustomRefreshLayoutDemo/raw/master/captures/refresh_detail.png)
1、distanceY在（0，mHeadHeight），无涟漪效果。
```Java
    if (distanceY < mHeadHeight)
    {
        Log.e(LOG_TAG, "distanceY < mHeadHeight");
        mWaveView.setHeadHeight((int) distanceY);
        mWaveView.setControlY(0);
        mWaveView.setControlX((int) controlX);
        mWaveView.invalidate();
        mCurrentHeaderHeight = (int) distanceY;
        offsetY = mCurrentHeaderHeight;
        mArrow.setVisibility(View.GONE);
        mTip.setVisibility(View.GONE);
        mProgressBar.setVisibility(View.GONE);
    
    }
```
2、distanceY在（mHeadHeight，mHeadHeight + mControlY]。
```Java
    if (distanceY > mHeadHeight && (distanceY <= (mHeadHeight + mControlY)))
    {
        Log.e(LOG_TAG, "distanceY > mHeadHeight && (distanceY < (mHeadHeight + mControlY/2))");
        float currentWaveHeight = distanceY - mHeadHeight;
        mWaveView.setHeadHeight((int) mHeadHeight);
        mWaveView.setControlY((int) currentWaveHeight);
        mWaveView.setControlX((int) controlX);
        mWaveView.invalidate();
        mCurrentHeaderHeight = (int) mHeadHeight;
        offsetY = mHeadHeight + currentWaveHeight / 2;
        mProgressBar.setVisibility(View.GONE);
        if (currentWaveHeight / mControlY > 0.5f)
        {
            mArrow.animate().setListener(new AnimatorListenerAdapter()
            {
                @Override
                public void onAnimationStart(Animator animation)
                {
                    super.onAnimationStart(animation);
                    mTip.setText("下拉刷新");
                }
    
                @Override
                public void onAnimationEnd(Animator animation)
                {
                    super.onAnimationEnd(animation);
                    //动画结束后，显示控件，否则出现不和谐的过渡效果
                    mArrow.setVisibility(View.VISIBLE);
                    mTip.setVisibility(View.VISIBLE);
    
                }
            });
            mArrow.animate()
                    .rotationX(0)
                    .setDuration(15)
                    .start();
        }
    }
```
这里的distanceY选择的范围是(mHeadHeight,mHeadHeight + mControlY],动画效果过渡更加自然，若是(mHeadHeight,mHeadHeight + mControlY/2]，圆弧一下子撑开，过渡效果粗糙。

> 值得注意的是，为什么offsetY= mHeadHeight + currentWaveHeight / 2 ？
先介绍下贝塞尔曲线的基本原理
![images](http://ww4.sinaimg.cn/large/005Xtdi2jw1f361bjqj2vj308c0dwwem.jpg)
连接DE，取点F，使得: AD:AB = BE:BC = DF:DE
![images](http://ww2.sinaimg.cn/large/005Xtdi2jw1f361oje6h1j308c0dwdg0.jpg)
注：以上图片来源于[安卓自定义View进阶-Path之贝塞尔曲线](http://www.gcssloop.com/customview/Path_Bezier)
好了，假设一种最极限的情况，是我们的控制点的X坐标是屏幕宽度的一半，如下图
![images](https://github.com/gaoyuyu/CustomRefreshLayoutDemo/raw/master/captures/prove.png)
根据AD:AB = BE:BC = DF:DE，且三角形ABC是一个等边三角形，BC=AB，所以有AD = BE，根据三角形中位线定理和相似三角形，很容易知道BF是mControlY的1/2。

3、distanceY 大于mHeadHeight + currentWaveHeight / 2
```Java
    else if (distanceY > (mHeadHeight + mControlY))
    {
        Log.e(LOG_TAG, "distanceY > (mHeadHeight + mControlY / 2)");
        mWaveView.setHeadHeight((int) mHeadHeight);
        mWaveView.setControlY((int) mControlY);
        mWaveView.setControlX((int) controlX);
        mWaveView.invalidate();
        mCurrentHeaderHeight = (int) mHeadHeight;
        offsetY = mHeadHeight + mControlY / 2;
        mProgressBar.setVisibility(View.GONE);

        mArrow.animate().setListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationStart(Animator animation)
            {
                super.onAnimationStart(animation);
                mTip.setText("释放立即刷新");
                mArrow.setVisibility(View.VISIBLE);
                mTip.setVisibility(View.VISIBLE);
            }
        });
        mArrow.animate()
                .rotationX(180)
                .setDuration(50)
                .start();
    }
```
4、最后设置下mChildView的偏移，和重绘头布局
```Java
    //设置子View的Y轴偏移量
    mChildView.setTranslationY(offsetY);
    //重新设置header的高度
    mHeaderLayout.getLayoutParams().height = (int) offsetY;
    //重绘
    mHeaderLayout.requestLayout();
```
5、切记return true
**MotionEvent.ACTION_UP：**
处理手指离开时的逻辑相对简单，只需获取到的mChildView在Y轴上的偏移，大于一定的范围做出相应的处理即可。
```Java
    //当偏移量大于mHeadHeight + mWaveHeight / 2时，刷新
    if (mChildView.getTranslationY() >= (mHeadHeight + mControlY / 2))
    {
        Log.e(LOG_TAG, "MotionEvent.ACTION_UP mChildView.getTranslationY() >= (mHeadHeight + mControlY / 2)");
        mChildView.animate().setListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationStart(Animator animation)
            {
                super.onAnimationStart(animation);
                mArrow.setVisibility(View.GONE);
                mProgressBar.setVisibility(View.VISIBLE);
                mTip.setText("正在加载");
            }
        });
        mChildView.animate().translationY(mHeadHeight).start();

        isRefreshing = true;
        if (onMaterialRefreshListener != null)
        {
            onMaterialRefreshListener.onRefresh(MaterialRefreshLayout.this);
        }
    }
    else
    {
        mChildView.animate().setListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationStart(Animator animation)
            {
                super.onAnimationStart(animation);
                mArrow.setVisibility(View.GONE);
                mProgressBar.setVisibility(View.GONE);
                mTip.setVisibility(View.GONE);
            }
        });
        mChildView.animate().translationY(0).start();

    }
    return true;
```
**正在刷新状态**
当需要设置正在处于刷新状态时，layout初始化时mChildView为null， 通过打印日志是先执行setRefreshing在执行onAttachedToWindow，所以为null。
所以setRefreshing还需要放在onAttachedToWindow()方法里面。
```Java
    public void setRefreshing(boolean refreshing)
    {
        isRefreshing = refreshing;

        if (isRefreshing)
        {
            if (mChildView == null)
            {
                return;
            }
            mChildView.animate().translationY(mHeadHeight).start();
            mWaveView.setHeadHeight((int) mHeadHeight);
            mWaveView.setControlY(0);
            mWaveView.setControlX(1);
            mWaveView.invalidate();
            mProgressBar.setVisibility(View.VISIBLE);
            mTip.setText("正在加载");
            mTip.setVisibility(View.VISIBLE);

        }
    }
```
**刷新完成**
```Java
    public void finishRefresh()
    {
        if (mChildView != null)
        {
            mChildView.animate().translationY(0).start();
            setRefreshing(false);
        }
    }
```
####三、基本用法
基本用法也是和SwipeRefreshLayout一样
```Java
    //正在刷新
    rl.setRefreshing(true);
    
    //设置监听
    rl.setOnMaterialRefreshListener(new OnMaterialRefreshListener()
    {
        @Override
        public void onRefresh(MaterialRefreshLayout refreshLayout)
        {
            handler.sendEmptyMessageDelayed(0,2000);
        }
    });
        
    private Handler handler = new Handler()
    {
        @Override
        public void handleMessage(Message msg)
        {
            super.handleMessage(msg);
            //完成刷新
            rl.finishRefresh();
        }
    };
```
###源码
Github：https://github.com/gaoyuyu/CustomRefreshLayoutDemo


----------
相关推荐链接（内含贝塞尔曲线教程）：
自定义View教程：[安卓自定义View教程目录](http://www.gcssloop.com/customview/CustomViewIndex)


