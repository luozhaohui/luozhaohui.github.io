---
layout: post
title: "Android多线程分析之一：使用Thread异步下载图像"
date: 2014-07-08 20:38:35 +0800
comments: true
categories: [软件开发]
tags: [android, thread]
description: Android多线程分析之一：使用Thread异步下载图像
keywords: android, thread
---

打算整理一下对<code>Android Framework</code>中多线程相关知识的理解，主要集中在<code>Framework</code>层的<code>thread</code>, <code>Handler</code>, <code>Looper</code>, <code>MessageQueue</code>, <code>Message</code>, <code>AysncTask</code>，当然不可避免地要涉及到<code>native</code>方法，因此也会分析<code>dalvik</code>中和线程以及消息处理相关的代码：如<code>dalvik</code>中的<code>C++ Thread</code>类以及<code>MessageQueue</code>类。本文将从一个使用<code>Thread</code>的简单应用入手，引入<code>Thread</code>这个话题，接下来的几篇文章会依次介绍前面提到的那些主题。

<!--more-->

## 权限
这是一个使用<code>Android Thread</code>从网络上异步下载图片并在<code>ImageView</code>中显示的的简单示例。因为需要访问网络，所以要在<code>manifest.xml</code>中添加网络访问权限：

``` xml
	<uses-permission android:name="android.permission.INTERNET">
	</uses-permission>
```

## 布局
布局文件很简单，一个<code>Button，一个<code>ImageView：

``` xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="10dip" >

	<Button
		android:id="@+id/LoadButton"
		android:layout_width="fill_parent"
		android:layout_height="wrap_content"
		android:text="Load">
	</Button>

	<ImageView
		android:id="@+id/ImageVivew" 
		android:layout_width="match_parent" 
		android:layout_height="400dip" 
		android:scaleType="centerInside" 
		android:padding="2dp">
	</ImageView> 
	
</LinearLayout>
``` 

## 代码
接下来看代码。首先来看定义：图片的 <code>url</code> 路径，两个消息值以及一些控件：

``` java
    private static final String sImageUrl = "http://fashion.qqread.com/ArtImage/20110225/0083_13.jpg";

	private static final int MSG_LOAD_SUCCESS = 0;
	private static final int MSG_LOAD_FAILURE = 1;
	
    private Button mLoadButton;
    private ProgressDialog mProgressBar;
    private ImageView mImageView;
``` 

然后来看控件的设置：

``` java
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		
		Log.i("UI thread", " >> onCreate()");
		
		mProgressBar = new ProgressDialog(this);
		mProgressBar.setCancelable(true);
		mProgressBar.setMessage("Image downloading ...");
		mProgressBar.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
		mProgressBar.setMax(100);
		
		mImageView = (ImageView)this.findViewById(R.id.ImageVivew);
		
		mLoadButton = (Button)this.findViewById(R.id.LoadButton);
		mLoadButton.setOnClickListener(new View.OnClickListener() {
            @Override 
            public void onClick(View v) {
        		mProgressBar.setProgress(0);
        		mProgressBar.show();
        		
        		new Thread() {
        			@Override
        			public void run() {
        				Log.i("Load thread", " >> run()");
                        Bitmap bitmap = loadImageFromUrl(sImageUrl);
                        if (bitmap != null) {
                        	Message msg = mHandler.obtainMessage(MSG_LOAD_SUCCESS, bitmap);
                        	mHandler.sendMessage(msg);
                        }
                        else {
                        	Message msg = mHandler.obtainMessage(MSG_LOAD_FAILURE, null);
                        	mHandler.sendMessage(msg);
                        }
        			}
        		}.start();
            }
        });
	}
```

<code>loadImageFromUrl</code>是一个从网络下载<code>Bitmap</code>的<code>static</code>函数：

``` java
    static Bitmap loadImageFromUrl(String uil) {
    	Bitmap bitmap = null;
        try{
            InputStream in = new java.net.URL(sImageUrl).openStream();
            bitmap = BitmapFactory.decodeStream(in);
            in.close();
        }
        catch (Exception e) {
        	e.printStackTrace();
        }
        return bitmap;
    }
```

<code>mHandler</code>是主线程也就是<code>UI</code>线程处理消息的<code>Handler</code>：

``` java
    private Handler mHandler= new Handler(){
        @Override
        public void handleMessage(Message msg) {
        	Log.i("UI thread", " >> handleMessage()");
        	
            switch(msg.what){
            case MSG_LOAD_SUCCESS:
            	Bitmap bitmap = (Bitmap) msg.obj;
                mImageView.setImageBitmap(bitmap);
                
                mProgressBar.setProgress(100);
                mProgressBar.setMessage("Image downloading success!");
                mProgressBar.dismiss();
                break;
                
            case MSG_LOAD_FAILURE:
                mProgressBar.setMessage("Image downloading failure!");
                mProgressBar.dismiss();
            	break;
            }
        }
    };
```

## 综述
纵观上面的代码，当点击<code>load</code>按钮时，会创建一个匿名<code>Thread</code>，并调用其<code>start()</code>启动运行线程，在这个线程中进行图像下载并解码成<code>Bitmap</code>，然后通过<code>Handler</code>向<code>UI</code>线程发送消息以通知下载结果。这都是在匿名<code>Thead</code>中处理的。主线程也就是<code>UI</code>线程收到消息之后，会分发给<code>Handler</code>，在它的<code>handleMessage</code>方法中根据消息<code>id</code>来处理下载结果，要么成功要么失败，并相应地更新<code>UI</code>。

运行该示例：

可以从<code>logcat</code>的第四栏看到<code>UI thread(tid: 817)</code>和<code>Load thread(tid: 830)</code>的线程<code>id</code>是不同的，因为它们是两个独立的线程。

![Android多线程分析之一：使用Thread异步下载图像](/images/posts/20140709-android-thread-tutoria01_1.png)

在匿名线程下载完毕之后，为什么不直接在这个线程的<code>run()</code>中更新<code>UI</code>呢？这样做有什么后果？这些问题将在后文详细解答。
