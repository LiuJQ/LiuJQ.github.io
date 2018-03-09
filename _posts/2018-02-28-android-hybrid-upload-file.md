---
layout: post
title: Android Hybrid -- WebView上传文件图片
tags: [Android, Hybrid]
---

&emsp;&emsp;Android Hybrid开发中，常会遇到涉及选择图片上传的需求，比如实名认证功能需要上传用户身份证照片。这次我们就一起来探究一下Hybrid开发中，通过WebView实现选择图片上传功能的技术方案。

&emsp;&emsp;通常前端的代码是类似下面这样的：
```html
<input type="file" accept="image/*">
```

&emsp;&emsp;前端发起input请求之后，Android端又是怎么样接收该请求的呢？如果不对WebView作任何处理，触发前端的input事件后，页面不会有任何响应。通过对WebView的了解，发现需要开发者设置WebChromeClient并实现相应的回调接口，然后在接口中处理文件上传请求。

### WebChromeClient接口
&emsp;&emsp;从Android 2.0发展到Android 5.0以后，Google开发者对该接口进行了多次改版，最终演变成下面这些接口方法：
```java
webview.setWebChromeClient(new WebChromeClient() {
    // For Android < 3.0
    public void openFileChooser(ValueCallback<Uri> valueCallback) {
        ***
    }

    // For Android >= 3.0
    public void openFileChooser(ValueCallback valueCallback, String acceptType) {
        ***
    }

    // For Android >= 4.1
    public void openFileChooser(ValueCallback<Uri> valueCallback, String acceptType, String capture) {
        ***
    }

    // For Android >= 5.0
    @Override
    public boolean onShowFileChooser(WebView webView, ValueCallback<Uri[]> filePathCallback, WebChromeClient.FileChooserParams fileChooserParams) {
        ***
        return true;
    }
});
```
&emsp;&emsp;Android 3.0以后可以从接口中获得前端传递的文件类型参数acceptType，Android 5.0以后该参数被封装在FileChooserParams类里面。所有的接口方法都会提供ValueCallback参数，顾名思义，它就是用来给我们选择完图片文件之后回调给前端的回调接口，Android 5.0以后该回调接口支持Uri数组，因此可以实现同时选择多个文件的功能。


### 选择图片
&emsp;&emsp;文件请求过来了，那么我们就可以在接口中调起选择图片的页面了。Android提供原生API调起选择图片页面：
```java
Intent intent = new Intent(Intent.ACTION_PICK, android.provider.MediaStore.Images.Media.EXTERNAL_CONTENT_URI);
if (intent.resolveActivity(getActivity().getPackageManager()) != null) {
    startActivityForResult(intent, START_ACTIVITY_UPLOAD_FILE_REQUEST);
}
```
&emsp;&emsp;结合多个Android版本API，我们可以提供一个统一的处理方法：
```java
protected void openFileInput(final ValueCallback<Uri> fileUploadCallbackFirst,
                                 final ValueCallback<Uri[]> fileUploadCallbackSecond,
                                 final boolean allowMultiple,
                                 final boolean capture,
                                 String acceptType) {
    if (mFileUploadCallbackFirst != null) {
        mFileUploadCallbackFirst.onReceiveValue(null);
    }
    mFileUploadCallbackFirst = fileUploadCallbackFirst;

    if (mFileUploadCallbackSecond != null) {
        mFileUploadCallbackSecond.onReceiveValue(null);
    }
    mFileUploadCallbackSecond = fileUploadCallbackSecond;

    if (!TextUtils.isEmpty(acceptType) && acceptType.startsWith("image")) {
        Intent intent = new Intent(Intent.ACTION_PICK, android.provider.MediaStore.Images.Media.EXTERNAL_CONTENT_URI);
        intent.putExtra(Intent.EXTRA_ALLOW_MULTIPLE, allowMultiple && Build.VERSION.SDK_INT >= 18);
        if (intent.resolveActivity(getActivity().getPackageManager()) != null) {
            startActivityForResultSafe(intent, START_ACTIVITY_UPLOAD_FILE_REQUEST);
        }
    } else {
        Log.w(TAG, "Please contact developers to support other files.");
        // 无法处理也要回调给H5，否则H5页面无法继续响应用户
        if (mFileUploadCallbackFirst != null) {
            mFileUploadCallbackFirst.onReceiveValue(null);
            mFileUploadCallbackFirst = null;
        } else if (mFileUploadCallbackSecond != null) {
            mFileUploadCallbackSecond.onReceiveValue(null);
            mFileUploadCallbackSecond = null;
        }
    }
}
```

### 返回图片
&emsp;&emsp;从上面的代码可发现，调起选择图片页面的方式是startActivityForResult，因此图片Uri返回的处理应该在onActivityResult方法。那么针对Android 5.0以下版本，需要通过mFileUploadCallbackFirst来回调给前端，Android 5.0以上则需要通过mFileUploadCallbackSecond回调给前端，并且要注意Android 5.0以上有可能有多张图片的情况。
```java
@Override
public void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == START_ACTIVITY_UPLOAD_FILE_REQUEST) {
        if (resultCode == Activity.RESULT_OK) {
            if (mFileUploadCallbackFirst != null) {
                Uri photoUri = null;
                // TODO 获取所选图片的Uri
                // TODO 是否需要对图片作压缩处理？
                mFileUploadCallbackFirst.onReceiveValue(photoUri);
                mFileUploadCallbackFirst = null;
            } else if (mFileUploadCallbackSecond != null) {
                Uri[] photoUris = null;
                // TODO 获取所选图片的Uri数组
                // TODO 是否需要对所有图片作压缩处理？
                mFileUploadCallbackSecond.onReceiveValue(photoUris);
                mFileUploadCallbackSecond = null;
            }
        }
    } else {
        // 用户取消选择，也要回调给前端
        if (mFileUploadCallbackFirst != null) {
            mFileUploadCallbackFirst.onReceiveValue(null);
            mFileUploadCallbackFirst = null;
        } else if (mFileUploadCallbackSecond != null) {
            mFileUploadCallbackSecond.onReceiveValue(null);
            mFileUploadCallbackSecond = null;
        }
    }
}
```
&emsp;&emsp;值得关注的是，很多时候用户选择图片是不会考虑图片大小的，因此带来的性能消耗也是不确定的，我们需要设计一套图片压缩算法，为图片上传功能带来更好的体验。图片的压缩可以参考之前的文章，我们讨论过Bitmap的加载优化，可以关注一下：[Android -- Bitmap加载性能优化]({{ site.baseurl }}{% post_url 2018-02-27-android-bitmap-compress %})
