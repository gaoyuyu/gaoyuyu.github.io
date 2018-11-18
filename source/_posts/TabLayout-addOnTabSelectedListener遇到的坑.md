---
title: TabLayout addOnTabSelectedListener遇到的坑
date: 2018-04-26 11:29:44
tags:
---

在给TabLayout.addOnTabSelectedListener的时候，因为业务逻辑的关系（FUCK！），在select的时候有数据源的传参,没错，就是那个data
<!--more-->
```Java
    @Override
    public void onTabSelected(TabLayout.Tab tab)
    {
        LogUtil.e(Constant.LOG_TAG, "onTabSelected data--->" + data.toString());
        showSubCateFragment(data, tab.getPosition(), defaultSecondPosition);
    }

    @Override
    public void onTabUnselected(TabLayout.Tab tab)
    {
    }

    @Override
    public void onTabReselected(TabLayout.Tab tab)
    {
        LogUtil.e(Constant.LOG_TAG, "onTabSelected data--->" + data.toString());
        showSubCateFragment(data, tab.getPosition(), defaultSecondPosition);
    }
```
一开始写的时候，就是TabLayout.addOnTabSelectedListener(new xxxx(){.......})这样的写法，后来发现新增Tab后，一点击新增的Tab，直接就数组越界了，后来发现在onTabSelected()中还是旧的数据源，所以100%会越界
```Java

//前面的一段代码是获取数据源

shopTabLayout.addOnTabSelectedListener(new TabLayout.OnTabSelectedListener()
{
    @Override
    public void onTabSelected(TabLayout.Tab tab)
    {
        showSubCateFragment(data, tab.getPosition());
    }

    @Override
    public void onTabUnselected(TabLayout.Tab tab)
    {

    }

    @Override
    public void onTabReselected(TabLayout.Tab tab)
    {
        showSubCateFragment(data, tab.getPosition());
    }
});

```
所以，我的解决方法是
1.监听器设置成全局
2.新建一个类，implement TabLayout.OnTabSelectedListener，传入我的参数
3.在更新的时候先removeOnTabSelectedListener


```Java

//Step1
private OnTabSelectedListener listener = null;

//Step2
listener = new OnTabSelectedListener(data, defaultSecondPosition);

shopTabLayout.addOnTabSelectedListener(listener);


 public class OnTabSelectedListener implements TabLayout.OnTabSelectedListener
 {
     private List<Map<String, List<ChildBean>>> data;
     private int defaultSecondPosition;

     public OnTabSelectedListener(List<Map<String, List<ChildBean>>> data, int defaultSecondPosition)
     {
         this.data = data;
         this.defaultSecondPosition = defaultSecondPosition;
     }

     @Override
     public void onTabSelected(TabLayout.Tab tab)
     {
         LogUtil.e(Constant.LOG_TAG, "onTabSelected data--->" + data.toString());
         showSubCateFragment(data, tab.getPosition(), defaultSecondPosition);
     }

     @Override
     public void onTabUnselected(TabLayout.Tab tab)
     {
     }

     @Override
     public void onTabReselected(TabLayout.Tab tab)
     {
         LogUtil.e(Constant.LOG_TAG, "onTabSelected data--->" + data.toString());
         showSubCateFragment(data, tab.getPosition(), defaultSecondPosition);
     }
 }

//Step3
shopTabLayout.removeOnTabSelectedListener(listener);
```
