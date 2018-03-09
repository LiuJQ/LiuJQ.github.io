---
layout: post
title: Android Hybrid -- WebView上传拍照图片
tags: [Android, Hybrid]
---

&emsp;&emsp;基于上一次对Android Hybrid开发中WebView选择图片上传的探讨，我们可以扩展一下需求，假如产品提出要直接从前端调起Android相机拍照呢？Google了下HTML5有没有相关的API，找到一个国外开发者提到的解决方案：
>
[Capturing Audio & Video in HTML5](https://www.html5rocks.com/en/tutorials/getusermedia/intro/)
```html
<input type="file" accept="image/*;capture=camera">
<input type="file" accept="video/*;capture=camcorder">
<input type="file" accept="audio/*;capture=microphone">
```

&emsp;&emsp;如文中所言，这仅仅是作者自己提议的解决方案，不是HTML5官方认可的API。抱着试一试的心态，通过chrome调试修改html页面，新增上述的input标签，然后在Android端拦截上文（[Android Hybrid -- WebView上传文件图片]({{ site.baseurl}}{% post_url 2018-02-28-android-hybrid-upload-file %})）中提到的WebChromeClient#onShowFileChooser接口，查看传递过来的参数。这下好了，accept所有参数都没传递过来...

### 前端传递参数
&emsp;&emsp;所以，前端正确的解决方案是怎样的？经过一翻摸索，找到了正确的姿势：
```html
<input type="file" accept="image/*" capture="camera">
<!-- 或者 -->
<input type="file" accept="image/*" capture>
```
&emsp;&emsp;再次通过chrome调试查看WebChromeClient#onShowFileChooser接口接收到的参数，这次accept参数可以接收到了，那么capture参数怎么接收呢？

### Android端接收参数
&emsp;&emsp;上文（[Android Hybrid -- WebView上传文件图片]({{ site.baseurl}}{% post_url 2018-02-28-android-hybrid-upload-file %})）中提到的WebChromeClient#onShowFileChooser接口参数WebChromeClient.FileChooserParams，我们来一窥庐山真面目（为方便浏览已删减大部分）：
```java
/**
 * Parameters used in the {@link #onShowFileChooser} method.
 */
public static abstract class FileChooserParams {

    ...

    /**
     * Returns an array of acceptable MIME types. The returned MIME type
     * could be partial such as audio/*. The array will be empty if no
     * acceptable types are specified.
     */
    public abstract String[] getAcceptTypes();

    /**
     * Returns preference for a live media captured value (e.g. Camera, Microphone).
     * True indicates capture is enabled, false disabled.
     *
     * Use <code>getAcceptTypes</code> to determine suitable capture devices.
     */
    public abstract boolean isCaptureEnabled();

    ...
}
```
&emsp;&emsp;原来webkit内核已经帮我们做好了参数解析和封装。值得注意的是，capture参数在Android端是boolean类型的，即使前端写成capture="camera"，其实是Android端也拿不到字符串，这也是为什么上面我们提到前端写成capture也是可以工作的原因。

### Android处理拍照请求
&emsp;&emsp;跟选择图片文件一样，拍照也需要开发者自己调起。拍照保存的文件路径需要提前定义好，传递给拍照应用。
```java
Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
if (takePictureIntent.resolveActivity(getActivity().getPackageManager()) != null) {
    // Create the File where the photo should go
    File photoFile = null;
    try {
        File cacheDir = new File(FileUtil.WALLET_IMAGE_CACHE_DIR);
        if (!cacheDir.exists()) {
            //noinspection ResultOfMethodCallIgnored
            cacheDir.mkdirs();
        }
        photoFile = File.createTempFile(FileUtil.generateCacheFilePrefix(), FileUtil.IMAGE_FILE_SUFFIX_JPG, cacheDir);
    } catch (IOException ex) {
        // Error occurred while creating the File
        ex.printStackTrace();
    }
    // Continue only if the File was successfully created
    if (photoFile != null) {
        capturedPhotoUri = Uri.fromFile(photoFile);
        takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, capturedPhotoUri);
        startActivityForResultSafe(takePictureIntent, START_ACTIVITY_UPLOAD_FILE_REQUEST);
    }
}
```

### 前置摄像头
&emsp;&emsp;有些时候，需要直接调起自拍模式，笔者尝试了下，在魅族手机上是可以实现，其他厂商的手机暂未测试：
```java
takePictureIntent.putExtra("android.intent.extras.CAMERA_FACING", android.hardware.Camera.CameraInfo.CAMERA_FACING_FRONT);
takePictureIntent.putExtra("android.intent.extras.LENS_FACING_FRONT", 1);
takePictureIntent.putExtra("android.intent.extra.USE_FRONT_CAMERA", true);
```
