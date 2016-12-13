# Http
Http请求框架

public class BaseApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        HttpManager.Opration.setBaseUrl("http://www.xxx.yyy/"); // 请求路径前缀
        HttpManager.Opration.setShowLoger(true); // 是否打印网络请求详情
    }
}
