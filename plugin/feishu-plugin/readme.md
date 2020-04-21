新增飞书报警支持，在原有的项目基础上，新增报警规则比较简单；下面介绍一下实现与使用姿势

### 1. 飞书报警支持

Quick-Alarm系统提供基于SPI方式进行报警方式的扩展，当我们需要新增一种报警方式时，实现接口`IExecute`，并注册spi文件即可

所以我们的`FeishuExecute`实现如下

```java
public class FeishuExecute implements IExecute {

    @Override
    public void sendMsg(List<String> users, String title, String msg) {
        users.forEach(user -> FeishuPublisher.sendMessage(title, msg, user));
    }

}
```

具体的钉钉推送，抽象了一个辅助类，关于钉钉机器人的使用姿势，可以参考一下官方文档: [获取自定义机器人webhook](https://getfeishu.cn/hc/zh-cn/articles/360024984973-%E5%9C%A8%E7%BE%A4%E8%81%8A%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%9C%BA%E5%99%A8%E4%BA%BA)

从官方文档中，调用飞书机器人，实际上就是向一个特定的url，发送post请求；http请求工具这里选择了 `OkHttp` 这个库，具体实现代码如下

```java
public class FeishuPublisher {
    private static final String TEMPLATE = "title:\t%s\n\ncontent:\t%s";
    private static final Logger logger = LoggerFactory.getLogger(FeishuPublisher.class);
    private static final String FEISHU_URL = "https://open.feishu.cn/open-apis/bot/hook/";
    private static final MediaType JSON;
    private static OkHttpClient okHttpClient;

    static {
        okHttpClient = new OkHttpClient();
        JSON = MediaType.get("application/json; charset=utf-8");
    }

    public static void sendMessage(String title, String content, String token) {
        try {
            doPost(title, content, token);
        } catch (Exception e) {
            logger.error("failed to publish msg: {} to DingDing! {}", content, e);
        }
    }

    public static String doPost(String title, String content, String token) throws IOException {
        RequestBody body = RequestBody.create(buildTextMsgBody(title, content), JSON);

        try (Response response = okHttpClient.newCall(new Request.Builder().url(FEISHU_URL + token).post(body).build())
                .execute()) {
            return response.body().string();
        }
    }

    private static String buildTextMsgBody(String title, String content) {
        JSONObject msg = new JSONObject(4);
        msg.put("title", title);
        msg.put("text", content);
        return msg.toJSONString();
    }
}
```

上面基本上就完成了一个报警器的代码实现，接下来别忘了需要注册，在资源目录下，新建文件夹 `META-INF/services`（注意上面是两层目录）, 添加文件`com.hust.hui.alarm.core.execut.api.IExecute`， 文件内容如下

```
com.hust.hui.alarm.plugin.feishu.FeishuExecute
```

### 2. 测试使用

使用钉钉报警时，需要先有一个自定义的钉钉机器人，可以在一个群里面申请，会获得一个token；然后将这个token 配置到我们的预警规则里面

```json
{
    "XXX,YYY": {
        "level": "LOG",
        "autoIncEmergency": true,
        "max": 30,
        "min": 3,
        "threshold": [
            {
                "level": "FEISHU",
                "threshold": 10,
                "max": 12,
                "users": [
                    "请用实际的token替换!!!"
                ]
            }
        ],
        "users": [
            "yihui"
        ]
    }
}
```

在上面的配置中，所以我们需要在users里面填写你的机器人token，这个需要额外注意；

此外新版的钉钉机器人有三种安全校验方式

- 关键词：需要你的报警内容中，包含对应的关键字
- 加签：暂时不支持这种场景，感觉意义不大
- ip白名单：确保使用的机器ip在白名单内部

```java
public static void main(String[] args) throws InterruptedException {
    // 测试异常升级的case
    // 计数 [10 - 12) 钉钉报警
    for (int i = 0; i < 40; i++) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                AlarmWrapper.getInstance().sendMsg("YYY", "异常报警升级测试");
            }
        }).start();
    }

    Thread.sleep(1000 * 600);
}
```
