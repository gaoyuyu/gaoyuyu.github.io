---
title: Android自定义View之超简单圆形数字指示器
date: 2016-09-30 18:58:28
tags:
---

MIUI8内置的天气App的一种效果 <!--more-->
![images](http://img.blog.csdn.net/20160930181105054)

不过好像没有看到有其他的什么动画效果，也有可能是在4.7屏下这个Layout在其他Layout的下面，加载完数据有效果看不到？
不纠结这个，心血来潮重新自定义写个就是了。



效果图：
![images](https://github.com/gaoyuyu/CircleIndexView/raw/master/captures/appgif.gif)

自定义View的流程不再这里过多叙述，直接上代码。
```Java
    //（灰）外圈画笔
    private Paint outPaint = new Paint();
    //（绿）内圈画笔
    private Paint inPaint = new Paint();
    //文字画笔
    private Paint middleTextPaint = new Paint();
    //外切矩形
    private RectF mRectF;

    //圆初始弧度
    private float startAngle = 135;
    //圆结束弧度
    private float sweepAngle = 270;

    int mCenter = 0;
    int mRadius = 0;

    private int indexValue;
    private String middleText = "N/A";
    private String bottomText = "";
    private int circleWidth;
    private int circleHeight;
    private float inSweepAngle;

    private float numberTextSize;
    private float middleTextSize;
    private float bottomTextSize;

    private int outCircleColor;
    private int inCircleColor;
    private int numberColor;
    private int middleTextColor;
    private int bottomTextColor;
```
为了兼容Android LOLLIPOP+，需要重写View的构造函数，在构造函数中初始化画笔和自定义属性。
```Java
    public CircleIndexView(Context context)
    {
        this(context, null);
    }

    public CircleIndexView(Context context, AttributeSet attrs)
    {
        this(context, attrs, -1);
    }

    public CircleIndexView(Context context, AttributeSet attrs, int defStyleAttr)
    {
        super(context, attrs, defStyleAttr);
        initParams(context, attrs);
        initPaint();
    }


    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public CircleIndexView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)
    {
        super(context, attrs, defStyleAttr, defStyleRes);
        initParams(context, attrs);
        initPaint();
    }
```
```Java
    /**
     * 初始化View参数
     *
     * @param context
     * @param attrs
     */
    private void initParams(Context context, AttributeSet attrs)
    {
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.circleindexview);
        middleText = ta.getString(R.styleable.circleindexview_middleText);
        bottomText = ta.getString(R.styleable.circleindexview_bottomText);
        numberTextSize = ta.getDimension(R.styleable.circleindexview_numberTextSize,
                TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 20, context.getResources().getDisplayMetrics()));
        middleTextSize = ta.getDimension(R.styleable.circleindexview_middleTextSize,
                TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 15, context.getResources().getDisplayMetrics()));
        bottomTextSize = ta.getDimension(R.styleable.circleindexview_bottomTextSize,
                TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 15, context.getResources().getDisplayMetrics()));
        outCircleColor = ta.getColor(R.styleable.circleindexview_outCircleColor, Color.LTGRAY);
        inCircleColor = ta.getColor(R.styleable.circleindexview_inCircleColor, Color.GREEN);
        numberColor = ta.getColor(R.styleable.circleindexview_numberColor, Color.GREEN);
        middleTextColor = ta.getColor(R.styleable.circleindexview_middleTextColor, Color.LTGRAY);
        bottomTextColor = ta.getColor(R.styleable.circleindexview_bottomTextColor, Color.LTGRAY);
        ta.recycle();
    }
```
在初始化参数的时候，对于sp/dp的值需要通过TypeValue进行转换。
```Java
ta.getDimension(R.styleable.circleindexview_bottomTextSize,TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_SP, 15, context.getResources().getDisplayMetrics()));
```
>**TypedValue.COMPLEX_UNIT_SP**对应的是**sp**，**TypedValue.COMPLEX_UNIT_DIP**对应的是**dp**
当然，除了getDimension()之外，还有getDimensionPixelOffset()，getDimensionPixelSize()，各自都有区别，感兴趣可以查查Api。

测量View，在这里，为了让整个View比较好看，刻意设置了layout_width和layout_height无论是match_parent或者wrap_content，都是200dp的默认值。（measureWidth()和measureHeight()代码基本相同）
```Java
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
    {
        setMeasuredDimension(measureWidth(widthMeasureSpec), measureHeight(heightMeasureSpec));
    }

    private int measureWidth(int widthMeasureSpec)
    {
        int result = (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, 200, getResources().getDisplayMetrics());
        int specMode = MeasureSpec.getMode(widthMeasureSpec);
        int specSize = MeasureSpec.getSize(widthMeasureSpec);
        switch (specMode)
        {
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
                break;
            case MeasureSpec.AT_MOST:
                break;
        }
        setCircleWidth(result);
        return result;
    }
```
最后，是将整个view绘制出来。代码有注释，就不多赘述。
```Java

    @Override
    protected void onDraw(Canvas canvas)
    {
        super.onDraw(canvas);

        mCenter = getCircleWidth() / 2;
        mRadius = getCircleWidth() / 2 - 50;

        mRectF = new RectF(mCenter - mRadius, mCenter - mRadius, mCenter
                + mRadius, mCenter + mRadius);

        //外圆圈
        canvas.drawArc(mRectF, startAngle, sweepAngle, false, outPaint);
        //内圆圈
        canvas.drawArc(mRectF, startAngle, getInSweepAngle(), false, inPaint);

        //中心数字
        middleTextPaint.setColor(getNumberColor());
        middleTextPaint.setTextSize(getNumberTextSize());
        canvas.drawText(getIndexValue() + "", getCircleWidth() / 2, getCircleHeight() / 2, middleTextPaint);

        //中心文字(etc. Pm25)
        middleTextPaint.setColor(getMiddleTextColor());
        middleTextPaint.setTextSize(getMiddleTextSize());
        canvas.drawText(getMiddleText(), getCircleWidth() / 2, getCircleHeight() / 2 + 40, middleTextPaint);

        //底部文字(etc. 空气污染指数)
        middleTextPaint.setColor(getBottomTextColor());
        middleTextPaint.setTextSize(getBottomTextSize());
        canvas.drawText(getBottomText(), getCircleWidth() / 2, getCircleHeight() - 50, middleTextPaint);

    }
```
暴露公共方法给用户更新数据，同时实现动画效果，这里用到了属性动画。
```Java
    public void updateIndex(int value)
    {
        float inSweepAngle = sweepAngle * value / 100;
        ValueAnimator angleAnim = ValueAnimator.ofFloat(0f, inSweepAngle);
        ValueAnimator valueAnim = ValueAnimator.ofInt(0, value);
        angleAnim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                float currentValue = (float) valueAnimator.getAnimatedValue();
                setInSweepAngle(currentValue);
                invalidate();
            }
        });
        valueAnim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                int currentValue = (int) valueAnimator.getAnimatedValue();
                setIndexValue(currentValue);
                invalidate();
            }
        });
        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.setInterpolator(new DecelerateInterpolator());
        animatorSet.setStartDelay(150);
        animatorSet.playTogether(angleAnim, valueAnim);
        animatorSet.start();
    }
```

项目开源，已上传到jcenter，具体使用见[gaoyuyu/CircleIndexView](https://github.com/gaoyuyu/CircleIndexView)。


欢迎各位star，fork，在使用过程中有任何BUG，欢迎大家留言，共同学习共同进步~!


----------


相关推荐链接：
自定义View教程：[安卓自定义View教程目录](http://www.gcssloop.com/customview/CustomViewIndex)


