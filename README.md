# Http
版本：2.3.5<br>
作者：西门提督<br>
日期：2016-12-13

##技术包含：Rxjava+Retrofit2+Okhttp3+FastJson
<strong>框架支持：<br><br>
<strong>普通GET请求<br>
<strong>普通POST请求<br>
<strong>Token失效自动刷新，续发接口<br>
<strong>嵌套请求（比如第二个接口需要第一个接口的参数）<br>
<strong>数据集合循环<br>
<strong>定时器<br>
<strong>周期动作<br>
<strong>轮询请求<br>
<strong>多异步请求并发，不按顺序返回接受<br>
<strong>多异步请求并发，顺序返回接受<br>
<strong>多异步请求并发，等待所有异步完成再做操作<br>
<strong>普通上传文件<br>
<strong>上传文件显示进度<br>
<strong>多文件上传显示进度<br>
<strong>普通下载<br>
<strong>下载文件显示进度<br>
##Http请求框架用法如下：

###1.AndroidManifest.xml添加权限：

// 访问网络权限<br>
uses-permission android:name="android.permission.INTERNET"<br>
// 往sdcard中写入数据的权限<br>
uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"<br>
// 在sdcard中创建/删除文件的权限<br>
uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"

###2.Application初始化操作
    @Override
    public void onCreate() {
        super.onCreate();
        HttpManager.Opration.setBaseUrl("http://www.xxx.yyy/"); // 请求路径前缀，可参考Retrofit2
        HttpManager.Opration.setShowLoger(true); // 是否打印网络请求详情，可参考Okhttp3
    }

###普通GET请求：
    public void testGet() {
        Subscription s = HttpHelper.Builder
            .builder(service.getVersion(getVersionParamas()))
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e(" >>> ", "before(new Action0)");
                }
            })
            .loadable(this) // 请求Dialog实现
            .filter(new Action1<BaseBean<VersionEntity>>() { // 非必需实现
                @Override
                public void call(BaseBean<VersionEntity> o) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    if (!o.isSuccess()) Log.e(" >>> ", o.isSuccess() + "");
                }
            })
            .callback(new HttpCallback<BaseBean<VersionEntity>>() {
                @Override
                public void onSuccess(BaseBean<VersionEntity> o) {
                    Log.e(" >>> ", o.getData().getChangeLog());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###普通POST请求：
    public void testPost() {
        Subscription s = HttpHelper.Builder
            .builder(service.versionAction(getVersionParamas()))
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e(" >>> ", "before(new Action0)");
                }
            })
            .loadable(this) // 请求Dialog实现
            .filter(new Action1<BaseBean<VersionEntity>>() { // 非必需实现
                @Override
                public void call(BaseBean<VersionEntity> o) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    if (!o.isSuccess()) Log.e(" >>> ", o.isSuccess() + "");
                }
            })
            .callback(new HttpCallback<BaseBean<VersionEntity>>() {
                @Override
                public void onSuccess(BaseBean<VersionEntity> o) {
                    Log.e(" >>> ", o.getData().getChangeLog());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###当某些请求需要携带Token，而Token失效自动刷新再续发原请求：
    public void testToken() {
        Subscription s = TokenHelper.Builder
            .builder(service.loginReward(mToken, testUrl1), service.login(testUrl2, "simon@cmonbaby.com", "123456"))
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e(" before >>> ", "before(new Action0)");
                }
            })
            .loadable(this) // 请求Dialog实现
            .retryCount(3) // 非必需重写，默认3次请求失败重试
            .tokenFilter(new TokenAction<BaseHttpEntity<TokenEntity>, BaseHttpEntity<UserInfo>>() {
                @Override
                public Observable<BaseHttpEntity<UserInfo>> call(BaseHttpEntity<TokenEntity> token) {
                     // token自动刷新后，得到token再返回续发请求接口
                    mToken = token.getData().getXauthToken();
                    Log.e("token >>> ", mToken);
                    return service.loginReward(mToken, testUrl1);
                }
            })
            .actionFilter(new Action1<BaseHttpEntity<UserInfo>>() { // 非必需实现
                @Override
                public void call(BaseHttpEntity<UserInfo> info) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    if (!info.isSuccess()) Log.e("actionFilter >>> ", info.getErr());
                }
            })
            .callback(new HttpCallback<BaseHttpEntity<UserInfo>>() {
                @Override
                public void onSuccess(BaseHttpEntity<UserInfo> info) {
                    // 最终请求返回数据
                    Log.e("callback >>> ", info.getData().toString());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###嵌套请求（比如第二个接口需要第一个接口的参数）
    public void nesting(View btn) {
        Subscription s = NestingHelper.Builder
            .builder(service.login(testUrl2, "simon@cmonbaby.com", "123456"), service.loginReward(mToken, testUrl1))
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    Log.e(" >>> ", "before(new Action0)");
                }
            })
            .loadable(this)
            .nestingCall(new NestingFunction<BaseHttpEntity<TokenEntity>, BaseHttpEntity<UserInfo>>() {
                @Override
                public Observable<BaseHttpEntity<UserInfo>> call(BaseHttpEntity<TokenEntity> token) {
                    Log.e("token >>> ", token.getData().getXauthToken());
                    // 得到第一个接口数据返回
                    mToken = token.getData().getXauthToken();
                    // 请求第二个接口
                    return service.loginReward(mToken, testUrl1);
                }
            })
            .toFilter(new Action1<BaseHttpEntity<UserInfo>>() { // 非必需实现
                @Override
                public void call(BaseHttpEntity<UserInfo> info) {
                    if (!info.isSuccess()) Log.e("toFilter >>> ", info.getErr());
                }
            })
            .callback(new HttpCallback<BaseHttpEntity<UserInfo>>() {
                @Override
                public void onSuccess(BaseHttpEntity<UserInfo> info) {
                    // 第二个接口数据返回
                    Log.e("callback >>> ", info.getData().toString());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }