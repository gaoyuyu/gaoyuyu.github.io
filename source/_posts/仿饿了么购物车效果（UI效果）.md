---
title: 仿饿了么购物车效果（UI效果）
date: 2017-08-06 13:47:15
tags:
---

仿饿了么购物车效果（UI效果）
<!--more-->

 ![images](https://github.com/gaoyuyu/StickyListDemo/raw/master/captures/cart.gif)
 
> * 右列表标题悬停
> * 左右列表滑动时联动
> * 添加购物车时动画效果


## 右列表标题悬停&左右列表滑动时联动

github开源库：[StickyListHeadersListView](https://github.com/emilsjolander/StickyListHeaders)
StickyListHeadersListView使用：http://blog.csdn.net/zuiwuyuan/article/details/49872533

gradle添加依赖
```Java
    compile 'se.emilsjolander:stickylistheaders:2.7.0'
```
主Layout简单，基本LinearLayout实现，左边列表直接用RecyclerView实现
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.gaoyy.stickylistdemo.MainActivity"
    >

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="300dp"
        android:layout_above="@+id/linearLayout"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_alignParentTop="true"
        android:orientation="horizontal">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/left"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="0.3"/>

        <se.emilsjolander.stickylistheaders.StickyListHeadersListView
            android:id="@+id/list"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="0.7"
            />
    </LinearLayout>

    <LinearLayout
        android:id="@+id/linearLayout"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:layout_alignParentBottom="true"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:background="#433d3d"
        android:orientation="horizontal">
        <ImageView
            android:id="@+id/cart"
            android:layout_width="50dp"
            android:layout_height="50dp"
            android:padding="10dp"
            android:layout_marginLeft="10dp"
            android:src="@mipmap/icon_cart"/>

    </LinearLayout>
</RelativeLayout>
```
数据模拟，左Type右Sample，唯一需要注意的是，**在初始化数据时，数据必须按照分组顺排列**
```Java
public class Sample
{
    //item id
    private String id;
    //所属组id
    private String groupId;
    //item title
    private String title;
    //item desc
    private String desc;
    //组 title
    private String groupTitle;
    //数量
    private int count;

    public Sample(String id, String groupId, String title, String desc, String groupTitle,int count)
    {
        this.id = id;
        this.groupId = groupId;
        this.title = title;
        this.desc = desc;
        this.groupTitle = groupTitle;
        this.count= count;
    }
    
    //setter and getter...
}

public class Type
{
    private int status;
    private String type;

    public Type(int status, String type)
    {
        this.status = status;
        this.type = type;
    }
    
    //setter and getter...
}

...

private void initData()
{
    int random = (int) (Math.random() * 100 + 1);
    for (int i = 0; i < 15; i++)
    {
        headerData.add(new Type(0, "种类" + i));
    }

    for (int i = 0; i < headerData.size(); i++)
    {
        for (int k = 0; k < 10; k++)
        {
            data.add(new Sample("" + k, "" + i, "种类" + i + "   分类" + k, (int) (Math.random() * 100 + 1) + "", "种类" + i, 0));
        }
    }
}
```
StickyListHeadersListView的适配器需实现StickyListHeadersAdapter接口的`getHeaderView() getHeaderId()`方法，其中`getHeaderView()`与普通ListView的`getView()`实现一致，`getHeaderId()`则需要返回数据的组id
```Java
public class SampleAdapter extends BaseAdapter implements StickyListHeadersAdapter
{
    ...
    @Override
    public View getHeaderView(int position, View convertView, ViewGroup parent)
    {
        HeaderViewHolder headerViewHolder;
        if (convertView == null)
        {
            headerViewHolder = new HeaderViewHolder();
            convertView = inflater.inflate(R.layout.header, parent, false);
            headerViewHolder.header = (TextView) convertView.findViewById(R.id.header);
            convertView.setTag(headerViewHolder);
        }
        else
        {
            headerViewHolder = (HeaderViewHolder) convertView.getTag();
        }
        headerViewHolder.header.setText(data.get(position).getGroupTitle());
        return convertView;
    }
    
    @Override
    public long getHeaderId(int position)
    {
        //return the first character of the country as ID because this is what headers are based upon
        return Long.parseLong(data.get(position).getGroupId());
    }
}
```
实现滑动时左右联动(左-ReListAdapter，右-SampleAdapter)
1.左边RecyclerView点击item时可调用StickyListHeadersListView的setSelection(position)来实现右边列表的显示。**注意，这里的position传入的是item的position，不是header的组id**。左列表item状态变化在adapter中实现，由Type中status控制（1-选中，0-未选中）；
2.右边列表滑动时，左边列表item的选中状态也相应变化。右列表滑动时，设置setOnScrollListener来监听item位置变化，更新左列表item选中状态；

ReListAdapter
```Java
if(data.get(position).getStatus() == 1)
{
    leftViewHolder.layout.setBackgroundColor(context.getResources().getColor(android.R.color.white));
    leftViewHolder.tv.setTextColor(context.getResources().getColor(android.R.color.black));
}
if(data.get(position).getStatus() == 0)
{
    leftViewHolder.layout.setBackgroundColor(context.getResources().getColor(R.color.gray));
    leftViewHolder.tv.setTextColor(context.getResources().getColor(R.color.gray_text));
}

...
//更新数据，刷新列表
public void updateData(List<Type> data)
{
    this.data = data;
    notifyDataSetChanged();
}
```
MainActivity
```Java
    reListAdapter.setOnItemClickListener(new ReListAdapter.OnItemClickListener()
    {
        @Override
        public void onItemClick(View view, int position)
        {
            int select = -1;
            for (int i = 0; i < data.size(); i++)
            {
                if (Integer.valueOf(data.get(i).getGroupId()) == position)
                {
                    select = i;
                    break;
                }
            }
            updateTypeList(position);
            list.setSelection(select);

        }
    });
    
    list.setOnScrollListener(new AbsListView.OnScrollListener()
    {
        @Override
        public void onScrollStateChanged(AbsListView view, int scrollState)
        {

        }

        @Override
        public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount)
        {
            Sample sample = data.get(firstVisibleItem);
            int groupId = Integer.valueOf(sample.getGroupId());
            left.smoothScrollToPosition(groupId);
            updateTypeList(groupId);
        }
    });
    
    private void updateTypeList(int position)
    {
        for (int i = 0; i < headerData.size(); i++)
        {
            Type type = headerData.get(i);
    
            if (position == i)
            {
                type.setStatus(1);
                headerData.remove(i);
                headerData.add(i, type);
            }
            else
            {
                type.setStatus(0);
                headerData.remove(i);
                headerData.add(i, type);
            }
        }
        reListAdapter.updateData(headerData);
    }
```

> 这里有个BUG，当每组的数据量比较少时，即一屏显示完所有的数据或者余下的数据时，左边列表的选中状态会默认选中右边列表出现在顶部的item的组，下面的组选不到。原因在于右边列表的OnScrollListener，在onScroll方法中取到的是位于顶部item的position，暂时没有想到解决办法。

 ![images](https://github.com/gaoyuyu/StickyListDemo/raw/master/captures/bug.png)
 
 左右联动基本实现(>.<)
 
## 添加购物车动画效果实现

首先，添加add的点击监听，这里用接口的方式实现，在Activity中setListener，在BasicOnClickListener的onClick中传入itemViewHolder.parent，为了方便后续在Activity中添加动画效果。

item布局
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/item_parent"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    >

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentTop="true"
        android:layout_marginLeft="12dp"
        android:layout_marginStart="12dp"
        android:layout_marginTop="5dp"
        android:layout_toEndOf="@+id/imageView"
        android:layout_toRightOf="@+id/imageView"
        android:text="TextView"
        android:textSize="20sp"/>

    <TextView
        android:id="@+id/desc"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignBottom="@+id/imageView"
        android:layout_alignLeft="@+id/title"
        android:layout_alignStart="@+id/title"
        android:layout_marginBottom="5dp"
        android:text="TextView"
        android:textColor="@color/colorAccent"
        android:textSize="18sp"/>

    <ImageView
        android:id="@+id/imageView"
        android:layout_width="80dp"
        android:layout_height="80dp"
        android:layout_alignParentLeft="true"
        android:layout_alignParentStart="true"
        android:layout_alignParentTop="true"
        android:scaleType="fitXY"
        app:srcCompat="@mipmap/ic_launcher"/>

    <ImageView
        android:id="@+id/add"
        android:layout_width="25dp"
        android:layout_height="25dp"
        android:layout_alignBottom="@+id/desc"
        android:layout_alignParentEnd="true"
        android:layout_alignParentRight="true"
        android:layout_marginEnd="24dp"
        android:layout_marginRight="24dp"
        app:srcCompat="@mipmap/button_add"/>

    <TextView
        android:id="@+id/count"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignBaseline="@+id/desc"
        android:layout_alignBottom="@+id/desc"
        android:layout_marginEnd="12dp"
        android:layout_marginRight="12dp"
        android:layout_toLeftOf="@+id/add"
        android:layout_toStartOf="@+id/add"
        android:text="1"
        android:textSize="20sp"
        android:visibility="visible"/>

    <ImageView
        android:id="@+id/minus"
        android:layout_width="25dp"
        android:layout_height="25dp"
        android:layout_alignBottom="@+id/count"
        android:layout_marginRight="12dp"
        android:layout_toLeftOf="@+id/count"
        android:layout_toStartOf="@+id/count"
        android:visibility="visible"
        app:srcCompat="@mipmap/button_minus"/>


</RelativeLayout>
```

SampleAdapter
```Java
    private OnOperationClickListener onOperationClickListener;

    public interface OnOperationClickListener
    {
        void onOperationClick(View parent,View view, int position);
    }

    public void setOnOperationClickListener(OnOperationClickListener listener)
    {
        this.onOperationClickListener = listener;
    }
    
    @Override
    public View getView(int position, View convertView, ViewGroup parent)
    {
        //...
        
        if(onOperationClickListener != null)
        {
            itemViewHolder.add.setOnClickListener(new BasicOnClickListener(itemViewHolder,position));
            itemViewHolder.miuns.setOnClickListener(new BasicOnClickListener(itemViewHolder,position));
        }
    }
    public class BasicOnClickListener implements View.OnClickListener
    {
        ItemViewHolder itemViewHolder;
        int position;

        public BasicOnClickListener(ItemViewHolder itemViewHolder,int position)
        {
            this.itemViewHolder = itemViewHolder;
            this.position = position;
        }

        @Override
        public void onClick(View view)
        {
            switch (view.getId())
            {
                case R.id.add:
                    onOperationClickListener.onOperationClick(itemViewHolder.parent,itemViewHolder.add,position);
                    break;
                case R.id.minus:
                    onOperationClickListener.onOperationClick(itemViewHolder.parent,itemViewHolder.miuns,position);
                    break;
            }
        }
    }
```
MainActivity
1.点击add时count加1，更新adapter，miuns显现，点击minus时count减1，当减到0时miuns隐藏，通过设置miuns的tag来实现。
```Java
SampleAdapter
getView()
{
        if(data.get(position).getCount() == 0)
        {
            itemViewHolder.miuns.setAlpha(0f);
            itemViewHolder.count.setAlpha(0f);
            itemViewHolder.miuns.setTag(true);
        }
        else
        {
            itemViewHolder.count.setAlpha(1f);
            itemViewHolder.miuns.setAlpha(1f);
            itemViewHolder.count.setText(data.get(position).getCount()+"");
            itemViewHolder.miuns.setTag(false);
        }
}
```
2.添加动画
> * 获取根布局，item中的add，购物车cart的坐标
> * 添加一个add的ImageView到根布局中，位置与item中的add相同
> * 为新增的add添加抛物线效果，设置transitionX和transitionY的属性动画，结束点的位于cart的位置，X轴上线性，Y轴上加速，同时设置渐隐动画。（这里的抛物线效果也可以用贝塞尔曲线实现）
> * 新增add在动画结束时从根布局remove掉
> * 新增add在动画结束时购物车cart设置一个放大的属性动画


```Java
        sampleAdapter.setOnOperationClickListener(new SampleAdapter.OnOperationClickListener()
        {
            @Override
            public void onOperationClick(View parent, View view, int position)
            {
                ImageView add = (ImageView) parent.findViewById(R.id.add);
                ImageView miuns = (ImageView) parent.findViewById(minus);

                Sample sample = data.get(position);
                int count = sample.getCount();
                switch (view.getId())
                {
                    case R.id.add:
                        //添加时动画效果
                        createAnim(add);
                        
                        count = count + 1;
                        data.get(position).setCount(count);

                        boolean tag = ((boolean) miuns.getTag());
                        if (tag)
                        {
                            minusAnim(miuns);
                        }
                        else
                        {
                            sampleAdapter.updateDataCount(data);
                        }
                        break;
                    case R.id.minus:
                        count = count - 1;
                        if (count < 0)
                        {
                            count = 0;
                        }
                        data.get(position).setCount(count);
                        sampleAdapter.updateDataCount(data);
                        break;
                }
            }
        });
        
        
    /**
     * 添加时动画效果
     * @param add
     */
    private void createAnim(ImageView add)
    {
        //获取+号的坐标
        int[] addPoint = new int[2];
        add.getLocationInWindow(addPoint);

        //获取购物车的坐标
        int[] cartPoint = new int[2];
        cart.getLocationInWindow(cartPoint);

        //获父布局的坐标
        int[] parentPoint = new int[2];
        activityMain.getLocationInWindow(parentPoint);

        final ImageView newAdd = new ImageView(MainActivity.this);
        newAdd.setLayoutParams(new RelativeLayout.LayoutParams(dip2px(MainActivity.this, 25), dip2px(MainActivity.this, 25)));
        newAdd.setImageResource(R.mipmap.button_add);
        newAdd.setX(addPoint[0]);
        newAdd.setY(addPoint[1] - parentPoint[1]);
        activityMain.addView(newAdd);

        //X轴平移动画
        ValueAnimator x = ValueAnimator.ofInt((int) newAdd.getX(), cartPoint[0]);
        x.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                int value = (int) valueAnimator.getAnimatedValue();
                newAdd.setTranslationX(value);
            }
        });
        //设置线性插值器
        x.setInterpolator(new LinearInterpolator());

        //Y轴平移动画
        ValueAnimator y = ValueAnimator.ofInt((int) newAdd.getY(), cartPoint[1] - parentPoint[1]);
        y.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                int value = (int) valueAnimator.getAnimatedValue();
                newAdd.setTranslationY(value);
            }
        });
        //设置加速插值器
        y.setInterpolator(new AccelerateInterpolator());


        //新增ImageView加号的渐隐动画
        ObjectAnimator newAddAlpha = ObjectAnimator.ofFloat(newAdd, "Alpha", 1.0f, 0.0f);
        newAddAlpha.addListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationEnd(Animator animation)
            {
                super.onAnimationEnd(animation);
                //动画结束移除view
                activityMain.removeView(newAdd);
            }
        });

        //动画集合
        AnimatorSet set = new AnimatorSet();
        set.playTogether(x, y, newAddAlpha);
        set.start();

        set.addListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationEnd(Animator animation)
            {
                super.onAnimationEnd(animation);
                //购物车的放大动画
                cartAnim();
            }
        });
    }
    
    /**
     * 减号平移动画
     *
     * @param minus
     */
    public void minusAnim(final View minus)
    {
        ObjectAnimator translationXAnim = ObjectAnimator.ofFloat(minus, "TranslationX", 100, 0);
        ObjectAnimator alphaAnim = ObjectAnimator.ofFloat(minus, "Alpha", 0f, 1f);
        AnimatorSet set = new AnimatorSet();
        set.playTogether(translationXAnim, alphaAnim);
        set.addListener(new AnimatorListenerAdapter()
        {
            @Override
            public void onAnimationEnd(Animator animation)
            {
                super.onAnimationEnd(animation);
                sampleAdapter.updateDataCount(data);
            }
        });
        set.start();
    }
    
    /**
     * 购物车放大动画
     */
    private void cartAnim()
    {
        ObjectAnimator cartScale = ObjectAnimator.ofFloat(cart, "ScaleXY", 0.6f, 1.0f);
        cartScale.addUpdateListener(new ValueAnimator.AnimatorUpdateListener()
        {
            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator)
            {
                float value = (float) valueAnimator.getAnimatedValue();
                cart.setScaleX(value);
                cart.setScaleY(value);
            }
        });
        cartScale.start();
    }
```
这里需要注意的是，坐标计算。
![坐标图](http://img.blog.csdn.net/20170806134426637?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

动画效果基本实现(>.<)


源码：
github地址：[StickyListDemo](https://github.com/gaoyuyu/StickyListDemo)
