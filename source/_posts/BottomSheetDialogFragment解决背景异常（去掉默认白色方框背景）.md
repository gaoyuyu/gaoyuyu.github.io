---
title: BottomSheetDialogFragment 解决背景异常（去掉默认白色方框背景）
date: 2018-05-23 21:29:21
tags:
---

BottomSheetDialogFragment的使用很简单
```Java
public class CustomBottomSheetDialogFragment extends BottomSheetDialogFragment implements View.OnClickListener
{
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull final LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState)
    {
        View rootView = inflater.inflate(R.layout.spec_bottom_sheet, container, false);
        return rootView;
    }
}
```
<!--more-->
![这里写图片描述](https://img-blog.csdn.net/20180523212116898?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0dhb195dXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

但是默认的布局后面有个白色的背景，不符合需求，需要去掉外层默认的白色背景

设置Dialog主题即可

```Java
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        //设置背景透明，才能显示出layout中诸如圆角的布局，否则会有白色底（框）
        setStyle(BottomSheetDialogFragment.STYLE_NORMAL, R.style.CustomBottomSheetDialogTheme);
    }

```
```xml
    <style name="CustomBottomSheetDialogTheme" parent="Theme.Design.Light.BottomSheetDialog">
        <item name="bottomSheetStyle">@style/CustomBottomSheetStyle</item>
    </style>

    <style name="CustomBottomSheetStyle" parent="Widget.Design.BottomSheet.Modal">
        <item name="android:background">@android:color/transparent</item>
    </style>
```
![这里写图片描述](https://img-blog.csdn.net/2018052321284411?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0dhb195dXl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
