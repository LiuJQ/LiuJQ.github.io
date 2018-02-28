---
layout: post
title: Android自定义组合View
tags: [Android, Custom View]
---

## 背景
&emsp;&emsp;最近在开发一款电商类B端APP，比较忙，现在最新版本要上线了。刚好年底才有时间停下来梳理一下自己写过的东西，顺便分享一些出来，抛砖引玉，还请大家不吝赐教。

## 设计图UI效果
![UI效果图](/assets/img/android_screenshots/20150211-custom-views_1.png)

&emsp;&emsp;我们这里要介绍的就是后面三块的UI效果实现。可以看到这几个块都是类似的，但是如果要一个一个用布局写出来，不仅工作量大，而且还会造成XML文件代码冗余、文件臃肿。显然，我们可以通过自定义View来实现一个块的效果，然后在需要的地方include进来就好了，代码也简洁好看。

## 实现思路
&emsp;&emsp;最外层布局采用LinearLayout，图片部分和底部文字部分就可以使用android:layout_weight来控制显示比例。文字部分也可以使用LinearLayout来布局，设置android:gravity="center"就好，里面的TextView就不赘述了。

## 动手实现
### 继承LinearLayout实现外部轮廓
&emsp;&emsp;先写好XML布局文件，调整好要实现的效果。下面是XML代码：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/recommended_item_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/item_bound"
    android:orientation="vertical"
    android:clickable="true" >

    <com.personal.views.RemoteImageView
        android:id="@+id/recommended_item_front_cover"
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="2.5"
        android:scaleType="centerCrop"
        android:src="@drawable/default_loading" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1"
        android:gravity="center"
        android:orientation="vertical" >

        <TextView
            android:id="@+id/recommended_item_title"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:singleLine="true"
            android:ellipsize="end"
            android:gravity="center"
            android:textColor="@color/v123_light_gray"
            android:textSize="15dip" />

        <LinearLayout
            android:id="@+id/recommended_item_price_layout"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:paddingBottom="3dip"
            android:orientation="horizontal" >

            <TextView
                android:id="@+id/recommended_item_newprice"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textColor="@color/red"
                android:textSize="15dp" />

            <TextView
                android:id="@+id/recommended_item_oldprice"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="15dip"
                android:textColor="@color/v123_light_gray"
                android:textSize="12dp" />
        </LinearLayout>
    </LinearLayout>

</LinearLayout>
```
 
&emsp;&emsp;新建Java代码文件RecommendedItem.java继承LinearLayout，把上面的布局文件使用LayoutInflater加载进来:

``` java
public class RecommendedItem extends LinearLayout {
	public static final String TAG = RecommendedItem .class.getSimpleName();
	
	private Context mContext;
	private boolean isDataSet = false;
	private float titleTextSize, newPriceTextSize, oldPriceTextSize;
	
	private RemoteImageView mItemCover;
	private TextView mItemTitle, mItemNewPrice, mItemOldPrice;
	private LinearLayout mPriceLayout, mRootLayout;
	
	public RecommendedItem (Context context) {
		super(context);
		init(context, null);
	}
	
	public RecommendedItem (Context context, AttributeSet attrs) {
		super(context, attrs);
		init(context, attrs);
	}

	@SuppressLint("NewApi")
	public RecommendedItem (Context context, AttributeSet attrs,
			int defStyle) {
		super(context, attrs, defStyle);
		init(context, attrs);
	}

	private void init(Context context, AttributeSet attrs) {
		mContext = context;	  LayoutInflater.from(context).inflate(R.layout.v2_recommended_item, this, true);
	}
}
```

### 定义各种需要的属性
&emsp;&emsp;在values文件夹下新建attrs.xml文件，添加我们需要的View属性。
```xml
<declare-styleable name="RecommendedItem">
        <attr name="title_textsize" format="dimension"/>
        <attr name="newPrice_textsize" format="dimension"/>
        <attr name="oldPrice_textsize" format="dimension"/>
</declare-styleable>
```
&emsp;&emsp;然后我们就可以在XML文件中引用这些属性了，在RecommendedItem类的init方法中获取我们在XML文件中写的这些属性的值。
```java
TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.RecommendedItem);   
this.titleTextSize = array.getDimension(R.styleable.RecommendedItem_title_textsize, 15);   
this.newPriceTextSize = array.getDimension(R.styleable.RecommendedItem_newPrice_textsize, 15);
this.oldPriceTextSize = array.getDimension(R.styleable.RecommendedItem_oldPrice_textsize, 12);
array.recycle(); // 一定要调用，否则这次的设定会对下次的使用造成影响
		
Log.d(TAG, "this.titleTextSize=" + this.titleTextSize);
Log.d(TAG, "this.newPriceTextSize=" + this.newPriceTextSize);
Log.d(TAG, "this.oldPriceTextSize=" + this.oldPriceTextSize);
```
### 初始化View
```java
@Override
protected void onFinishInflate() {
	super.onFinishInflate();
	this.mItemCover = (RemoteImageView) this.findViewById(R.id.recommended_item_front_cover);
	this.mItemTitle = (TextView) this.findViewById(R.id.recommended_item_title);
	this.mItemNewPrice = (TextView) this.findViewById(R.id.recommended_item_newprice);
	this.mItemOldPrice = (TextView) this.findViewById(R.id.recommended_item_oldprice);
	this.mPriceLayout = (LinearLayout) this.findViewById(R.id.recommended_item_price_layout);
	this.mRootLayout = (LinearLayout) this.findViewById(R.id.recommended_item_root);
		
	this.mItemTitle.setTextSize(this.titleTextSize);
	this.mItemNewPrice.setTextSize(this.newPriceTextSize);
	this.mItemOldPrice.setTextSize(this.oldPriceTextSize);
}
```

### 添加一些必要的接口方法
&emsp;&emsp;自定义的View给外部使用提供接口，动态修改它的属性，实现我们想要的显示效果。
```java
// 设置点击事件监听器
public void setOnClickListener(OnClickListener listener) {
	if(listener != null) {
		this.mRootLayout.setOnClickListener(listener);
	}
}

public void setItemCover(int resId) {
	this.mItemCover.setImageResource(resId);
}

public void setItemCover(String url) {
	this.mItemCover.setImageUrl(url);
}

public void setItemCover(Bitmap bitmap, boolean need2Recycle) {
	this.mItemCover.setImageBitmap(bitmap);
	if(need2Recycle) {
		if(bitmap != null) {
			bitmap = null;
			System.gc();
		}
	}
}

public void loadItemCover(String remoteUrl) {
	UILManager.displayImage(remoteUrl, this.mItemCover, UILManager.optionsPicsPreview);
}

public void setItemTitle(String title) {
	this.mItemTitle.setText(title);
}

public String getItemTitle() {
	if(!TextUtils.isEmpty(this.mItemTitle.getText())) {
		return this.mItemTitle.getText().toString();
	}
	return null;
}

public void setItemNewPrice(String newPrice) {
	this.mItemNewPrice.setText(String.format(mContext.getString(R.string.recommended_item_price_format), newPrice));
}

public void setItemOldPrice(String oldPrice) {
	this.mItemOldPrice.setText(String.format(mContext.getString(R.string.recommended_item_price_format), oldPrice));
	this.mItemOldPrice.getPaint().setFlags(Paint.STRIKE_THRU_TEXT_FLAG);
}

public void setPriceLayoutVisible(boolean visible) {
	if(!visible) {
		this.mPriceLayout.setVisibility(View.GONE);
	} else {
		this.mPriceLayout.setVisibility(View.VISIBLE);
	}
}

public boolean isDataSet() {
	return isDataSet;
}

public void setDataSet(boolean isDataSet) {
	this.isDataSet = isDataSet;
	if(isDataSet && this.mPriceLayout.getVisibility() == View.GONE) 
		this.mPriceLayout.setVisibility(View.VISIBLE);
}

public void clearView() {
	this.mItemCover.setImageResource(R.drawable.default_loading);
	this.mItemTitle.setText(null);
	this.mPriceLayout.setVisibility(View.GONE);
}
```

### 自定义View完成
&emsp;&emsp;至此，我们根据UI设计图实现的自定义View就完成，可以方便地在其他任何布局文件中使用了。

## 自定义View的引用
```xml
<com.personal.views.RecommendedItem
    android:id="@+id/recommended_spike_1"
    android:layout_width="0dip"
    android:layout_height="match_parent"
    android:layout_marginLeft="10dip"
    android:layout_weight="1" />
```
## 实现效果图
![实现效果图](/assets/img/android_screenshots/20150211-custom-views_2.png)
