# RxJava
**RxJava现在满大街都是，最近才开始学习，这里记录相应的笔记<br>**

## 官方文档
[【ReactiveX/RxJava Wiki】](https://github.com/ReactiveX/rxjava/wiki)

## 1. 简介
* Webview是一个使用WebKit引擎的浏览器控件,但是没有提供地址栏，只是单纯的展示一个网页
* 大型项目中引入Webview可以节省开发成本

## 2. 使用说明
### 2.1 简单使用
```
//加载一个网页
webView.loadUrl("http://conow.cn/");

//加载本地html文件
//例如网络不好导致加载失败，可以直接加载本地的html来避免页面一直白屏
webView.loadUrl("file:///android_asset/networkError.html");
```


### 2.2 常用类

#### 2.2.1 WebSettings
> WebSettings用于管理WebView状态配置，当WebView第一次被创建时，WebView就已经有一个默认的配置，这些默认的配置将通过get方法返回，通过WebView中的getSettings方法获得一个WebSettings对象，如果一个WebView被销毁，在WebSettings中所有回调方法将抛出IllegalStateException异常。

etc.
```
        WebSettings settings = webView.getSettings();
        // 开启database storage API功能
        settings.setDatabaseEnabled(true);
        settings.setDatabasePath(dataPath);

        //开启缓存，缓存js，css，图片等等
        settings.setAppCacheEnabled(true);
        settings.setAppCachePath(cachePath);

        //设置缓存模式
        settings.setCacheMode(WebSettings.LOAD_DEFAULT);

        //设置支持js
        settings.setJavaScriptEnabled(true);
```


#### 2.2.2 WebViewClient
主要是监控webview的各种状态（开始加载、加载中、加载失败...）

在项目中，通常重写`shouldOverrideUrlLoading`和`shouldInterceptRequest`这个方法，拦截请求的url来判断当前进行什么业务操作

接下来说一下`shouldOverrideUrlLoading`这个方法，而`shouldInterceptRequest`这个方法在后面的webview优化中会提到

etc.
```
// 此处涉及到了与原生应用的交互，通过拦截符合所需要的url，将控制权转移出去，做另外的操作
// 例如这里，如果拦截到url不是我们项目中的，则跳转第三方网站

@Override
public boolean shouldOverrideUrlLoading(WebView view, String url) {
    if (!StringUtils.isEmpty(url)) {
        if (!url.startsWith(ARPConstant.URL_START)) {
            if(frontActivity!=null){
                Intent intent = new Intent(frontActivity, HttpWebViewActivity.class);
                intent.putExtra(HttpWebViewActivity.URL, url);
                intent.putExtra(HomeActivity.SHOW_WEB_VIEW, true);
                //跳转
                frontActivity.startActivity(intent);
                AppContext.webviewToAppTransition(frontActivity);
            }
            //return true表示不调用手机本身的浏览器
            return true;
        }
    }
    return super.shouldOverrideUrlLoading(view, url);
}
```

#### 2.2.3 WebChromeClient
> WebChromeClient是Html/Js和Android客户端进行交互的一个中间件，其将webview中js所产生的事件封装，然后传递到Android客户端

在我们项目中比较常用的就是`onConsoleMessage`方法了，
顾名思义就是获取webview的打印信息。
就像上面拦截url来监听需要什么动作一样，
同理这里可以获取webview打印的信息来告诉原生应用究竟要干什么，这里也涉及到js与原生的交互。

etc.

最近开发公文手写签批，恰好也涉及到整个的交互过程，下面来看看

前端代码
```
//手写签批,webview发出消息
function toAppDocEdit(cnEditorId, docUrl){
    appLog('docEdit', JSON.stringify({
        cnEditorId: cnEditorId, //手写签批组件id
        docUrl: docUrl //公文附件url
    }));
}

// APP日志打印
var appLog = function (methodStr, contentStr) {
    var logStr = methodStr+APP_FLAG+contentStr;
    if(IS_ANDROID_APP){
        console.log(logStr);// 打印语句
    }else if(IS_IOS_APP){
        ioslog(logStr);
    }
};
```

Android代码
```
@Override
public void onConsoleMessage(String message, int lineNumber, String sourceID) {
    if (!StringUtils.isEmpty(message)) {
        //接收到手写签批的log信息
        if(message.startsWith(docEdit)){
            //解析
            ...
            //跳转到相关Activity进行处理
            ...
        }
    }
    super.onConsoleMessage(message, lineNumber, sourceID);
}
```

上面的只是webview与原生的交互过程，下面来看一下整个手写签批的流程

![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/webviewNotifyApp.png "webview通知原生")


![](https://github.com/ConowDevNotes/AndroidDevNotes/blob/master/res/img/appNotifyWebview.png "原生通知webview")


### 2.3 优化

#### 2.3.1 资源预加载

**Webview一次加载过程会有许多外部依赖的 JS、CSS、图片等资源需要下载，我们可以提前将这些资源缓存好，等页面加载时直接进行替换**

1. 将前端的一些框架文件（jQuery,angular）打包到apk安装里面
2. 使用`shouldInterceptRequest`这个方法来拦截请求的url，判断当前请求的资源是否我们已经在apk包中存在，
如果存在则取之来加载，反之从网络加载。

etc.
```
@Override
public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
    String url = request.getUrl().toString();
    //解析请求文件的名称
    ...
    //从asset读取文件
    if(WebviewUtil.webviewResSet.contains(fileName)){
        AssetManager manager = AppContext.getInstance().getAssets();
        try {
            String assetPath = WebviewUtil.ASSET_WEBVIEW_RES_PATH + fileName;
            //开启流，读取apk安装包里面的文件
            InputStream is = manager.open(assetPath);
            WebResourceResponse webResourceResponse;
            assert fileName != null;
            if(fileName.endsWith(".js")){
                webResourceResponse = new WebResourceResponse("application/x-javascript","UTF-8",is);
            }
            return webResourceResponse;
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return super.shouldInterceptRequest(view, request);
}
```