---
layout: post
title: "Android多线程分析之五：使用AsyncTask异步下载图像"
date: 2014-07-19 21:12:40 +0800
comments: true
categories: [软件开发]
tags: [android, thread]
description: Android多线程分析之五：使用AsyncTask异步下载图像
keywords: android, thread
---

## 引言
在本系列文章的第一篇[Android多线程分析之一：使用Thread异步下载图像](https://luozhaohui.github.io/blog/2014/07/08/android-thread-tutoria01/)中，曾演示了如何使用 <code>Thread</code> 来完成异步任务。<code>Android</code> 为了简化在 <code>UI</code> 线程中完成异步任务（毕竟 <code>UI</code> 线程是 <code>app</code> 最重要的线程），实现了一个名为 <code>AysncTask</code> 的模板类。使用 <code>AysncTask</code> 能够在异步任务进行的同时，将任务进度状态反馈给 <code>UI</code> 线程（如让 <code>UI</code> 线程更新进度条）。正是由于它与 <code>UI</code> 线程紧密相关，使用的时候要就有一些限制，<code>AysncTask</code> 必须在 <code>UI</code> 线程中创建，并在 <code>UI</code> 线程中启动（通过调用其 <code>execute()</code> 方法）；此外，<code>AysncTask</code> 设计的目的是用于一些耗时较短的任务，如果是耗时较长的任务不推荐使用 <code>AysncTask</code>。

可以用简化记忆 “三参数，四步骤” 来学习 <code>AysncTask</code>。 即带有三个模板参数 <code>Params</code>, <code>Progress</code>, <code>Result</code>，四个处理步骤：<code>onPreExecute</code>，<code>doInBackground</code>，<code>onProgressUpdate</code>，<code>onPostExecute</code>。

<!--more-->

## 简介

## 三参数

> <code>Params</code> 是异步任务所需的参数类型，也即 <code>doInBackground(Params... params)</code> 方法的参数类型；  
> <code>Progress</code> 是指进度的参数类型，也即 <code>onProgressUpdate(Progress... values)</code> 方法的参数类型；  
> <code>Result</code> 是指任务完成返回的参数类型，也即 <code>onPostExecute(Result result)</code> 或 <code>onCancelled(Result result)</code> 方法的参数类型。  

如果某一个参数类型没有意义或没有被用到，传递 <code>void</code> 即可。

### 四步骤

> <code>protected void onPreExecute()</code>：在 <code>UI</code> 线程中运行，在异步任务开始之前被执行，以便 <code>UI</code> 线程完成一些初始化动作，如将进度条清零；  
> <code>protected abstract Result doInBackground(Params... params)</code>：在后台线程中运行，这是完成异步任务的地方，它是抽象接口，子类必须提供实现；  
> <code>protected void onProgressUpdate(Progress... values)</code>：在 <code>UI</code> 线程中运行，在异步任务执行的过程中可以通过调用 <code>void publishProgress(Progress... values)</code> 方法通知 <code>UI</code> 线程在 <code>onProgressUpdate</code> 方法内更新进度状态；  
> <code>protected void onPostExecute(Result result)</code>：在 <code>UI</code> 线程中运行，当异步任务完成之后被执行，以便 <code>UI</code> 线程更新任务完成状态。  

<code>AysncTask</code> 支持取消异步任务，当异步任务被取消之后，上面的步骤四就不会被执行了，取而代之将执行 <code>onCancelled(Result result)</code>，以便 <code>UI</code> 线程更新任务被取消之后的状态。谨记：上面提到的这些方法都是回调函数，不需要用户手动去调用。

以前的 <code>AysncTask</code> 是基于单一后台线程实现的，而从 <code>Android 3.0</code> 起 <code>AysncTask</code> 是基于 <code>Android</code> 的并发库（<code>java.util.concurrent</code>）实现的，本文中不会展开讨论其具体实现，只是演示如何使用 <code>AysncTask。

## 使用示例

有了前面的轮廓介绍，再来使用 <code>AysncTask</code> 是非常容易的，下面的例子与[Android多线程分析之一：使用Thread异步下载图像](https://luozhaohui.github.io/blog/2014/07/08/android-thread-tutoria01/)中的例子非常相似，只不过是使用 <code>AysncTask</code> 来完成异步任务罢了。

## 权限
这是一个使用 <code>AysncTask</code> 从网络上异步下载图片并在 <code>ImageView</code> 中显示的的简单示例。因为需要访问网络，所以要在 <code>manifest.xml</code> 中添加网络访问权限：
``` xml
	<uses-permission android:name="android.permission.INTERNET">
	</uses-permission>
```

### 布局
布局文件很简单，一个 <code>Button</code>，一个 <code>ImageView</code>：

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

### 代码：

首先来看定义：图片的 <code>url</code> 路径，两个消息值以及一些控件：

``` java
    private static final String sImageUrl = "http://fashion.qqread.com/ArtImage/20110225/0083_13.jpg";
    private Button mLoadButton;
    private ImageView mImageView;
```

然后来看控件的设置：

``` java
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		
		Log.i("UI thread", " >> onCreate()");
		
		mImageView = (ImageView)this.findViewById(R.id.ImageVivew);
		
		mLoadButton = (Button)this.findViewById(R.id.LoadButton);
		mLoadButton.setOnClickListener(new View.OnClickListener() {
            @Override 
            public void onClick(View v) {
            	LoadImageTask task = new LoadImageTask(v.getContext());
                task.execute(sImageUrl);
            }
        });
	}
```

<code>LoadImageTask</code> 继承自 <code>AysncTask</code>，由这个类去完成异步图片下载任务，并相应地更新 <code>UI</code> 状态。

``` java
	class LoadImageTask extends AsyncTask<String, Integer, Bitmap> 
	{
		private ProgressDialog mProgressBar;
	    
		LoadImageTask(Context context)
		{
			mProgressBar = new ProgressDialog(context);
			mProgressBar.setCancelable(true);
			mProgressBar.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
			mProgressBar.setMax(100);
		}
		
		@Override
		protected Bitmap doInBackground(String... params) {
			Log.i("Load thread", " >> doInBackground()");
			
			Bitmap bitmap = null;
			
			try{
				publishProgress(10);
				Thread.sleep(1000);
				
	            InputStream in = new java.net.URL(sImageUrl).openStream();
	            publishProgress(60);
				Thread.sleep(1000);
				
	            bitmap = BitmapFactory.decodeStream(in);
	            in.close();
            } catch (Exception e) {
            	e.printStackTrace();
			}

			publishProgress(100);
			return bitmap;
		}
		
		@Override
		protected void onCancelled() {
            super.onCancelled();
        }
		 
	    @Override
        protected void onPreExecute() {
		
			mProgressBar.setProgress(0);
	    	mProgressBar.setMessage("Image downloading ... %0");
	    	mProgressBar.show();
	    	
	    	Log.i("UI thread", " >> onPreExecute()");
        }
	     
	    @Override
        protected void onPostExecute(Bitmap result) {
	    	Log.i("UI thread", " >> onPostExecute()");
	    	if (result != null) {
		    	mProgressBar.setMessage("Image downloading success!");
	        	mImageView.setImageBitmap(result);
	    	}
	    	else {
		    	mProgressBar.setMessage("Image downloading failure!");
	    	}

            mProgressBar.dismiss(); 
        }
        
	   @Override
		protected void onProgressUpdate(Integer... values) {
		   Log.i("UI thread", " >> onProgressUpdate() %" + values[0]);
		   mProgressBar.setMessage("Image downloading ... %" + values[0]);
		   mProgressBar.setProgress(values[0]);
		}
	};
```

在 <code>LoadImageTask 中，前面提到的四个步骤都涉及到了：

首先在任务开始之前在 <code>onPreExecute()</code> 方法中设置进度条的初始状态（<code>UI</code>线程）；然后在下载线程中执行 <code>doInBackground()</code> 以完成下载任务，并在其中调用 <code>publishProgress()</code> 来通知 <code>UI</code> 线程更新进度状态；<code>UI</code> 线程在 <code>onProgressUpdate()</code> 中得知进度，并更新进度条（<code>UI线程</code>）；最后下载任务完成，<code>UI</code> 线程在 <code>onPostExecute()</code>中得知下载好的图像，并更新<code>UI</code>显示该图像（<code>UI</code>线程）。