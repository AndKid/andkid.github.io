---
title: Android Camera
date: 2015-10-18 14:11:32
tags:
- android
- camera
---
支持Camera Preview直接显示在设备的View是SurfaceView
```java
	mCamera.setPreviewDisplay(SurfaceHolder holder);
	mCamera.startPreview();
```
间接实现，先把从Camera获得的图片流载入SurfaceTexture
```java
	mCamera.setPreviewTexture(SurfaceTexture);
	mCamera.setPreviewCallback(PreviewCallback cb);
	//mCamera.setPreviewCallbackWithBuffer(PreviewCallback cb);
	//mCamera.addCallbackBuffer(byte[] callbackBuffer);//preview frame将缓存在callbackBuffer并重用
	mCamera.startPreview();
	```
在PreviewCallback中重写public void onPreviewFrame(byte[] data, Camera camera)方法  
data为图片流，经过处理后可显示在SurfaceView上  
另外，data为拆分像素后的字节数组，猜测其index按像素一排排有序排列，比如一排有4个像素，一个像素2个字节，则index为15 26 37 48
