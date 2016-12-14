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
<strong>数组集合循环<br>
<strong>定时操作<br>
<strong>周期性操作<br>
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
        HttpManager.Opration.setBaseUrl("http://www.cmonbaby.com/"); // 请求路径前缀，可参考Retrofit2
        HttpManager.Opration.setShowLoger(true); // 是否打印网络请求详情，可参考Okhttp3
    }

###3.BaseActivity初始化操作（implements HttpLoadable, FileLoadable）
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            // 初始化Retrofit，可以作为全局变量，方便子类复用
            RetrofitUtils retrofit = RetrofitUtils.getInstance();
        }

        @Override
        public boolean isKeepShowing() {
            // 是否持续保持Dialog不消失，比如多个请求并发
            return false;
        }

        @Override
        public void showDialogLoading() { // 接口请求Dialog
            // new Dialog过程自定义，最好将对象名也设为全局变量
            dialog.show();
        }

        @Override
        public void dismissDialogLoading() {
            if (dialog != null && dialog.isShowing()) dialog.dismiss();
        }

        @Override
        public void showProgressDialog() { // 上传下载进度Dialog
            // new Dialog过程自定义，最好将对象名也设为全局变量
            dialog.show();
        }

        @Override
        public void setDialogTitle(String title) {
            // 设置Dialog标题，以上两者Dialog都适用
            if (dialog != null) dialog.setTitle(title);
        }

        @Override
        public void setDialogContent(String content) {
            // 设置Dialog内容摘要，以上两者Dialog都适用
            if (dialog != null) dialog.setContent(content);
        }

        @Override
        public void setDialogMaxProgress(int maxProgress) {
            // 设置上传下载最大进度
            if (dialog != null) dialog.setMaxProgress(maxProgress);
        }

        @Override
        public void setDialogProgress(int progress) {
            // 设置上传下载进度
            if (dialog != null) dialog.setProgress(progress);
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
            .loadable(this) // 可不填，请求Dialog实现
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
            .loadable(this) // 可不填，请求Dialog实现
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
            .loadable(this) // 可不填，请求Dialog实现
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
    public void nesting() {
        Subscription s = NestingHelper.Builder
            .builder(service.login(testUrl2, "simon@cmonbaby.com", "123456"), service.loginReward(mToken, testUrl1))
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e(" >>> ", "before(new Action0)");
                }
            })
            .loadable(this) // 可不填，请求Dialog实现
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
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
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

###数组、集合的遍历
    public void list() {
        String[] names = {"张三", "李四", "王五", "赵六", "钱七"};
        Observable
            .from(names)
            .subscribe(new Action1<String>() {

                @Override
                public void call(String name) {
                    Log.e("list >>> ", name);
                }
            });
    }

###定时操作
    public void timer() {
        Observable.timer(2, TimeUnit.SECONDS)
            .subscribe(new Observer<Long>() {

                @Override
                public void onCompleted() {
                    Log.e ("timer >>> ", "completed");
                }

                @Override
                public void onError(Throwable e) {
                    Log.e("timer >>> ", "error");
                }

                @Override
                public void onNext(Long number) {
                    Log.e ("timer >>> ", "hello world");
                }
            });
    }

###周期性操作
    public void cycle() {
        Observable.interval(2, TimeUnit.SECONDS)
            .subscribe(new Observer<Long>() {

                @Override
                public void onCompleted() {
                    Log.e ("cycle >>> ", "completed");
                }

                @Override
                public void onError(Throwable e) {
                    Log.e("cycle >>> ", "error");
                }

                @Override
                public void onNext(Long number) {
                    Log.e ("cycle >>> ", "hello world");
                }
            });
    }

###轮询请求
    public void polling() {
        Observable.create(new Observable.OnSubscribe<BaseBean<VersionEntity>>() {
            @Override
            public void call(final Subscriber<? super BaseBean<VersionEntity>> observer) {
                Schedulers.newThread().createWorker()
                        .schedulePeriodically(new Action0() {
                            @Override
                            public void call() {
                                Log.e("polling >>> ", "polling");
                            }
                        }, 3000, 5000, TimeUnit.MILLISECONDS);
            } // 首次执行行动之前等待， 执行动作之间等待的时间间隔
        }).subscribe(new HttpCallback<BaseBean<VersionEntity>>() {
            @Override
            public void onSuccess(BaseBean<VersionEntity> bean) {
                Log.e("polling >>> ", bean.getData().getChangeLog());
            }
        });
    }

###多异步请求并发，不按顺序返回接受
    public void merge() { // 拼接两个Observable的输出，不保证顺序，按照事件产生的顺序发送给订阅者
        Subscription s = Merge2Helper.Builder.builder(service.versionAction(getVersionParamas()), service.login(testUrl2, "simon@cmonbaby.com", "123456"))
            .loadable(this) // 可不填，请求Dialog实现
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "before");
                }
            })
            .result1(new Action1<BaseBean<VersionEntity>>() {
                @Override
                public void call(BaseBean<VersionEntity> version) {
                    // 第一个接口返回
                    Log.e("result1 >>> ", version.getData().getChangeLog());
                }
            })
            .result2(new Action1<BaseHttpEntity<TokenEntity>>() {
                @Override
                public void call(BaseHttpEntity<TokenEntity> token) {
                    // 第二个接口返回
                    Log.e("result2 >>> ", token.getData().getXauthToken());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

### 多异步请求并发，顺序返回接受
    public void concat() {
        Subscription s = Concat2Helper.Builder.builder(service.versionAction(getVersionParamas()), service.login(testUrl2, "li_ai@baxter.com", "123456"))
            .loadable(this) // 可不填，请求Dialog实现
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "before");
                }
            })
            .result1(new Action1<BaseBean<VersionEntity>>() {
                @Override
                public void call(BaseBean<VersionEntity> version) {
                    // 第一个接口返回
                    Log.e("result1 >>> ", version.getData().getChangeLog());
                }
            })
            .result2(new Action1<BaseHttpEntity<TokenEntity>>() {
                @Override
                public void call(BaseHttpEntity<TokenEntity> token) {
                    // 第二个接口返回
                    Log.e("result2 >>> ", token.getData().getXauthToken());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###多异步请求并发，等待所有异步完成再做操作
    public void zip() {
        Subscription s = Zip2Helper.Builder.builder(service.versionAction(getVersionParamas()), service.login(testUrl2, "simon@cmonbaby.com", "123456"), String.class)
            .loadable(this) // 可不填，请求Dialog实现
            .before(new Action0() { // 非必需实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "before");
                }
            })
            .result1(new Action1<BaseBean<VersionEntity>>() {
                @Override
                public void call(BaseBean<VersionEntity> version) {
                    // 第一个接口返回
                    Log.e("result1 >>> ", version.getData().getChangeLog());
                }
            })
            .result2(new Action1<BaseHttpEntity<TokenEntity>>() {
                @Override
                public void call(BaseHttpEntity<TokenEntity> token) {
                    // 第二个接口返回
                    Log.e("result2 >>> ", token.getData().getXauthToken());
                }
            })
            .result(new Func2<BaseBean<VersionEntity>, BaseHttpEntity<TokenEntity>, String>() {
                @Override
                public String call(BaseBean<VersionEntity> version, BaseHttpEntity<TokenEntity> token) {
                    // 得到第一个，第二个接口返回后，执行第三个接口
                    Log.e("result >>> ", version.getData().getChangeLog() + "\n" + token.getData().getXauthToken());
                    return version.getData().getChangeLog() + "\n" + token.getData().getXauthToken();
                }
            })
            .callback(new HttpCallback<String>() {
                @Override
                public void onSuccess(String s) {
                    // 第三个接口数据返回
                    Log.e("callback >>> ", s);
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###普通上传文件
    public void testUpload() {
        String url = "http://www.cmonbaby.com/image";
        String path = Environment.getExternalStorageDirectory().getAbsolutePath();
        File file = new File(path + "/" + "simon.jpg");
        RequestBody body = RequestBody.create(MediaType.parse("image/jpeg"), file);
        MultipartBody.Part part = MultipartBody.Part.createFormData("file", file.getName(), body);
        service.uploadFile(url, getVersionParamas(), part)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribeOn(AndroidSchedulers.mainThread())
            .subscribe(new HttpCallback<BaseBean<VersionEntity>>() {
                @Override
                public void onSuccess(BaseBean<VersionEntity> version) {
                    // 上传完成
                    Log.e("simon >>> ", version.getData().getName());
                }
            });
    }

###上传文件显示进度
    public void testUploadProgress() {
        final String url = "http://www.cmonbaby.com/image";
        Subscription s = UploadFile.Builder.upload(VersionService.class, service.uploadFile(null, null, null))
            .loadable(this) // 可不填，请求Dialog实现
            .dialogTitle("图片上传") // 可不填
            .dialogContent("正在上传……") // 可不填
            .fileType("image/jpeg") // 可不填
            .fileParams("file") // 可不填
            .filePath(Environment.getExternalStorageDirectory().getAbsolutePath() + "/testUpload.jpg") // 测试路径
            .requestService(new UploadFileCall<BaseBean<VersionEntity>, VersionService>() {
                @Override
                public Observable<BaseBean<VersionEntity>> uploadFile(VersionService service, MultipartBody.Part part) {
                    return service.uploadFile(url, getVersionParamas(), part);
                }
            })
            .before(new Action0() { // 可不实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "init something");
                }
            })
            .progressListener(new ProgressListener() { // 上传进度监听
                @Override
                public void onProgress(long progress, long total, boolean isFinish) {  // 可不实现
                    Log.e("progress >>> ", progress + " / " + total + " --- " + isFinish);
                }
            })
            .filter(new Action1<BaseBean<VersionEntity>>() { // 可不实现
                @Override
                public void call(BaseBean<VersionEntity> version) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    Log.e("filter >>> ", version.getData().getName());
                }
            })
            .callback(new HttpCallback<BaseBean<VersionEntity>>() {
                @Override
                public void onSuccess(BaseBean<VersionEntity> version) {
                    // 上传完成
                    Log.e("callback >>> ", version.getData().getName());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###多文件上传显示进度
    public void testUploadFiles() {
        final String url = "http://www.cmonbaby.com/images";
        String path = Environment.getExternalStorageDirectory().getAbsolutePath();
        Subscription s = UploadFiles.Builder.uploadFiles(VersionService.class, service.uploadFiles(null, null, null))
            .loadable(this) // 可不填，请求Dialog实现
            .dialogTitle("图片上传") // 可不填
            .dialogContent("正在上传……") // 可不填
            .fileParams("files") // 可不填
            .files(new File(path + "/" + "simon.jpg")) // 测试路径1
            .files(new File(path + "/" + "testUpload.jpg")) // 测试路径2
            .files(new File(path + "/" + "temp.jpg")) // 测试路径3
            .requestService(new UploadFilesCall<BaseListBean<VersionEntity>, VersionService>() {
                @Override
                public Observable<BaseListBean<VersionEntity>> uploadFiles(VersionService service, Map<String, RequestBody> requestBodyMap) {
                    return service.uploadFiles(url, requestBodyMap, getVersionParamas());
                }
            })
            .before(new Action0() { // 可不实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "init something");
                }
            })
            .progressListener(new ProgressListener() { // 上传进度监听
                @Override
                public void onProgress(long progress, long total, boolean isFinish) {  // 可不实现
                    Log.e("progress >>> ", progress + " / " + total + " --- " + isFinish);
                }
            })
            .filter(new Action1<BaseListBean<VersionEntity>>() { // 可不实现
                @Override
                public void call(BaseListBean<VersionEntity> version) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    Log.e("filter >>> ", version.getData().size() + "");
                }
            })
            .callback(new HttpCallback<BaseListBean<VersionEntity>>() {
                @Override
                public void onSuccess(BaseListBean<VersionEntity> version) {
                    // 上传完成
                    Log.e("callback >>> ", version.getData().size() + "");
                }
            })
            .toSubscribe();
        addSubscription(s);
    }

###普通下载
    public void testDown() {
        String url = "http://www.cmonbaby.com/download/http-2.3.5.apk";
        service.downloadFile(url)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribeOn(AndroidSchedulers.mainThread())
            .subscribe(new HttpCallback<Response<ResponseBody>>() {
                @Override
                public void onSuccess(Response<ResponseBody> response) {
                    // 下载完成，流的写入代码此处省略
                    Log.e("simon >>> ", response.body().toString());
                }
            });
    }

###下载文件显示进度
    public void testDownProgress() {
        final String url = "http://www.cmonbaby.com/download/http-2.3.5.apk";
        Subscription s = DownloadFile.Builder.download(VersionService.class)
            .loadable(this) // 可不填，请求Dialog实现
            .dialogTitle("文件下载") // 可不填
            .dialogContent("文件下载中，请稍候……") // 可不填
            .responseService(new DownLoadCall<VersionService>() {
                @Override
                public Observable<Response<ResponseBody>> downloadFile(VersionService service) {
                    return service.downloadFile(url);
                }
            })
            .before(new Action0() { // 可不实现
                @Override
                public void call() {
                    // 请求之前做一些事情，周期只有一次
                    Log.e("before >>> ", "init something");
                }
            })
            .progressListener(new ProgressListener() { // 下载进度监听
                @Override
                public void onProgress(long progress, long total, boolean isFinish) {
                    Log.e("progress >>> ", progress + " / " + total + " --- " + isFinish);
                }
            })
            .filter(new Action1<Response<ResponseBody>>() { // 可不实现
                @Override
                public void call(Response<ResponseBody> response) {
                    // 数据过滤，比如服务端返回200，但业务出现错误可在此进行处理
                    Log.e("filter >>> ", response.body().toString());
                }
            })
            .callback(new HttpCallback<Response<ResponseBody>>() {
                @Override
                public void onSuccess(Response<ResponseBody> response) {
                    // 下载完成，流的写入代码此处省略
                    Log.e("callback >>> ", response.body().toString());
                }
            })
            .toSubscribe();
        addSubscription(s);
    }