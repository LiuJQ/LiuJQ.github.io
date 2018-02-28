---
layout: post
title: 基于HTML5的ichartjs图表组件在Android中的应用
tags: [Android, hybrid, ichartjs, HTML5, WebApp]
feature-img: "assets/img/pexels/computer.jpeg"
thumbnail: "assets/img/pexels/computer.jpeg"
---

### 一、什么是ichartjs？

&emsp;&emsp;[ichartjs](http://www.ichartjs.com/) 是一款基于HTML5的图形库。使用纯javascript语言, 利用HTML5的canvas标签绘制各式图形。 ichartjs致力于为您的应用提供简单、直观、可交互的体验级图表组件。是WEB/APP图表展示方面的解决方案 。如果你正在开发HTML5的应用，ichartjs正好适合您。 ichartjs目前支持饼图、环形图、折线图、面积图、柱形图、条形图。ichartjs是基于Apache License 2.0协议的开源项目。

### 二、如何使用ichartjs？
在你的html页面中，添加如下javascript引用即可：
```html
<script type="text/javascript" src="ichart.1.2.min.js"></script>
```

### 三、ichartjs在Android中的应用
&emsp;&emsp;了解过WebApp开发的童鞋应该都知道，Android平台应用开发的另外一种方式，使用WebView加载HTML页面，在HTML页面中实现我们的功能和界面开发，那么Android代码和HTML代码交互就通过JavaScript来实现。ichartjs的使用也是这一套路线。
下面以一个ichartjs的3D柱状图的demo来讲解它在Android中的应用。首先定义好需要的实体：
```java
package com.example.achartdemo.beans;

public class Contact {
	private String name; // 浏览器的名称
	private double value; // 浏览器对应的所占市场份额值
	private String color; // 在柱形图中所显示的颜色
	
	/**
	 * 构造函数
	 * @param name 浏览器的名称
	 * @param value 浏览器对应的所占市场份额值
	 * @param color 在柱形图中所显示的颜色
	 */
	public Contact(String name, double value, String color) {
		this.name = name;
		this.value = value;
		this.color = color;
	}
	
	// 下面是三个实例变量的getters and setters
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public double getValue() {
		return value;
	}
	public void setValue(double value) {
		this.value = value;
	}
	public String getColor() {
		return color;
	}
	public void setColor(String color) {
		this.color = color;
	}
	
}
```

然后模拟http请求加载数据的过程：
```java
package com.example.achartdemo.http;
import java.util.ArrayList;
import java.util.List;

import com.example.achartdemo.beans.Contact;

public class ContactService {

	public List<Contact> getContacts() {
		List<Contact> contacts = new ArrayList<Contact>();
		contacts.add(new Contact("IE", 32.85, "#a5c2d5"));
		contacts.add(new Contact("Chrome", 33.59, "#cbab4f"));
		contacts.add(new Contact("Firefox", 22.85, "#76a871"));
		contacts.add(new Contact("Safari", 7.39, "#9f7961"));
		contacts.add(new Contact("Opera", 1.63, "#a56f8f"));
		contacts.add(new Contact("Other", 1.69, "#6f83a5"));
		return contacts;
	}
}
```

定义界面展示的XML布局文件：
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.example.achartdemo.MainActivity$PlaceholderFragment" >
    
    <WebView 
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</RelativeLayout>
```

在Actitivy或者Fragment中加载布局并作View的初始化操作：
```java
package com.example.achartdemo;

import android.annotation.SuppressLint;
import android.content.Context;
import android.os.Bundle;
import android.os.Handler;
import android.support.v4.app.Fragment;
import android.support.v7.app.ActionBarActivity;
import android.view.LayoutInflater;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.view.ViewGroup;
import android.webkit.WebSettings;
import android.webkit.WebView;

public class MainActivity extends ActionBarActivity {
	
	private static Context mContext;
	private static Handler mHandler = new Handler();

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		mContext = this;

		if (savedInstanceState == null) {
			getSupportFragmentManager().beginTransaction()
					.add(R.id.container, new PlaceholderFragment()).commit();
		}
	}

	@Override
	public boolean onCreateOptionsMenu(Menu menu) {

		// Inflate the menu; this adds items to the action bar if it is present.
		getMenuInflater().inflate(R.menu.main, menu);
		return true;
	}

	@Override
	public boolean onOptionsItemSelected(MenuItem item) {
		// Handle action bar item clicks here. The action bar will
		// automatically handle clicks on the Home/Up button, so long
		// as you specify a parent activity in AndroidManifest.xml.
		int id = item.getItemId();
		if (id == R.id.action_settings) {
			return true;
		}
		return super.onOptionsItemSelected(item);
	}

	/**
	 * A placeholder fragment containing a simple view.
	 */
	public static class PlaceholderFragment extends Fragment {
		private WebView mWebView;

		public PlaceholderFragment() {
		}

		@Override
		public View onCreateView(LayoutInflater inflater, ViewGroup container,
				Bundle savedInstanceState) {
			View rootView = inflater.inflate(R.layout.fragment_main, container,
					false);
			initView(rootView);
			return rootView;
		}
		
		@SuppressLint({ "SetJavaScriptEnabled", "JavascriptInterface" })
		private void initView(View rootView) {
			mWebView = (WebView) rootView.findViewById(R.id.webview);
			mWebView.setHorizontalScrollBarEnabled(true);
			mWebView.setScrollBarStyle(View.SCROLLBARS_INSIDE_INSET);
			WebSettings settings = mWebView.getSettings();
			settings.setDefaultTextEncodingName("UTF-8");
			settings.setJavaScriptEnabled(true);
			settings.setSupportZoom(true);
			settings.setBuiltInZoomControls(true);
			mWebView.addJavascriptInterface(new JSInterface(mContext, mHandler, mWebView), JSInterface.TAG);
			String url = "file:///android_asset/index.html";
			mWebView.loadUrl(url);
		}
	}

}
```

关键的代码是我们需要在WebView中展现的HTML页面，代码如下：
```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="UTF-8" />
		<title>Hello World</title>
		<meta name="viewport" content="width=device-width, user-scalable=no" target-densitydpi="device-dpi"/>
		<meta name="Description" content="" />
		<meta name="Keywords" content="" />
		<script type="text/javascript" src="ichart.1.2.min.js"></script>
		<script type="text/javascript">
			var data = new Array();
			var contact = window.JSInterface.getContacts(); //得到JSInterface中转换出的json字符串
			eval('data='+contact); //得到json数据

			$(function(){	
				new iChart.Column3D({
					render : 'canvasDiv', //渲染的Dom目标,canvasDiv为Dom的ID
					data: data, //绑定数据
					title : 'Top 5 Browsers in August 2012', //设置标题
					showpercent:true, //显示百分比
					decimalsnum:2, 
					width : 600, //设置宽度，默认单位为px
					height : 300, //设置高度，默认单位为px
					align:'left',
					offsetx:50,
					animation : true,//开启过渡动画
					duration_animation_duration:800,//800ms完成动画 
					legend : {
						enable : true
					},
					coordinate:{ //配置自定义坐标轴
						scale:[{ //配置自定义值轴
							 width:600,
							 position:'left', //配置左值轴	
							 start_scale:0, //设置开始刻度为0
							 end_scale:40, //设置结束刻度为40
							 scale_space:8, //设置刻度间距为8
							 listeners:{ //配置事件
								parseText:function(t,x,y){ //设置解析值轴文本
									return {text:t+"%"}
								}
							}
						}]
					}
				}).draw(); //调用绘图方法开始绘图
			});
		</script>
	</head>

	<body>
		<div id='canvasDiv'></div>
	</body>
</html>
```

刚才在Activity中引用的JSInterface类，是我们用来跟HTML页面进行数据交互的JS接口类，代码如下：
```java
package com.example.achartdemo;

import java.util.List;
import java.util.Random;

import org.json.JSONArray;
import org.json.JSONException;
import org.json.JSONObject;

import android.annotation.SuppressLint;
import android.content.Context;
import android.os.Handler;
import android.util.Log;
import android.webkit.WebView;
import android.widget.Toast;

import com.example.achartdemo.beans.Contact;
import com.example.achartdemo.http.ContactService;

public class JSInterface {
	public static final String TAG = JSInterface.class.getSimpleName();
	private Context		mContext	= null;
	private Handler		mHandler	= null;
	private WebView		mView;

	private JSONArray	jsonArray	= new JSONArray();
	private Random		random		= new Random();

	public JSInterface(Context context, Handler handler, WebView webView) {
		mContext = context;
		mHandler = handler;
		mView = webView;
	}

	public void init() {
		// 通过handler来确保init方法的执行在主线程中
		mHandler.post(new Runnable() {

			public void run() {
				// 调用网页setContactInfo方法
				mView.loadUrl("javascript:setContactInfo('" + getJsonStr() + "')");
			}
		});
	}

	public int getW() {
		return px2dip(mContext.getResources().getDisplayMetrics().widthPixels);
	}

	public int getH() {
		return px2dip(mContext.getResources().getDisplayMetrics().heightPixels);
	}

	public int px2dip(float pxValue) {
		final float scale = mContext.getResources().getDisplayMetrics().density;
		return (int) (pxValue / scale + 0.5f);
	}

	public void setValue(String name, String value) {
		Toast.makeText(mContext, name+" "+value+"%", Toast.LENGTH_SHORT).show();
	}

	@SuppressLint("DefaultLocale")
	public String getRandColorCode() {
		String r, g, b;
		Random random = new Random();
		r = Integer.toHexString(random.nextInt(256)).toUpperCase();
		g = Integer.toHexString(random.nextInt(256)).toUpperCase();
		b = Integer.toHexString(random.nextInt(256)).toUpperCase();

		r = r.length() == 1 ? "0" + r : r;
		g = g.length() == 1 ? "0" + g : g;
		b = b.length() == 1 ? "0" + b : b;

		return "#" + r + g + b;
	}

	public String getJsonStr() {
		try {

			for (int i = 0; i < 10; i++) {
				JSONObject object1 = new JSONObject();
				object1.put("name", "name" + i);
				object1.put("value", random.nextInt(30));
				object1.put("color", getRandColorCode());
				jsonArray.put(object1);
			}
			Log.i("", jsonArray.toString());
			return jsonArray.toString();
		} catch (JSONException e) {
			e.printStackTrace();
		}
		return null;
	}
	
	/** 
     * 实现将list转换成json格式字符串 
     * @return json格式的字符串 
     */  
    public String getContacts() {  
    	ContactService contactService = new ContactService();
        List<Contact> contacts = contactService.getContacts();  
        String json = null;  
        try {  
            JSONArray array = new JSONArray();  
            for (Contact contact : contacts) {  
  
                JSONObject item = new JSONObject();  
                item.put("name", contact.getName());  
                item.put("value", contact.getValue());  
                item.put("color", contact.getColor());  
                array.put(item);  
            }  
            json = array.toString();  
            Log.i(TAG, json);  
        } catch (JSONException e) {  
            e.printStackTrace();  
        }  
        return json;  
    } 
}
```

&emsp;&emsp;上述的代码片段即可组合成一个工程，编译运行了。当然，我们需要把ichartjs的JS库和写好的HTML页面引入到我们工程目录下的assets文件夹下。另外，有一个需要注意的地方，我们在定义3D柱状图的时候，配置了柱状图的展示动画，需要在AndroidMenifest.xml文件中配置应用可以开启硬件加速：
```xml
<application
    android:hardwareAccelerated="true"
    android:allowBackup="true"
    android:icon="@drawable/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme" >
    <activity
        android:name="com.example.achartdemo.MainActivity"
        android:label="@string/app_name" >
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

下面附上运行的效果图：
![Demo](/assets/img/android_screenshots/20140423.png)