---
title: 自定义Ripple效果的Colorfulbar
date: 2017-03-04 23:19:59
tags:
---

近日在github上看到的一种效果。正好最近也在进阶学习自定义ViewGroup，便开始着手进行对这种效果小小实现了一把。
原效果：出自于 [open-android/Android1](https://github.com/open-android/Android1) 

<!--more-->

> [open-android/Android1](https://github.com/open-android/Android1)是[open-android/Android](https://github.com/open-android/Android) 下的一个子链接。

 ![](http://upload-images.jianshu.io/upload_images/4037105-4825eae343c2a062.gif?imageMogr2/auto-orient/strip)
 
 原创效果：效果实现有一些出入，不要在意这些细节。
 ![images](https://github.com/gaoyuyu/LearningCustomView/raw/master/captures/colorfulbar.gif)

 
 思考如何去做：
 1. 图标和文字是用自定义View还是自定义ViewGroup实现比较好
 2. 图标和文字的动画
 3. Ripple效果的实现
 回答：
 1. 图标和文字用自定义ViewGroup实现比较好，中间摆放一个ImageView和TextView。为什么？为了验证到底是自定义View好还是自定义ViewGroup好，我花了2-3晚上去码代码去尝试，得出的结论是自定义ViewGroup显得更加简洁一点，对view的动画操作比较简单。
 2. 动画自然是View的属性动画
 3. 最外层是重写的LinearLayout，根据点击的坐标重绘，实现ripple效果。


----------


 
#### **AnimItem : 图标icon+文字Text的ViewGroup** 
```Java
public class AnimItemV extends ViewGroup implements View.OnClickListener
```
成员变量。其中，status为1时显示的只有icon，为2时显示icon和文字。初始值默认为1，显示icon。
```Java
    private ImageView icon;
    private TextView text;

    private int x;
    private int y;

    //1-icon,2-icon+text
    private int status = 1;

    private static final String TAG = AnimItemV.class.getSimpleName();

    public AnimItemV(Context context)
    {
        this(context, null);
    }

    public AnimItemV(Context context, AttributeSet attrs)
    {
        this(context, attrs, -1);
    }

    public AnimItemV(Context context, AttributeSet attrs, int defStyleAttr)
    {
        super(context, attrs, defStyleAttr);
        setOnClickListener(this);
    }

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public AnimItemV(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)
    {
        super(context, attrs, defStyleAttr, defStyleRes);
        setOnClickListener(this);
    }
```
onMeassure
```Java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)
    {
        setTag(1);
        int childCount = getChildCount();
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int sizeWidth = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int sizeHeight = MeasureSpec.getSize(heightMeasureSpec);


        /**
         * wrap_content下的width，height
         */

        int width = 0;
        int height = 0;


        for (int i = 0; i < childCount; i++)
        {
            View childView = getChildAt(i);
            measureChild(childView, widthMeasureSpec, heightMeasureSpec);

            MarginLayoutParams childMargin = (MarginLayoutParams) childView.getLayoutParams();

            //子view的宽
            int childWidth = childView.getMeasuredWidth() + childMargin.leftMargin + childMargin.rightMargin;
            //子view的高
            int childHeight = childView.getMeasuredHeight() + childMargin.topMargin + childMargin.bottomMargin;


            width = Math.max(width, childWidth);
            height += childHeight;


        }


        width = Math.max(width, height);
        height = Math.max(width, height);


        //sizeWidth = width, sizeHeight = height是为了应对weight的情况
        sizeWidth = width;
        sizeHeight = height;

        setMeasuredDimension((widthMode == MeasureSpec.EXACTLY) ? sizeWidth : width + getPaddingLeft() + getPaddingRight(),
                (heightMode == MeasureSpec.EXACTLY) ? sizeHeight : height + getPaddingTop() + getPaddingBottom());


    }
```
onLayout，坐标的计算并不复杂，画个草稿就可以想明白。
```Java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b)
    {
        int width = getWidth();
        int height = getHeight();


        icon = (ImageView) getChildAt(0);
        MarginLayoutParams iconMargin = (MarginLayoutParams) icon.getLayoutParams();
        int iconWidth = icon.getMeasuredWidth() + iconMargin.leftMargin + iconMargin.rightMargin;
        int iconHeight = icon.getMeasuredHeight() + iconMargin.topMargin + iconMargin.bottomMargin;


        text = (TextView) getChildAt(1);
        text.setAlpha(0.0f);
        MarginLayoutParams textMargin = (MarginLayoutParams) text.getLayoutParams();
        int textWidth = text.getMeasuredWidth() + textMargin.leftMargin + textMargin.rightMargin;
        int textHeight = text.getMeasuredHeight() + textMargin.topMargin + textMargin.bottomMargin;


        int lc = (width - iconWidth) / 2;
        int tc = (width - iconWidth) / 2;
        int rc = lc + iconWidth;
        int bc = tc + iconHeight;

        icon.layout(lc, tc, rc, bc);


        int lc1 = (width - textWidth) / 2;
        int tc1 = bc;
        int rc1 = lc1 + textWidth;
        int bc1 = tc1 + textHeight;


        /**
         * -tc字体全部显示
         */
//            text.layout(lc1,tc1-tc,rc1,bc1-tc);
        text.layout(lc1, tc1, rc1, bc1);
    }
```
最后是ImageView和TextView的属性动画，提供Public方法供外部调用。在写复原动画的时候要特别注意Y轴的数值变化，见注释。
```Java
    /**
     * icon向上移动，text向上渐显动画
     */
    public void startAnim()
    {
        MarginLayoutParams iconMargin = (MarginLayoutParams) icon.getLayoutParams();
        int iconWidth = icon.getMeasuredWidth() + iconMargin.leftMargin + iconMargin.rightMargin;
        int tc = (getWidth() - iconWidth) / 2;

        //icon Y轴向上移动tc
        ObjectAnimator iconYAnim = ObjectAnimator.ofFloat(icon, "TranslationY", -tc).setDuration(500);

        //text透明度从0变到1
        ObjectAnimator textAlphaAnim = ObjectAnimator.ofFloat(text, "Alpha", text.getAlpha(), 1.0f).setDuration(500);
        //text Y轴向上移动tc
        ObjectAnimator textYAnim = ObjectAnimator.ofFloat(text, "TranslationY", -tc).setDuration(500);


        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.playTogether(iconYAnim, textAlphaAnim, textYAnim);
        animatorSet.start();

        setTag(2);


    }

    /**
     * 复原动画
     */
    public void reverseAnim()
    {
        MarginLayoutParams iconMargin = (MarginLayoutParams) icon.getLayoutParams();
        int iconWidth = icon.getMeasuredWidth() + iconMargin.leftMargin + iconMargin.rightMargin;
        int tc = (getWidth() - iconWidth) / 2;

        /**
         *注意这里是0，因为是-tc+tc，所以是0
         */
        //icon Y轴向下移动0
        ObjectAnimator iconYAnim = ObjectAnimator.ofFloat(icon, "TranslationY", 0).setDuration(500);

        //text透明度从1变到0
        ObjectAnimator textAlphaAnim = ObjectAnimator.ofFloat(text, "Alpha", text.getAlpha(), 0.0f).setDuration(500);
        //text Y轴向下移动0
        ObjectAnimator textYAnim = ObjectAnimator.ofFloat(text, "TranslationY", 0).setDuration(500);


        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.playTogether(iconYAnim, textAlphaAnim, textYAnim);
        animatorSet.start();

        setTag(1);

    }
```
最后，加上onckick事件，通过tag的获取到的status来判断应该执行哪个动画效果。
```Java
    @Override
    public void onClick(View view)
    {
        if ((int) getTag() == 1)
        {
            startAnim();
        }
        else if ((int) getTag() == 2)
        {
            reverseAnim();
        }
    }
```
#### **AnimLinearLayout** 
```Java
public class AnimLinearLayout extends LinearLayout
```
```Java
    //存放一组颜色
    private int[] rippleColors;
    //子ViewGroup的X轴坐标
    private int childX;
    //子ViewGroup的Y轴坐标
    private int childY;
    //ripple的半径
    private float radius;
    //ripple的颜色
    private int rippleColor;

    //....getter and setter....

    public AnimLinearLayout(Context context, AttributeSet attrs)
    {
        this(context, attrs, -1);
    }

    public AnimLinearLayout(Context context, AttributeSet attrs, int defStyleAttr)
    {
        super(context, attrs, defStyleAttr);
    }

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    public AnimLinearLayout(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)
    {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
```

在onAttachedToWindow中，遍历子view并为每一个子view设置OnTouchListener
```Java
    @Override
    protected void onAttachedToWindow()
    {
        super.onAttachedToWindow();


        int childCount = getChildCount();

        for (int i = 0; i < childCount; i++)
        {

            final AnimItemV animItemV = (AnimItemV) getChildAt(i);
            animItemV.setOnTouchListener(new OnTouchListener()
            {
                @Override
                public boolean onTouch(View view, MotionEvent motionEvent)
                {
                    for (int i = 0; i < getChildCount(); i++)
                    {
                        if (animItemV.getId() == getChildAt(i).getId())
                        {
                            continue;
                        }
                        ((AnimItemV) getChildAt(i)).reverseAnim();
                    }
                    return false;
                }
            });
        }

    }
```
在onInterceptTouchEvent中，手指按下时获取x，y坐标，手指松开时执行相应的动画效果。
```Java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event)
    {
        Log.e(TAG, TAG + " onInterceptTouchEvent");
        int action = event.getAction();
        switch (action)
        {
            case MotionEvent.ACTION_DOWN:
                int childCount = getChildCount();
                int x = (int) event.getX();
                int y = (int) event.getY();

                Log.e(TAG, "x-->" + x);
                for (int i = 0; i < childCount; i++)
                {
                    AnimItemV animItemV = (AnimItemV) getChildAt(i);
                    if(x>=animItemV.getTop()&&x<=animItemV.getRight())
                    {
                        setRippleColor(getResources().getColor(rippleColors[i]));
                        break;
                    }
                }


                setChildX(x);
                setChildY(y);


                break;

            case MotionEvent.ACTION_UP:
                int radius = Math.max(getWidth(), getHeight());
                ValueAnimator animator = ValueAnimator.ofInt(0, radius);
                animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
                {
                    @Override
                    public void onAnimationUpdate(ValueAnimator valueAnimator)
                    {
                        int currentRadius = (int) valueAnimator.getAnimatedValue();
                        setRadius((float) (currentRadius));
                        invalidate();
                    }
                });

                animator.setInterpolator(new AccelerateDecelerateInterpolator());
                animator.start();

                animator.addListener(new AnimatorListenerAdapter()
                {
                    @Override
                    public void onAnimationEnd(Animator animation)
                    {
                        super.onAnimationEnd(animation);
                        setBackgroundColor(rippleColor);
                    }
                });
                break;

        }


        return super.onInterceptTouchEvent(event);
    }
```

layout.xml 由于是继承自LinearLayout，所以LinearLayout的一些属性可以直接用。
```xml
<com.gaoyy.learningcustomview.view.AnimLinearLayout
        android:id="@+id/activity_colorful_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="#888888"
        android:gravity="center"
        android:orientation="horizontal"
        tools:context="com.gaoyy.learningcustomview.ui.ColorfulBarActivity"
        android:layout_alignParentBottom="true">


        <com.gaoyy.learningcustomview.view.AnimItemV
            android:id="@+id/item1"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"
            >


            <ImageView
                android:layout_width="50dp"
                android:layout_height="50dp"
                android:src="@mipmap/ic_moment"/>

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Moment"
                android:textColor="#ffffff"
                android:textSize="16sp"/>


        </com.gaoyy.learningcustomview.view.AnimItemV>

        <com.gaoyy.learningcustomview.view.AnimItemV
            android:id="@+id/item2"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            >


            <ImageView
                android:layout_width="50dp"
                android:layout_height="50dp"
                android:src="@mipmap/ic_friend"/>

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Friend"
                android:textColor="#ffffff"
                android:textSize="16sp"/>


        </com.gaoyy.learningcustomview.view.AnimItemV>

        ...


    </com.gaoyy.learningcustomview.view.AnimLinearLayout>
```

项目地址：https://github.com/gaoyuyu/LearningCustomView

 
 